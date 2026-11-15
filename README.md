# Projekt inżynierski — instrukcja rozwoju raportu R Markdown

> **Dokumentacja techniczna i standard pracy dla kolejnych autorów oraz asystentów AI.**  
> **Stan referencyjny:** `Inzynierka_uporzadkowana(2)_poprawiona.Rmd` (15.11.2026).  
> **Temat:** Prognozowanie hierarchicznych szeregów czasowych na podstawie miesięcznych dochodów federalnych USA.  
> **Zasada nadrzędna:** zmieniamy tylko to, co potrzebne; nie psujemy poprawnych obliczeń, nie rozdzielamy kodu na osobne `.R` bez wyraźnego polecenia i nie poprawiamy wizualizacji kosztem poprawności statystycznej.

## Spis treści

1. [Co jest źródłem prawdy](#1-co-jest-źródłem-prawdy)
2. [Struktura i przepływ danych](#2-struktura-i-przepływ-danych)
3. [Konwencje R Markdown i kolejność chunków](#3-konwencje-r-markdown-i-kolejność-chunków)
4. [Rejestr istniejących funkcji](#4-rejestr-istniejących-funkcji)
5. [Jak używać obecnych funkcji](#5-jak-używać-obecnych-funkcji)
6. [Jak projektować nowe funkcje](#6-jak-projektować-nowe-funkcje)
7. [Wzorzec nowej funkcji wizualizacyjnej](#7-wzorzec-nowej-funkcji-wizualizacyjnej)
8. [Standard wyglądu, podpisów i źródeł](#8-standard-wyglądu-podpisów-i-źródeł)
9. [Przewodnik rozbudowy poszczególnych sekcji](#9-przewodnik-rozbudowy-poszczególnych-sekcji)
10. [Reguły metodologiczne i integralność danych](#10-reguły-metodologiczne-i-integralność-danych)
11. [Testowanie i diagnostyka błędów](#11-testowanie-i-diagnostyka-błędów)
12. [Gotowe prompty do ChatGPT / Codexa](#12-gotowe-prompty-do-chatgpt--codexa)
13. [Checklista przed przyjęciem zmian](#13-checklista-przed-przyjęciem-zmian)
14. [Co jest zaimplementowane, a czego jeszcze brakuje](#14-co-jest-zaimplementowane-a-czego-jeszcze-brakuje)

---

## 1. Co jest źródłem prawdy

Dokumentację przygotowano na podstawie **konkretnej wersji** raportu: `Inzynierka_uporzadkowana(2)_poprawiona.Rmd`. Przed każdą nową zmianą należy otworzyć **aktualny**, używany przez autora plik `.Rmd` — jego zawartość ma pierwszeństwo przed README, wcześniejszymi rozmowami, osobnymi `.R` i dawnymi instrukcjami.

**W szczególności:** istniejące pliki `DOKUMENTACJA_tabela_praca.md`, `INSTRUKCJA_wykres_kolowy.md`, `tabela_praca.R` i `wykres_kolowy.R` mogą opisywać **starsze, samodzielne warianty** funkcji. Nie wolno bez weryfikacji przyjąć, że ich sygnatury i sposób renderowania odpowiadają funkcjom w bieżącym `.Rmd`. Przykład różnicy: obecna `wykres_kolowy()` tworzy własny podpis, więc do jej chunka **nie dodaje się `fig.cap`**.

Projekt ma być **jednym samowystarczalnym plikiem R Markdown**, z funkcjami zdefiniowanymi w jego chunkach. Nie tworzymy i nie wywołujemy nowych plików `.R` przez `source()` bez jawnej decyzji autora.

### Co można zmienić bez większego ryzyka?

- Tekst metodologiczny, opisy i podpisy — po sprawdzeniu, czy zgadzają się z obliczeniami.
- Kolorystykę, grubości linii i układ paneli — bez zmiany wartości danych.
- Nowe, niezależne funkcje wizualizacyjne oparte na istniejących obiektach.
- Dodatkowe miary i analizy, jeśli jasno zdefiniowano ich metodę obliczania i zakres danych.

### Co wymaga szczególnej ostrożności?

- `mapa_grup`, `aggregate_key()`, definicje `wezly` i `id_wezla`.
- Podział trening/walidacja, jego horyzont i daty.
- Definicje modeli, sezonowość, miary błędu, uzgadnianie hierarchiczne.
- Zmiana nazw kolumn, od których zależy wiele następnych chunków.
- Wspólny renderer rysunków i funkcja `tabela_praca()` — są wykorzystywane wielokrotnie.

---

## 2. Struktura i przepływ danych

### 2.1. Hierarchia

Analizowane są miesięczne dochody federalne USA w miliardach USD. Hierarchia obejmuje **10 szeregów**:

```text
Dochody ogółem (1)
├── Grupa 1 (1)
│   ├── Podatki od osób fizycznych
│   ├── Akcyza
│   └── Podatki od spadków i darowizn
└── Grupa 2 (1)
    ├── Ubezpieczenia i emerytury
    ├── Podatki od przedsiębiorstw
    ├── Cła
    └── Pozostałe dochody

Łącznie: 1 + 2 + 7 = 10 szeregów.
```

Przypisanie kategorii pochodzi wyłącznie z `mapa_grup`. Grupy są **konstrukcją analityczną opartą na kryterium ilościowym**, nie klasyfikacją ekonomiczną U.S. Treasury. Nie wolno przedstawiać ich jako urzędowego podziału bez osobnego źródła.

### 2.2. Główne obiekty i ich zależności

| Obiekt | Gdzie powstaje | Do czego służy |
|---|---|---|
| `params$plik_danych` | YAML | Ścieżka do wejściowego CSV (w stanie referencyjnym `../DANE/DANE_FINANSOWE.csv`). |
| `wyglad` | `konfiguracja-wygladu` | Centralne ustawienia rysunków, tabel, kolorów i wybranych analiz. |
| `mapa_grup` | `mapowanie-grup` | Stała mapa siedmiu kategorii do dwóch grup. |
| `pozycje`, `sumy_opublikowane` | przygotowanie danych | Rozdzielenie kategorii od urzędowej sumy kontrolnej. |
| `skladniki` | `dane-07-kategorie` | Wpływy kategorii w **mld USD** z przypisaną grupą. |
| `dane_ts` | `dane-08-agregacja-hierarchii` | Hierarchiczny `tsibble`: ogółem, grupy i kategorie. |
| `wezly` | `dane-09-wezly` i `ustawienia-paneli-dane` | Metadane węzłów: poziom, grupa, kategoria, etykieta panelu, `id_wezla`. |
| `kontrola_sumy` | `dane-10-zgodnosc-sumy` | Porównanie sumy składników z opublikowaną sumą. |
| `trening`, `walidacja` | `podzial-dane` | Chronologiczne dane do modelowania i walidacji. |
| `daty_trening`, `daty_walidacji` | `podzial-dane` | Wspólne osie czasu dla wszystkich węzłów. |
| `historia` | `historia-dane` | Dane historyczne z etykietami paneli i datami `Date`. |
| `pelne_lata` | `udzialy-kategorii-dane` | Pełne lata kalendarzowe z treningu. |
| `udzialy` | `udzialy-kategorii-dane` | Sumy i udziały siedmiu kategorii. |
| `sezony` | `sezonowosc-dane` | Obserwacje z pełnych lat, z `rok` i `nr_miesiaca`. |
| `acf_trening` | `dane-acf` | ACF poziomów, granice odniesienia i znacznik opóźnień sezonowych. |
| `modele_bazowe` | `modele-i-prognozy` | Dopasowane Naive, SNaive, ETS i ARIMA. |
| `prognozy_bazowe` | `modele-i-prognozy` | Prognozy punktowe dla dat walidacyjnych. |
| `obserwacje`, `prognozy` | `prognozy-wykres-dane` | Dane z etykietami do wizualizacji porównawczej. |

**Ważne:** `wezly` jest **uzupełniane** o `etykieta_panelu` i `panel` w chunku `ustawienia-paneli-dane`. Przygotowanie wykresów zależnych od tych pól musi nastąpić dopiero po tym chunku.

### 2.3. Najważniejsze obiekty konfiguracyjne

```r
okres_sezonowy                # 12L — sezonowość miesięczna
h_walidacji                   # Domyślnie 12; może pochodzić z YAML
wyglad$wykres                # Wymiary, panele, grubość, czcionka
wyglad$kolory                # Kolory obserwacji, walidacji, ACF itd.
wyglad$kolory$modele         # Nazwane barwy Naive/SNaive/ETS/ARIMA
wyglad$linie_modeli          # Przypisanie rodzajów linii do modeli
paleta                       # Wielokolorowa paleta kategorii
motyw_szeregow               # Wspólny motyw ggplot2
```

**Nie hardkoduj dat ani liczby obserwacji** w funkcjach, jeśli te informacje można uzyskać z istniejących obiektów. Tekst opisowy z wpisanymi konkretnymi datami powinien być aktualizowany przy zmianie danych — najlepiej wyprowadzać te informacje z `daty_trening` i `daty_walidacji`.

---

## 3. Konwencje R Markdown i kolejność chunków

### 3.1. Trzy rodzaje chunków

**A. Definicja funkcji — bez wydruku:**

````markdown
```{r funkcja-nowa-analiza, include=FALSE}
nowa_funkcja <- function(dane, ...) {
  # ...
}
```
````

`include=FALSE` **nie blokuje wykonania**. Funkcja jest dostępna w dalszych chunkach. Nie ustawiaj `eval=FALSE`, jeśli ma być używana podczas Knit.

**B. Przygotowanie danych — bez wydruku:**

````markdown
```{r nowa-analiza-dane, include=FALSE}
dane_nowej_analizy <- ...
```
````

Nie generuj tu wykresów, tabel, `print()` ani `cat()`, chyba że to zamierzone narzędzie diagnostyczne, które nie trafia do raportu.

**C. Prezentacja — wynik w miejscu wywołania:**

````markdown
```{r nowa-analiza-wykres, results='asis'}
wykres_nowej_analizy(
  dane = dane_nowej_analizy,
  podpis = "...",
  zrodlo = "Opracowanie własne na podstawie danych U.S. Treasury."
)
```
````

Dla `tabela_praca()` stosujemy zwykły chunk prezentacyjny. Dla wykresów zwracających `knitr::asis_output()` stosujemy `results='asis'`.

### 3.2. Obowiązkowa kolejność logiczna

```text
YAML / parametry
   ↓
Konfiguracja wyglądu i biblioteki
   ↓
Import, czyszczenie i walidacja danych
   ↓
Mapowanie kategorii, agregacja hierarchiczna, indeks węzłów
   ↓
Podział chronologiczny, metadane paneli i motyw wykresów
   ↓
Definicje funkcji / dane konkretnej analizy
   ↓
Wykresy, tabele i komentarze wykorzystujące te obiekty
   ↓
Modele → prognozy → ocena jakości → uzgadnianie → wnioski
```

Przykład rzeczywistego błędu z tego projektu: chunk `acf-ogolem` nie może wykonać się przed `dane-acf`, bo `acf_trening` jeszcze nie istnieje. Analogicznie wykres nie może użyć `wykres_acf()`, zanim wykona się chunk `funkcja-wykres-acf`.

**Dodatkowa subtelność:** definicja `wykres_szeregow()` może znajdować się wyżej niż definicja `motyw_szeregow`, ponieważ R odszukuje tę zmienną przy **wywołaniu** funkcji. Samo **wywołanie** musi jednak następować po utworzeniu motywu.

### 3.3. Nazewnictwo

- Chunk funkcji: `funkcja-wykres-miar`.
- Chunk obliczeń: `miary-prognoz-dane`.
- Chunk prezentacji: `miary-prognoz-wykres`.
- Funkcja: `wykres_miar()` / `oblicz_miary()`.
- Obiekt: `miary_prognoz`.
- **Nazwy chunków muszą być unikatowe.** Po przeniesieniu chunków nie pozostawiaj starej kopii.
- Jeśli nowe wywołanie ma własny podpis/rysunek, nadaj mu **osobny chunk**. Wspólny renderer używa nazwy aktualnego chunka do identyfikacji zapisanych obrazów.

---

## 4. Rejestr istniejących funkcji

### 4.1. Funkcje formatowania i prezentacji

| Funkcja | Aktualny kontrakt | Wynik | Gdzie zdefiniowana |
|---|---|---|---|
| `liczba(x, digits=...)` | Formatuje liczby z polskim przecinkiem. | Tekst. | `setup` |
| `miesiac(x)` | Formatuje datę jako `YYYY-MM`. | Tekst. | `setup` |
| `tabela_raportu(x, podpis, ...)` | Starszy, prostszy wrapper `knitr::kable`. | Tabela. | `setup` |
| `tabela_praca(dane, podpis, ...)` | Rozbudowane formatowanie tabel, źródło i wyróżnienia. | Tabela PDF/HTML. | `funkcja-tabela-praca` |
| `wykres_kolowy(dane, podpis, ...)` | Koło / pierścień z etykietami i źródłem. | Rysunek w raporcie. | `funkcja-wykres-kolowy` |
| `renderuj_rysunki_pracy(wykresy, podpis, ...)` | Przyjmuje **listę ggplotów**, zapisuje i podpisuje. | Rysunek/rysunki PDF/HTML. | `funkcje-wykresow-eda` |
| `wykres_szeregow(dane, ...)` | Dane historii i parametry paneli/kolorów. | Rysunek/rysunki z rendererem. | `funkcje-wykresow-eda` |
| `wykres_sezonowosci(dane, ...)` | Dane sezonowe, gradient, wyróżnienie ostatniego roku. | Rysunek/rysunki z rendererem. | `funkcje-wykresow-eda` |
| `wykres_acf(dane_acf)` | Przyjmuje już obliczoną ACF. **Nie** generuje podpisu samodzielnie. | Obiekt `ggplot`. | `funkcja-wykres-acf` |
| `panele_szeregow(skale="fixed")` | Przygotowuje `facet_wrap` w stylu raportu. | Warstwa ggplot. | `ustawienia-paneli-styl` |

### 4.2. Funkcje pomocnicze do danych

| Funkcja | Znaczenie |
|---|---|
| `z_panelami(x)` | Dołącza metadane `wezly` i konwertuje indeks miesięczny na `miesiac` klasy `Date`. |
| `oblicz_acf(y)` | Liczy ACF pojedynczego szeregu, do trzech sezonów albo dostępnej długości, z `roczne` i `granica`. |
| `zakres(x)` | Tekst opisujący okres dat: początek – koniec. |
| `dodatnia_liczba_calkowita(x)` | Pomocnicza walidacja liczbowych parametrów. |

**Nie wszystkie funkcje powinny korzystać z `renderuj_rysunki_pracy()`.** Dotyczy on rysunków `ggplot`. `tabela_praca()` ma własny mechanizm generowania podpisu i źródła. `wykres_kolowy()` również ma własny mechanizm; nie należy go automatycznie pakować w drugi renderer.

---

## 5. Jak używać obecnych funkcji

### 5.1. `tabela_praca()` — tabelaryczne wyniki

Przykład na danych **demonstracyjnych**, a nie wynikach modeli:

```r
przyklad_miar <- data.frame(
  Model = c("Naive", "SNaive", "ETS", "ARIMA"),
  MAE = c(13.4, 12.2, 10.9, 11.5),
  RMSE = c(18.1, 16.6, 15.2, 16.0),
  MASE = c(1.10, 0.95, 0.83, 0.89)
)

tabela_praca(
  dane = przyklad_miar,
  podpis = "Porównanie błędów prognoz — przykład demonstracyjny",
  zrodlo = "Dane demonstracyjne.",
  wyroznienie = "kolumny",
  kryterium = "min",
  kolumny_oceny = c("MAE", "RMSE", "MASE"),
  miejsca_dz = 2,
  zatrzymaj_pozycje = TRUE
)
```

Tryby `wyroznienie`: `"brak"`, `"wiersz"`, `"kolumny"`, `"wiersze"`; kryterium `"min"` albo `"max"`. Wyróżnianie minimum osobno dla kolumn błędów jest sensowne. **Nie wybieraj minimum między MAE i MASE w jednym wierszu**: miary mają różne jednostki.

### 5.2. `wykres_kolowy()` — struktura udziałów

````markdown
```{r udzialy-kategorii-wykres, results='asis'}
wykres_kolowy(
  dane = udzialy,
  podpis = "Struktura dochodów federalnych USA według kategorii",
  zrodlo = "Opracowanie własne na podstawie danych U.S. Treasury.",
  kategoria = "kategoria",
  wartosc = "lacznie",
  typ = "pierscieniowy",
  kolory = paleta,
  procenty_legenda = FALSE,
  procenty_wycinki = TRUE,
  prog_etykiety = 0.05,
  etykiety_zewnetrzne = TRUE,
  legenda = "right"
)
```
````

Funkcja samodzielnie oblicza udziały z wartości i umieszcza etykiety. Duże udziały mogą być podpisywane wewnątrz, małe na zewnątrz z liniami prowadzącymi. Parametry wyglądu obejmują m.in. `kolor_linii`, `odstep_etykiet`, `grubosc_obramowania`, `styl_etykiet`, `szerokosc`, `wysokosc`. **Nie dodawaj `fig.cap` do takiego chunka.**

### 5.3. `wykres_szeregow()` — przebieg miesięczny

````markdown
```{r historia-grupy, results='asis'}
id_grupy <- wezly |>
  dplyr::filter(poziom == "Grupy") |>
  dplyr::pull(id_wezla)

wykres_szeregow(
  dane = historia,
  id_wezlow = id_grupy,
  podpis = "Miesięczne wpływy w dwóch grupach hierarchii",
  data_walidacji = pierwszy_miesiac_walidacji,
  kolumny = 2,
  wiersze = 1,
  skale = "fixed",
  kolor_linii = wyglad$kolory$obserwacje,
  szerokosc = 7,
  wysokosc = 3.5
)
```
````

Argumenty `skale`:

- `"fixed"` — wspólna skala, właściwa do bezpośredniego porównywania poziomów grup.
- `"free_y"` — osobna oś pionowa, właściwa dla kategorii o bardzo różnych wartościach.

Kolory w istniejącym kodzie: granat dla obserwacji, jasnoniebieski dla walidacji i przerywana linia na granicy treningu. Nie trzeba przypisywać osobnego koloru każdemu panelowi, bo to różne wykresy w osobnych polach.

### 5.4. `wykres_sezonowosci()` — pełne lata treningowe

````markdown
```{r sezonowosc-ogolem, results='asis'}
wykres_sezonowosci(
  dane = sezony,
  id_wezlow = wezly$id_wezla[wezly$poziom == "Ogółem"],
  podpis = "Sezonowość miesięcznych dochodów federalnych USA ogółem",
  kolumny = 1,
  wiersze = 1,
  skale = "fixed",
  kolory = c("#C5D1DD", "#355A85"),
  wyroznij_najnowszy_rok = TRUE,
  mnoznik_grubosci = 1.8,
  przezroczystosc_starszych = 0.65,
  szerokosc = 7,
  wysokosc = 3.4
)
```
````

Każda linia oznacza jeden rok; nowszy rok jest najciemniejszy i może być grubszy. Funkcja nie wyszukuje pełnych lat — **to zadanie obiektu `sezony`**, przygotowanego wcześniej.

### 5.5. `wykres_acf()` — autokorelacja

Ta funkcja **zwraca tylko `ggplot`**, dlatego potrzebne są `facet_wrap()` i wspólny renderer:

````markdown
```{r acf-ogolem, results='asis'}
p_acf <- acf_trening |>
  dplyr::filter(poziom == "Ogółem") |>
  wykres_acf() +
  ggplot2::facet_wrap(ggplot2::vars(panel), ncol = 1)

renderuj_rysunki_pracy(
  wykresy = list(p_acf),
  podpis = "Autokorelacja dochodów federalnych USA ogółem",
  zrodlo = "Opracowanie własne na podstawie danych U.S. Treasury.",
  szerokosc = 7,
  wysokosc = 3.2
)
```
````

Wartości ACF są z zakresu −1 do 1, zatem **wspólna oś Y** jest poprawna. Wyróżnione opóźnienia sezonowe są wielokrotnościami 12 miesięcy. Linie `±1.96/sqrt(n)` mają w tej implementacji **charakter orientacyjny**, przy założeniu białego szumu, a nie uniwersalnych jednoczesnych przedziałów ufności.

### 5.6. `renderuj_rysunki_pracy()` — nowy wykres ggplot

Jeżeli nowa funkcja zwraca jeden lub więcej obiektów `ggplot`, można użyć istniejącego renderera:

```r
renderuj_rysunki_pracy(
  wykresy = list(p1, p2),
  podpis = "Porównanie dwóch zestawów wykresów",
  zrodlo = "Opracowanie własne na podstawie danych U.S. Treasury.",
  szerokosc = 7,
  wysokosc = 4.8,
  zatrzymaj_pozycje = TRUE
)
```

Podczas Knit renderer sam zapisuje grafiki i tworzy osobny podpis dla każdej części. **Nie dodawaj zewnętrznego `fig.cap`**, bo powstaną zdublowane podpisy. Przy generowaniu kolejnych niezależnych rysunków preferuj oddzielne chunki, aby pliki wyjściowe nie korzystały z tych samych nazw.

---

## 6. Jak projektować nowe funkcje

### 6.1. Jedna odpowiedzialność = jedna funkcja

Preferowana architektura:

```text
oblicz_x()            → statystyki / dane wynikowe
przygotuj_x_dane      → ramka danych do wykresu
wykres_x()            → estetyka i układ wykresu
renderuj_rysunki_pracy() → podpis, źródło, zapis PDF/HTML
opis_x                 → interpretacja oparta na wynikach
```

Nie umieszczaj pobierania z API, modelowania, obliczania błędów, rysowania i generowania podpisu w jednej dużej funkcji.

### 6.2. Kontrakt funkcji — co musi być opisane

Każda nowa funkcja powinna określać:

1. **Cel** — co przedstawia i dlaczego jest potrzebna.
2. **Wejście** — dokładne nazwy, typy i znaczenie wymaganych kolumn.
3. **Wyjście** — `tibble`, `ggplot`, lista `ggplot`, `asis_output` lub tabela.
4. **Efekty uboczne** — zapis plików? generowanie PDF? brak zmian obiektu wejściowego?
5. **Parametry wizualizacji** — kolory, rozmiary, grubości, panele, podpis, źródło.
6. **Walidację** — czytelne komunikaty przy brakujących kolumnach, pustych danych, błędnych wartościach.
7. **Zależności** — istniejące funkcje/obiekty i pakiety.
8. **Przykład** — wywołanie na faktycznie istniejącym obiekcie, a przykłady wymyślone wyraźnie oznaczone jako demonstracyjne.

### 6.3. Wzór komentarzy

```r
# ============================================================
# NAZWA FUNKCJI — krótki opis
# ============================================================

#' Jednozdaniowy cel funkcji
#'
#' @param dane Wymagane kolumny i ich typy.
#' @param podpis Podpis rysunku w PDF/HTML.
#' @param zrodlo Źródło pod rysunkiem.
#' @param kolor_linii Kolor danych.
#'
#' @return Obiekt ggplot albo wynik renderowania.
#'
#' @details
#'   * Jak interpretować wykres.
#'   * Jakie są istotne ograniczenia.
#'   * Czy funkcja modyfikuje dane.
#'
#' @examples
#' wykres_przykladowy(dane = gotowe_dane, podpis = "...")

# --- 1. Walidacja -----------------------------------------
# --- 2. Wybór danych --------------------------------------
# --- 3. Budowanie ggplot ----------------------------------
# --- 4. Formatowanie --------------------------------------
# --- 5. Renderowanie --------------------------------------
```

### 6.4. Reguły kompatybilności

- **Nie zmieniaj** istniejącej sygnatury funkcji tylko po to, aby dodać jeden parametr estetyczny; dopisz argument z wartością domyślną.
- Nie usuwaj działających argumentów (`podpis`, `zrodlo`, `kolumny`, `wiersze`, `skale` itd.).
- Nie zmieniaj jednostek w środku funkcji rysującej. Wykresy obecnie używają `mld_usd`.
- Nie nadpisuj globalnych obiektów (`wezly`, `dane_ts`, `trening`, `walidacja`, `paleta`).
- Nie kopiuj całego renderera do każdej nowej funkcji. **Użyj `renderuj_rysunki_pracy()`**.
- Nie twórz drugiej funkcji o tej samej nazwie w dalszej części dokumentu.
- Używaj `ggplot2::` i `dplyr::` tam, gdzie to poprawia przejrzystość zależności.
- Nie używaj `setwd()` ani ścieżek zależnych od prywatnego komputera.

---

## 7. Wzorzec nowej funkcji wizualizacyjnej

Poniższy przykład pokazuje **standard implementacji**, a nie funkcję, która już jest w raporcie. Rysuje słupki porównujące modele dla jednej metryki. Dane demonstracyjne nie są wynikami projektu.

Najpierw definicja w `include=FALSE`:

````markdown
```{r funkcja-wykres-miar, include=FALSE}
# ============================================================
# wykres_miar() — porównanie błędów modeli
# ============================================================

#' Porównanie modeli według jednej miary błędu
#'
#' @param dane Ramka z kolumnami Model i Wartosc.
#' @param podpis Podpis rysunku.
#' @param zrodlo Źródło danych pod rysunkiem.
#' @param jednostka Nazwa i jednostka metryki na osi Y.
#' @param kolor Kolor słupków.
#' @param szerokosc Szerokość grafiki w calach.
#' @param wysokosc Wysokość grafiki w calach.
#'
#' @return W trakcie Knit numerowany rysunek, poza Knit ggplot.
#'
#' @details
#'   * Funkcja nie liczy błędów, wyłącznie je prezentuje.
#'   * Jedna kolumna numeryczna = jedna metryka i jednostka.
#'
#' @examples
#' wykres_miar(gotowe_miary, "Porównanie MASE", jednostka = "MASE")

wykres_miar <- function(
    dane,
    podpis,
    zrodlo = "Opracowanie własne.",
    jednostka = "Wartość miary",
    kolor = wyglad$kolory$obserwacje,
    szerokosc = 7,
    wysokosc = 3.6
) {
  # --- 1. Walidacja ---------------------------------------
  wymagane <- c("Model", "Wartosc")
  if (!is.data.frame(dane) || nrow(dane) == 0L ||
      !all(wymagane %in% names(dane))) {
    stop("Dane muszą zawierać kolumny Model i Wartosc.")
  }
  if (anyNA(dane$Model) || any(!nzchar(as.character(dane$Model))) ||
      !is.numeric(dane$Wartosc) || any(!is.finite(dane$Wartosc))) {
    stop("Niepoprawne nazwy modeli lub wartości metryki.")
  }

  # --- 2. Budowanie wykresu -------------------------------
  p <- ggplot2::ggplot(
    dane,
    ggplot2::aes(x = Model, y = Wartosc)
  ) +
    ggplot2::geom_col(fill = kolor, width = 0.65) +
    ggplot2::scale_y_continuous(
      labels = scales::label_number(decimal.mark = ","),
      expand = ggplot2::expansion(mult = c(0, 0.07))
    ) +
    ggplot2::labs(x = NULL, y = jednostka) +
    motyw_szeregow

  # --- 3. Renderowanie -----------------------------------
  renderuj_rysunki_pracy(
    wykresy = list(p),
    podpis = podpis,
    zrodlo = zrodlo,
    szerokosc = szerokosc,
    wysokosc = wysokosc
  )
}
```
````

Następnie oddzielny chunk z **danymi demonstracyjnymi**:

````markdown
```{r przyklad-miar-dane, include=FALSE}
przykladowe_miary <- tibble::tibble(
  Model = c("Naive", "SNaive", "ETS", "ARIMA"),
  Wartosc = c(1.12, 0.95, 0.84, 0.87)
)
```
````

I dopiero osobny chunk prezentacji:

````markdown
```{r przyklad-miar-wykres, results='asis'}
wykres_miar(
  dane = przykladowe_miary,
  podpis = "Porównanie MASE — przykład demonstracyjny",
  zrodlo = "Dane demonstracyjne; nie są wynikami projektu.",
  jednostka = "MASE"
)
```
````

**W kodzie produkcyjnym** `przykladowe_miary` należy zastąpić poprawnie obliczonymi wartościami na rzeczywistej walidacji. Nie wolno umieszczać demonstracyjnych wyników w końcowej pracy bez oznaczenia.

---

## 8. Standard wyglądu, podpisów i źródeł

### 8.1. Paleta i spójność kolorów

| Znaczenie | Wartość aktualnej konfiguracji |
|---|---|
| Obserwacje rzeczywiste | `#25374A` |
| Tło walidacji | `#E6F0F8` |
| Granica trening/walidacja | `#627A94` |
| ACF zwykłe opóźnienia | `#7C8FA5` |
| ACF opóźnienia sezonowe | `#007A77` |
| Sezonowość — starsze lata | `#C5D1DD` |
| Sezonowość — nowsze lata | `#355A85` |
| Naive | `#9A4D00` |
| SNaive | `#007A60` |
| ETS | `#0067A5` |
| ARIMA | `#85388F` |

Do wykresów kategorii wykorzystywana jest dodatkowa paleta `paleta` (m.in. Oxford Navy `#1B365D`, bursztyn `#D97724`, zieleń `#2E6F40`). **Nie przypisuj nowych przypadkowych kolorów w każdym chunku**. Jeżeli wprowadzisz nowy typ wykresu, najpierw oceń, czy istniejące barwy wystarczą; nowe kolory dodawaj do `wyglad$kolory`.

Kolor powinien mieć sens informacyjny:

- Historia: jeden kolor dla obserwacji, jasne tło walidacji.
- Sezonowość: gradient roczny; najnowszy rok grubszy i na wierzchu.
- ACF: barwa odróżnia opóźnienia sezonowe od pozostałych.
- Prognozy: model odróżniają **zarówno kolor, jak i rodzaj linii**, aby wydruk w skali szarości był nadal użyteczny.
- Porównanie metryk: jeden kolor / skala barw w obrębie tej samej miary; bez wprowadzania fałszywych porównań między jednostkami.

### 8.2. Podpisy i pozycjonowanie w PDF

Bieżący YAML używa `pdf_document`, silnika `xelatex`, `fig_crop: false`, `\usepackage{float}` i czcionki **EB Garamond**. Wspólny renderer korzysta ze środowiska `figure[H]` przy `zatrzymaj_pozycje = TRUE`.

Obowiązujące zasady:

1. **Wykresy z rendererem:** `results='asis'` + `podpis` + `zrodlo`, **bez `fig.cap`**.
2. **Wykres kołowy:** również ma własny podpis — **bez `fig.cap`**.
3. **Zwykły ggplot bez renderera:** może używać `fig.cap`, ale wtedy nie wolno drugi raz dodawać `\caption{...}`.
4. **Tabele:** używaj `tabela_praca()` i jej `podpis`/`zrodlo`, bez ręcznego powielania podpisów.
5. Domyślnie podpisy w języku polskim; nazwy modeli pozostają standardowe: Naive, SNaive, ETS, ARIMA, MinT itd.
6. Unikaj wymuszania `\newpage` po każdym małym rysunku. Warto ograniczać puste strony, choć ostateczny skład trzeba sprawdzić w PDF.

### 8.3. Logiczny porządek paneli

We wszystkich sekcjach hierarchicznych stosuj tę samą kolejność:

1. **Ogółem:** jeden szeroki rysunek.
2. **Grupa 1 i Grupa 2:** dwa panele, zwykle wspólna oś Y, jeśli porównujemy poziomy.
3. **Kategorie Grupy 1:** trzy panele (zwykle `free_y`).
4. **Kategorie Grupy 2:** cztery panele (zwykle `free_y`).

Nie dziel rysunków mechanicznie po cztery węzły, bo miesza to agregaty i kategorie. Dotyczy to przebiegów historycznych, sezonowości, ACF i przyszłych wykresów prognoz.

---

## 9. Przewodnik rozbudowy poszczególnych sekcji

### 9.1. Analiza danych / EDA

**Dane:** `dane_ts`, `skladniki`, `historia`, `sezony`, `wezly`, `udzialy`.  
**Istniejące wykresy:** `wykres_szeregow`, `wykres_sezonowosci`, `wykres_kolowy`.

Rozbudowa EDA może obejmować np. profil miesięczny, obserwacje odstające lub porównanie skal, ale nowa funkcja powinna korzystać z **już przygotowanych danych**, a nie ponownie agregować surowy CSV. Nie opisuj wizualnej obserwacji jako dowiedzionej istotności statystycznej, jeśli nie przeprowadzono właściwego testu.

### 9.2. Autokorelacja i diagnostyka

**Dane:** `acf_trening`, ewentualnie ACF pierwszych różnic przygotowane w `acf-roznic-dane`.  
**Funkcje:** `oblicz_acf`, `wykres_acf`, `renderuj_rysunki_pracy`.

Przy zmianie sposobu obliczania ACF oddziel **zmianę metodologii** od **zmiany kolorów czy paneli**. Nie nazywaj granic `±1.96/sqrt(n)` bezwarunkowo przedziałami ufności dla dowolnego procesu. W analizie ACF różnic uwzględnij, że pierwszy różnicowany szereg ma mniej dostępnych obserwacji.

### 9.3. Modele i prognozy bazowe

**Istniejące:** Naive, SNaive, ETS, ARIMA w `modele_bazowe`, następnie `prognozy_bazowe`.  
**Obecne renderowanie:** chunk `wykresy-prognoz`, jeszcze bez wydzielonej uniwersalnej funkcji z automatycznym podpisem.

Rekomendowana kolejna funkcja: `wykres_prognoz()`, ale **nie jest jeszcze zaimplementowana** w stanie referencyjnym. Powinna przyjmować obliczone obserwacje/prognozy, pozwalać wybrać węzły i modele, utrzymywać kolor + rodzaj linii na model oraz używać wspólnego renderera.

### 9.4. Ocena jakości prognoz

**Do zbudowania:** obliczenia i prezentacja MAE, RMSE, MASE oraz ewentualna walidacja krocząca. W aktualnym stanie referencyjnym nie ma jeszcze pełnej sekcji z obliczonymi miarami.

Proponowana architektura:

```text
oblicz_miary_prognoz()    → metryki po węzłach i modelach
miary-prognoz-dane        → gotowe tabele w układzie długim/szerokim
tabela_praca()            → tabela MAE/RMSE/MASE
wykres_porownania_modeli() → mapa cieplna lub wykres słupkowy
wnioski-z-miar            → interpretacja konkretnych wyników
```

Wszystkie miary muszą być liczone na **tych samych obserwacjach walidacyjnych** dla porównywanych modeli. MASE wymaga jawnie opisanej sezonowej prognozy odniesienia i sposobu obliczenia mianownika na treningu.

### 9.5. Uzgadnianie prognoz hierarchicznych

**Do zbudowania w aktualnym stanie referencyjnym:** Bottom-Up, Top-Down i metody optymalnego uzgadniania (OLS/WLS/MinT) wraz z ich oceną.

Wymagania projektowe:

- Wykorzystać **jedną ustaloną hierarchię** i tę samą definicję węzłów dla wszystkich metod.
- Jawnie sprawdzić **spójność addytywną** prognoz po uzgodnieniu.
- Porównywać jakość bazowych i uzgodnionych prognoz dla tych samych węzłów, dat i miar.
- Nie deklarować poprawy, jeśli dane pokazują pogorszenie lub brak wyraźnej różnicy.
- Nie mylić spójności hierarchicznej z dokładnością prognoz.
- W razie dodawania MinT jasno określić sposób oszacowania macierzy kowariancji błędów i założenia metody.

---

## 10. Reguły metodologiczne i integralność danych

1. **Brak przecieku informacji:** dopasowanie modeli i estymacja parametrów wykorzystywanych do prognoz nie mogą korzystać z okresu walidacji. Wyniki walidacji służą do oceny i ewentualnego porównania procedur, nie do dopasowania prognozy ocenianej na tym samym okresie bez właściwego schematu eksperymentu.
2. **Jednakowy horyzont:** wszystkie modele porównujemy na równych datach i węzłach.
3. **Jednostka:** `mld_usd`; nie mieszać USD, mln i mld bez jawnego przeliczenia.
4. **Hierarchia:** suma kategorii → suma grup → ogółem, w każdym miesiącu.
5. **Pełne lata:** jeśli analiza sezonowości opiera się na pełnych latach, nie dołączaj częściowych lat bez uzasadnienia.
6. **Rozdzielenie próby:** dane treningowe i walidacyjne mają własne obiekty; nie tworzymy ich ponownie z innych, potencjalnie niespójnych reguł.
7. **Bez cichych poprawek:** jeżeli obliczenia nie zgadzają się z wcześniejszym opisem, wskaż rozbieżność zamiast dopasowywać kod do oczekiwanego wyniku.
8. **Odporność na aktualizację:** liczby miesięcy, daty, wartości procentowe i opis dominujących kategorii wyznaczaj z aktualnych danych, o ile to możliwe.
9. **Rozróżnienie źródeł:** dane U.S. Treasury są zewnętrzne, ale podział na Grupy 1 i 2 oraz wykresy są opracowaniem autorów.
10. **Wnioski oparte na wynikach:** nie pisz, że ETS/ARIMA/MinT jest „najlepszy” bez określenia metryki, danych i horyzontu.

---

## 11. Testowanie i diagnostyka błędów

### 11.1. Minimalna ścieżka weryfikacji zmiany

1. Zapisz kopię pliku `.Rmd` przed modyfikacją.
2. Sprawdź, czy definicja funkcji występuje tylko raz.
3. Sprawdź unikalność etykiet chunków.
4. Wykonaj chunki zależności, następnie definicję, obliczenia i prezentację.
5. Sprawdź przykładowe wywołanie dla pojedynczego węzła i większego zestawu.
6. Zweryfikuj nie tylko brak błędu, lecz także poprawność **wartości, jednostek, podpisów, źródeł i osi**.
7. Uruchom pełny **Knit od początku w czystej sesji R**.
8. Obejrzyj wynikowy PDF: brak nakładających się etykiet, brak zdublowanych rysunków/podpisów, niepuste wykresy, rozsądne łamanie stron.

Przy sprawdzaniu błędów warto w R wykonać:

```r
exists("wykres_acf")
exists("acf_trening")
exists("renderuj_rysunki_pracy")
exists("motyw_szeregow")

# Nazwy/typy kolumn wejściowych:
str(historia)
str(sezony)
str(acf_trening)

# Struktura poziomów hierarchii:
wezly |>
  dplyr::count(poziom)
```

### 11.2. Najczęstsze błędy

| Komunikat lub objaw | Co sprawdzić | Jak naprawić |
|---|---|---|
| `could not find function "wykres_acf"` | Czy wykonał się `funkcja-wykres-acf`? | Umieść definicję przed wywołaniem; nie używaj `eval=FALSE`. |
| `object 'acf_trening' not found` | Czy `dane-acf` jest wcześniej niż `acf-ogolem`? | Przenieś obliczenia przed rysunek. |
| `object 'motyw_szeregow' not found` | Czy wykonano `ustawienia-paneli-styl`? | Nie wywołuj wykresów przed definicją motywu. |
| `object 'historia' not found` | Czy wykonano `historia-dane`? | Przygotuj wejście przed wykresem. |
| `object 'sezony' not found` | Czy wykonano `sezonowosc-dane` i `pelne_lata`? | Sprawdź zależności i pełne lata treningowe. |
| `Duplicate chunk label` | Czy po reorganizacji pozostał stary chunk? | Usuń duplikat lub zmień etykietę. |
| Dwa podpisy pod rysunkiem | Czy jest `fig.cap` oraz `renderuj_rysunki_pracy`? | Zostaw tylko mechanizm renderera. |
| Zbyt małe wykresy | `szerokosc`, `wysokosc`, liczba paneli, legenda | Zmień rozmiar lub podziel na rysunki według poziomów. |
| Wykresy mieszają ogółem z kategoriami | Mechaniczny podział `zestawy_paneli` | Oddziel ogółem → grupy → Grupa 1 → Grupa 2. |
| Wyniki po Knit inne niż po „Run” | Obiekty pozostałe w interaktywnej sesji | Uruchom czystą sesję i Knit od początku. |
| Brak PDF | Ścieżka CSV, pakiety, XeLaTeX, EB Garamond | Zweryfikuj zależności systemowe i konkretny log. |

### 11.3. Co powinien zawierać raport z weryfikacji zmiany

```text
Zmiana: [krótki opis]
Plik: [nazwa Rmd]
Chunki zmodyfikowane: [...]
Nowe funkcje: [...]
Zmodyfikowane interfejsy: [...]
Zależności: [...]
Testy wykonane: [jakie rzeczywiście uruchomiono]
Wynik Knit PDF: [PASS / FAIL / NIE URUCHOMIONO]
Znane ograniczenia: [...]
Potwierdzenie braku zmian metodologii: [TAK/NIE + zakres]
```

**Nie wolno deklarować „Knit działa”, jeśli nie przeprowadzono pełnego renderowania.** Jeśli brak R, pakietów, CSV lub LaTeX, podaj precyzyjnie, czego nie dało się zweryfikować.

---

## 12. Gotowe prompty do ChatGPT / Codexa

### 12.1. Uniwersalny prompt — dodanie nowej funkcji

Skopiuj i uzupełnij fragmenty w nawiasach kwadratowych:

```text
Pracujemy nad raportem inżynierskim dotyczącym prognozowania
hierarchicznych szeregów czasowych w R Markdown.

Załączam aktualny plik .Rmd oraz README.md projektu. Przed
zaproponowaniem zmian PRZECZYTAJ rzeczywiste definicje funkcji,
obiektów, nazwy chunków i miejsca ich użycia. Źródłem prawdy jest
aktualny .Rmd, nie poprzednia wersja kodu ani przykłady w README.

Zadanie: dodaj funkcję [NAZWA] do [CEL].
Dane wejściowe: [NAZWY ISTNIEJĄCYCH OBIEKTÓW I KOLUMN].
Oczekiwany wynik: [TABELA / GGPLOT / WIELE WYKRESÓW].
Zakres zmiany: [KONKRETNA SEKCJA/CHUNK].

Wymagania:
1. Zachowaj jeden samowystarczalny plik .Rmd; nie twórz pliku .R.
2. Oddziel obliczenia (include=FALSE) od wyświetlania (results='asis').
3. Nie zmieniaj istniejących danych, hierarchii, modeli ani podziału
   trening/walidacja, jeśli nie jest to konieczne i uzasadnione.
4. Wykorzystuj obecne funkcje i `wyglad` / `motyw_szeregow`.
5. Nowe wykresy ggplot z podpisem renderuj przez
   `renderuj_rysunki_pracy()`; nie dodawaj wtedy fig.cap.
6. Dodaj pełną dokumentację @param, @return, @details, @examples
   oraz ponumerowane komentarze wewnątrz funkcji.
7. Zwaliduj wejścia, zadbaj o czytelne błędy i zgodność ze
   starszymi wywołaniami.
8. Zachowaj porządek: ogółem, grupy, kategorie Grupy 1 i 2.
9. Nie wymyślaj żadnych wyników danych ani wartości metryk.
10. Nie deklaruj wykonania testów, których naprawdę nie uruchomiono.

Najpierw zwięźle opisz projekt proponowanej funkcji i zależności,
a potem dostarcz gotowe chunki do wklejenia, ich dokładną lokalizację,
krótki przykład użycia i checklistę testów.
```

### 12.2. Prompt — poprawa istniejącej funkcji bez przebudowy

```text
Przeanalizuj aktualny .Rmd i README.md. Chcę zmienić wyłącznie
funkcję [NAZWA] w zakresie: [KONKRETNA ZMIANA, np. grubość linii,
kolory, etykiety lub kolejność paneli].

Zachowaj jej istniejącą sygnaturę, nazwy argumentów, sposób
wywołania, podpisy PDF/HTML i obecne zachowanie domyślne,
chyba że wprost wskażę inny cel. Jeśli dodajesz nowy argument,
zapewnij wartość domyślną i zaktualizuj dokumentację.

Nie zmieniaj obliczeń statystycznych ani pozostałych chunków.
Pokaż dokładnie: które linie/sekcje podmienić, pełną poprawioną
definicję funkcji i przykład kompatybilnego wywołania.
Wypisz ewentualne ryzyka i co przetestować.
```

### 12.3. Prompt — uporządkowanie wykresów według hierarchii

```text
Na podstawie aktualnego .Rmd przeorganizuj wykresy w sekcji
[HISTORIA / SEZONOWOŚĆ / ACF / PROGNOZY] według poziomów:
1. Ogółem — 1 panel;
2. Grupa 1 i Grupa 2 — 2 panele;
3. Kategorie Grupy 1 — 3 panele;
4. Kategorie Grupy 2 — 4 panele.

Nie przeliczaj danych. Wykorzystaj `wezly`, `id_wezla` i już
istniejącą funkcję rysującą. Każdy logiczny zestaw ma otrzymać
własny chunk, podpis i źródło. Wspólne skale stosuj tylko wtedy,
gdy porównywane wielkości mają sensownie porównywalną skalę.
Usuń stary, zbędny chunk wykresów, aby uniknąć duplikatów.
Sprawdź kolejność definicji i przygotowania danych.
```

### 12.4. Prompt — nowa analiza / model / metoda uzgadniania

```text
Na podstawie aktualnego raportu .Rmd i README.md zaprojektuj
rozszerzenie o [NAZWA METODY / ANALIZY].

Najpierw wypisz:
- pytanie badawcze i uzasadnienie,
- dokładne wejścia i wyjścia,
- założenia statystyczne,
- sposób unikania przecieku danych z walidacji,
- metryki i plan porównania z obecnym benchmarkiem,
- testy spójności hierarchicznej (jeśli dotyczy),
- ryzyko naruszenia istniejących obiektów.

Następnie dodaj osobne chunki: obliczenia include=FALSE,
prezentacja tabeli/wykresu, komentarz do otrzymanych wyników.
Nie modyfikuj istniejących modeli w miejscu; nową metodę
zaimplementuj i zweryfikuj niezależnie, dopóki jej działanie
nie zostanie potwierdzone. Nie wymyślaj wyników.

Na koniec podaj dokładną listę zmodyfikowanych chunków,
testy i ewentualne ograniczenia wykonania.
```

### 12.5. Prompt — debugowanie pliku po błędzie Knit

```text
Załączam aktualny .Rmd i pełny log błędu Knit.
Znajdź rzeczywistą przyczynę błędu, a nie tylko zgaduj na
podstawie ostatniej wiadomości z konsoli.

Sprawdź:
- kolejność definicji funkcji i obiektów,
- nieistniejące kolumny lub źle dobrane typy,
- powtarzające się etykiety chunków,
- błędne użycie include/eval/results/fig.cap,
- ścieżki do plików, pakiety i zależności LaTeX,
- ewentualne używanie obiektów z poprzedniej sesji RStudio.

Podaj: przyczynę, numer lub nazwę chunka, minimalną poprawkę,
dlaczego działa oraz czego nie dało się przetestować.
Nie przepisuj całego raportu, jeśli wystarczy zmienić kilka linii.
```

### 12.6. Prompt — przegląd jakości końcowej pracy

```text
Wykonaj audyt aktualnego .Rmd jako raportu inżynierskiego.
Nie dopisuj nowych wyników i nie zmieniaj kodu na tym etapie.

Oceń osobno: poprawność metodologiczną, integralność danych,
spójność hierarchii, brak przecieku walidacji, czytelność wykresów,
interpretację wyników, powtarzalność analiz, strukturę sekcji,
spójność podpisów i dokumentację funkcji.

Zwróć problemy w kolejności: BŁĄD / WAŻNE / KOSMETYCZNE,
każdy z lokalizacją, uzasadnieniem i minimalną rekomendacją.
Wyraźnie oddziel pewne błędy od hipotez wymagających uruchomienia R.
```

---

## 13. Checklista przed przyjęciem zmian

- [ ] Rozumiem potrzebę zmiany i wiem, w której sekcji ma się pojawić.
- [ ] Sprawdziłem aktualny `.Rmd`; nie opieram się wyłącznie na README.
- [ ] Nowa funkcja nie duplikuje istniejącej.
- [ ] Nazwa nowego chunka jest unikatowa.
- [ ] Dane wejściowe są dostępne przed wywołaniem funkcji.
- [ ] Dane przygotowuję w `include=FALSE`, rysuję w `results='asis'`.
- [ ] Przy rendererze nie dodaję `fig.cap`.
- [ ] Funkcja ma komentarze `@param`, `@return`, `@details`, `@examples`.
- [ ] Funkcja ma kontrolę typów, kolumn, braków i niepoprawnych parametrów.
- [ ] Nie zmieniam domyślnego zachowania istniejących wywołań bez uzasadnienia.
- [ ] Kolory i stylistyka są zgodne z `wyglad` oraz `motyw_szeregow`.
- [ ] Wykresy hierarchiczne zachowują kolejność ogółem → grupy → kategorie.
- [ ] Jednostki, daty i okresy treningu/walidacji są poprawne.
- [ ] Nowe obliczenia nie wykorzystują przyszłych obserwacji podczas dopasowania.
- [ ] Sprawdziłem wynik w czystej sesji i pełny Knit, jeśli mam środowisko.
- [ ] Obejrzałem PDF pod kątem rozmiarów, etykiet, podpisów i źródeł.
- [ ] Zaktualizowałem README, jeśli zmienił się interfejs lub zależności funkcji.

---

## 14. Co jest zaimplementowane, a czego jeszcze brakuje

### Istnieje w pliku referencyjnym

- Import, normalizacja i walidacja CSV; agregacja kodów do kategorii.
- Hierarchia 1 → 2 → 7 oraz kontrola względem sumy opublikowanej.
- Podział na trening i walidację, sezonowość i ACF.
- `tabela_praca()`, `wykres_kolowy()`, `wykres_szeregow()`, `wykres_sezonowosci()`, `wykres_acf()` oraz wspólny renderer.
- Prezentacja historii, sezonowości i ACF w logicznych grupach hierarchicznych.
- Dopasowanie czterech modeli: Naive, SNaive, ETS i ARIMA; wykresy prognoz bazowych.

### Planowane / nieobecne w stanie referencyjnym

- Pełne obliczenie i zestawienie MAE/RMSE/MASE po modelach i węzłach.
- Funkcja `wykres_prognoz()` zastępująca dotychczasowy kod pętli.
- Funkcja `wykres_porownania_modeli()` (np. mapa błędów).
- Metody uzgadniania Bottom-Up / Top-Down / OLS / WLS / MinT wraz z obliczeniami i walidacją.
- Porównanie prognoz bazowych i uzgodnionych, test spójności.
- Rolling-origin / walidacja krocząca, analiza odporności wyników.

Przed zgłoszeniem nowego zadania warto zdecydować, czy chodzi o **samą prezentację istniejących wyników**, czy o **nową procedurę statystyczną**. Te zadania mają różne ryzyka i nie powinny być łączone bez potrzeby.

---

## Dziennik aktualizacji README

| Data | Wersja | Zmiana |
|---|---|---|
| 2026-11-15 | 1.0 | Pierwsza instrukcja rozwoju, przygotowana przez inspekcję bieżącego `.Rmd` i istniejących funkcji. |

**Status weryfikacji dokumentacji:** sprawdzono strukturę, sygnatury i zależności w pliku źródłowym. Nie wykonano pełnego Knit ani testów numerycznych w R (w środowisku przygotowania README brak `Rscript` i danych CSV). README jest instrukcją pracy z kodem, nie raportem potwierdzającym jego poprawne wykonanie.
