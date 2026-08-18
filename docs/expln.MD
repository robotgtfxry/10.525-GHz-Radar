# Sektorowy radar dopplerowski 10,525 GHz

**Wykrywanie i lokalizacja obiektów ruchomych z analizą sygnatury mikrodopplerowskiej**

Wersja dokumentacji: 1.0 · Data: 18 sierpnia 2026

---

## 1. Cel projektu

Zbudowanie radaru dopplerowskiego o pokryciu pełnych 360°, zdolnego wykryć obiekt ruchomy, określić sektor, w którym się znajduje, oraz — na podstawie kształtu widma prędkości — odróżnić od siebie różne klasy obiektów.

Konstrukcja opiera się na ośmiu modułach HB100 rozmieszczonych promieniście, przełączanych sekwencyjnie. Efekt obrotu wiązki uzyskiwany jest elektronicznie, bez części ruchomych.

### Co system potrafi

- wykrywa ruch w pełnym zakresie 360°
- wskazuje sektor (rozdzielczość ~45°, po interpolacji ~15°)
- mierzy prędkość radialną obiektu
- rozróżnia klasy obiektów po kształcie widma (człowiek / pojazd / wirnik)

### Czego nie potrafi — i dlaczego

**Nie mierzy odległości.** Moduł HB100 pracuje w trybie fali ciągłej bez modulacji. Nie ma czasu przelotu, który dałoby się zmierzyć, więc informacja o dystansie fizycznie nie istnieje w odbieranym sygnale. Jest to ograniczenie zasadnicze, nie kwestia oprogramowania.

**Nie odróżnia zbliżania od oddalania.** Moduł ma jedno wyjście pośredniej częstotliwości, bez kanałów I/Q. Widmo jest symetryczne, więc dostępny jest moduł prędkości, nie jej znak. Rozwiązaniem byłby moduł z wyjściem kwadraturowym.

---

## 2. Podstawy fizyczne

### Efekt Dopplera

Fala odbita od obiektu poruszającego się względem radaru wraca ze zmienioną częstotliwością. Różnica między nadaną a odebraną częstotliwością wynosi:

```
F_d = 2·V·(F_t / c)·cos θ
```

gdzie `V` to prędkość obiektu, `F_t` częstotliwość nadawania, `c` prędkość światła, `θ` kąt między kierunkiem ruchu a osią modułu.

Dla `F_t = 10,525 GHz` wzór upraszcza się do:

```
F_d = 19,49 · V     [V w km/h]
F_d = 70,2  · V     [V w m/s]
```

**Wniosek praktyczny:** cały interesujący nas zakres prędkości mieści się w paśmie akustycznym. Człowiek idący 1,4 m/s daje 98 Hz. Samochód 50 km/h daje 975 Hz. Końcówka śmigła drona przy 53 m/s daje 3,7 kHz.

### Bilans energetyczny

Równanie radarowe dla monostatycznego układu:

```
P_r = P_t · G² · λ² · σ / ((4π)³ · R⁴)
```

Dla HB100: `P_t ≈ 20 mW`, `G ≈ 8 dBi`, `λ = 2,85 cm`.

| Obiekt | RCS (σ) | Zasięg szacunkowy |
|---|---|---|
| Człowiek | ~1 m² | 15–20 m |
| Samochód | ~10 m² | 30–40 m |
| Dron (korpus) | ~0,01 m² | ~7 m |
| Śmigło drona | ~10⁻⁴ m² | 2–3 m |

Nota aplikacyjna modułu podaje punkt odniesienia zgodny z tym rachunkiem: parametr RSS określany jest dla przesunięcia dopplerowskiego 25 Hz przy tłumieniu toru 93 dB, co odpowiada człowiekowi w odległości 15 m idącemu wprost na moduł z prędkością 1,28 km/h. Kontrola: 1,28 × 19,49 = 25 Hz.

Zależność `R⁴` oznacza, że **czterokrotny wzrost zasięgu wymaga 256-krotnego wzrostu mocy**. Przy stałej mocy modułu jedyną drogą do większego zasięgu jest wydłużenie okna analizy (podwojenie okna to +3 dB) albo antena o większym zysku — ta druga droga wyprowadza jednak poza dopuszczalne EIRP.

---

## 3. Architektura systemu

```
     M1 (0°)
 M8          M2
     ┌─────┐
M7   │ CPU │   M3          8 modułów co 45°
     └─────┘                pierścień ~30 cm
 M6          M4
     M5 (180°)
```

### Ścieżka sygnału

```
HB100 → wzmacniacz 100×  →  74HC4051  →  wzm. 2  →  filtr  →  ESP32
       (na płytce sektora)   (mux 8:1)    (~40 dB)   4,8 kHz   GPIO34
                                 ↑
                            GPIO 25/26/27
```

### Dobór liczby modułów

Szerokość wiązki HB100 w azymucie wynosi około 80° (poziom połowy mocy). Przy odstępie 45° wiązki sąsiednich modułów silnie na siebie zachodzą.

Nie jest to wada. Obiekt widziany jednocześnie przez dwa moduły z różną amplitudą pozwala interpolować kąt znacznie dokładniej niż wynika z samego podziału na sektory — z 45° do około 15°.

### Budżet czasowy

| Parametr | Wartość |
|---|---|
| Czas na sektor (dwell) | 50 ms |
| Pełny obrót | 8 × 50 ms = 400 ms |
| Częstotliwość odświeżania | 2,5 skanu/s |
| Rozdzielczość FFT | 20 Hz |
| Rozdzielczość prędkości | 0,28 m/s |
| Zakres prędkości | do 68 m/s |

Wydłużenie okna do 100 ms daje 3 dB zysku czułości kosztem połowy prędkości odświeżania.

### Dlaczego multipleksowanie sygnału, a nie zasilania

Nota aplikacyjna dopuszcza pracę impulsową modułu (PRF 2 kHz, wypełnienie 4%), co oznacza, że oscylator DRO stabilizuje się w mikrosekundach. Przełączanie zasilania byłoby więc technicznie możliwe.

Zdecydowano jednak o multipleksowaniu **wyjścia sygnału** przy stale zasilanych modułach, ponieważ:

- praca impulsowa wymagałaby układu próbkująco-pamiętającego na każdym wyjściu
- oszczędność poboru prądu (320 mA) nie ma tu żadnego znaczenia
- eliminuje transjenty od przełączanego zasilania

---

## 4. Płytka sektora — układ elektryczny

Każdy moduł ma własną płytkę wzmacniacza montowaną bezpośrednio na goldpinach modułu.

### Schemat blokowy

```
HB100 pin IF ──┬── R1 12k ── GND
               │
               └── C1 10µF ── R2 10k ──┬── U1A pin 2 (−)
                                       │
                          R3 1M ───────┤
                          C2 33pF ─────┤
                                       │
              VREF ── U1A pin 3 (+)    │
                                       │
                        U1A pin 1 ─────┴── C5 1µF ── OUT

   Dzielnik odniesienia:
   +3V3 ── R4 100k ──┬── R5 100k ── GND
                     ├── C3 10µF ── GND
                     └── U1B pin 5 (+)
                         U1B pin 6 ── pin 7 (wtórnik) ── VREF
```

### Parametry toru

| Parametr | Wartość | Skąd |
|---|---|---|
| Wzmocnienie | 100× | R3 / R2 = 1M / 10k |
| Górna granica pasma | 4,8 kHz | 1 / (2π · R3 · C2) |
| Dolna granica pasma | ~1,6 Hz | 1 / (2π · R2 · C1) |
| Napięcie odniesienia | 1,65 V | dzielnik R4/R5 z 3,3 V |

### Kluczowe odstępstwo od noty aplikacyjnej

**C2 = 33 pF zamiast 2,2 nF.**

Układ z Aneksu 1 noty aplikacyjnej ma w pętli sprzężenia zwrotnego 1 MΩ równolegle z 2,2 nF, co daje górną granicę pasma:

```
f = 1 / (2π · 10⁶ · 2,2·10⁻⁹) ≈ 72 Hz
```

Producent celowo ograniczył pasmo, ponieważ moduł projektowano do wykrywania ludzi, a chodzący człowiek daje przesunięcie poniżej 100 Hz. Dla naszego zastosowania oznaczałoby to stłumienie sygnatury śmigieł (3,7 kHz) o około 34 dB.

Po zmianie na 33 pF:

```
f = 1 / (2π · 10⁶ · 33·10⁻¹²) ≈ 4,8 kHz
```

To jest **najważniejsza modyfikacja w całym projekcie**. Bez niej podstawowa funkcja badawcza układu nie działa.

### Dlaczego dwa napięcia zasilania

Moduł HB100 wymaga 5 V. Wzmacniacz zasilany jest z 3,3 V.

Powód jest ochronny: wyjście wzmacniacza trafia na przetwornik ADC procesora ESP32, którego wejście toleruje maksymalnie 3,3 V. Przy wzmocnieniu 100× wzmacniacz wchodzi w nasycenie przy silniejszym echu i wystawia napięcie równe zasilaniu. Zasilany z 3,3 V nie jest fizycznie w stanie uszkodzić wejścia procesora.

### Rola drugiej połówki wzmacniacza

Układ MCP6002 zawiera dwa wzmacniacze. Pierwszy wzmacnia sygnał, drugi pracuje jako **wtórnik napięciowy buforujący odniesienie 1,65 V**.

Bez bufora odniesienie miałoby impedancję równą równoległemu połączeniu R4 i R5, czyli 50 kΩ. Bufor obniża ją do ułamków oma, co zmniejsza skoki punktu pracy przy przełączaniu multipleksera i uodparnia układ na zakłócenia wnikające przez linię odniesienia.

---

## 5. Wykaz elementów

### Na jedną płytkę sektora (× 8)

| Oznaczenie | Wartość | Uwagi |
|---|---|---|
| U1 | MCP6002-I/P | PDIP-8, rail-to-rail, od 1,8 V |
| R1 | 12 kΩ | obciążenie wyjścia IF |
| R2 | 10 kΩ | wejście toru |
| R3 | 1 MΩ | sprzężenie zwrotne |
| R4, R5 | 100 kΩ | dzielnik odniesienia |
| C1 | 10 µF bipolarny | sprzężenie IF |
| C2 | **33 pF** ceramiczny | ograniczenie pasma |
| C3 | 10 µF | filtracja odniesienia |
| C4 | 100 nF ceramiczny | odsprzęganie zasilania U1 |
| C5 | 1 µF | sprzężenie wyjścia |
| — | goldpiny 4+4 | montaż na module |

### Elektronika centralna

| Element | Ilość | Funkcja |
|---|---|---|
| ESP32 (płytka rozwojowa) | 1 | akwizycja i przetwarzanie |
| 74HC4051 | 1 | multiplekser analogowy 8:1 |
| MCP6002-I/P | 1 | drugi stopień + filtr |
| AMS1117-3.3 lub moduł LDO | 1 | zasilanie analogowe |
| Powerbank 5 V / 2 A | 1 | zasilanie systemu |
| Dławik 100 µH + C 100 µF | 1 | filtr LC przed LDO |
| Rezystory, kondensatory | — | wg drugiego stopnia |

### Mechanika

| Element | Ilość |
|---|---|
| Płyta bazowa ⌀ 30 cm | 1 |
| Blacha aluminiowa 1 mm | ~30 × 30 cm (8 przegród) |
| Przewód 4-żyłowy | 8 × ~30 cm |
| Złącza JST-XH 4-pin | 8 par |

### Uwagi zakupowe

- **MCP6001 nie istnieje w obudowie przewlekanej** — nota katalogowa wymienia dla niego wyłącznie SC70-5 i SOT-23-5. Z tej rodziny przewlekane są tylko MCP6002 (PDIP-8) i MCP6004 (PDIP-14).
- Zamienniki o zgodnym rozkładzie nóżek: MCP6022-I/P, MCP6292-E/P.
- **Nie stosować LM324** (nie dociągnie wyjścia przy zasilaniu 3,3 V) ani **TL074** (wymaga minimum 7 V).
- **Nie kupować gotowych płytek wzmacniaczy do HB100** — wszystkie mają fabryczne 2,2 nF i pasmo 70 Hz.
- **Nie stosować ADS1115** — 860 próbek/s to o rząd wielkości za mało.

---

## 6. Zasady konstrukcyjne

### Reżim ścieżki mikrowoltowej

Sygnał na wyjściu IF ma poziom mikrowoltów. Przewód zbiera z otoczenia kilkanaście mikrowoltów zakłóceń na każde 20 cm długości.

| Odcinek | Poziom sygnału | Wymagania |
|---|---|---|
| IF → C1 | ~1 µV | **poniżej 2 cm**, masa tuż obok |
| Wyjście U1A → mux | ~100 µV | zwykły przewód, do 50 cm |
| Za drugim stopniem | ~10 mV | bez ograniczeń |

Cała logika sprowadza się do zasady: **wzmacniamy sygnał, nie wzmacniamy zakłóceń złapanych po wzmocnieniu**. Dlatego pierwszy stopień musi znajdować się przy module — wszystko złapane przed nim zostanie pomnożone razem z sygnałem.

### Masa gwiazdowa

Na schemacie wszystkie symbole masy tworzą jeden net. Fizycznie masa ma niezerową rezystancję, a przez wspólny odcinek płynie prąd ośmiu modułów (320 mA), co przy sygnale mikrowoltowym tworzy sprzężenie przez wspólną impedancję.

Rozwiązanie: jeden fizyczny punkt zbiorczy w centrum, z którego wychodzą trzy niezależne gałęzie:

```
            [PUNKT GWIAZDY]
           /       |       \
    masa modułów  masa    masa ESP32
    (duży prąd)  analogu  (cyfrowa)
```

Do każdego modułu prowadzić **osobną parę** zasilania i masy — nie łańcuchem od modułu do modułu.

### Odsprzęganie

Kondensator C4 (100 nF, ceramiczny) musi znajdować się bezpośrednio przy nóżkach 8 i 4 układu U1 — w odległości milimetrów. Dla napięcia stałego stanowi przerwę, ale przy 1 MHz jego reaktancja spada do ~1,6 Ω, więc zwiera zakłócenia do masy i stanowi lokalny zbiornik ładunku dla wzmacniacza.

Na schemacie narysowany jest z boku, połączony przez symbole zasilania. To jest przypadek, w którym schemat i płytka mówią co innego — schemat mówi „połączone", płytka musi realizować „połączone krótko".

### Przegrody międzysektorowe

Blacha aluminiowa promieniowo między modułami, wystająca 3–4 cm przed płaszczyznę anten. Bez niej moduł odbiera bezpośrednie promieniowanie sąsiadów, o 40–60 dB silniejsze od echa.

### Montaż na module

Nota aplikacyjna wprost ostrzega: nacisk lub naprężenie obudowy modułu powoduje nieodwracalne pogorszenie parametrów. Płytka wzmacniacza ma opierać się wyłącznie na goldpinach, bez skręcania przez korpus.

Płytkę montować **od strony przeciwnej do anten** — nie zasłania promieniowania, a jej miedź działa jako dodatkowy ekran od tyłu.

---

## 7. Przetwarzanie sygnału

### Akwizycja

Górna granica pasma 4,8 kHz wymusza próbkowanie co najmniej 10 kSPS. Zalecane 12–16 kSPS przez interfejs I2S z DMA — zwykła funkcja `analogRead()` w pętli ma drżenie taktowania, które degraduje FFT.

Wykorzystać **ADC1** (GPIO 32–39). ADC2 nie działa przy aktywnym WiFi.

Sterowanie multiplekserem: GPIO 25, 26, 27 (adres A/B/C).

### Sekwencja

1. Ustaw adres sektora na multiplekserze
2. **Odrzuć pierwsze 15 ms** — stała czasowa sprzężenia AC za multiplekserem
3. Zbieraj próbki przez pozostałe 35 ms
4. Okno Hanninga, FFT
5. Filtr zaporowy 100 Hz i 200 Hz (świetlówki)
6. Przejdź do kolejnego sektora

### Filtracja sieciowa

Nota aplikacyjna ostrzega przed zakłóceniami od świetlówek — przy sieci 50 Hz jest to 100 Hz, dokładnie w paśmie Dopplera chodzącego człowieka. Wąski filtr zaporowy IIR (Q ≈ 30) na 100 i 200 Hz usuwa zakłócenie kosztem zaledwie 1,5 cm/s zakresu prędkości.

Zasilanie z powerbanku zamiast zasilacza sieciowego eliminuje drugie źródło tego samego zakłócenia.

### Klasyfikacja obiektów

Kształt widma niesie informację o typie obiektu:

| Obiekt | Sygnatura widmowa |
|---|---|
| Człowiek | szeroki, rozmyty grzbiet do ~200 Hz (ruch kończyn) |
| Pojazd | wąski, silny pojedynczy prążek |
| Dron | prążek korpusu + symetryczne wstęgi 3–4 kHz od śmigieł |

Sygnatura śmigła wynika z tego, że łopatka nacierająca i oddalająca dają przesunięcia o przeciwnych znakach, więc widmo jest symetryczne względem prążka korpusu. Dla śmigła 5-calowego przy 8000 obr./min prędkość końcówki wynosi:

```
v = π · D · n / 60 = π · 0,127 · 8000 / 60 ≈ 53 m/s
F_d = 53 · 70,2 ≈ 3,7 kHz
```

Jest to technika znana w literaturze jako analiza mikrodopplerowska.

---

## 8. Wizualizacja

### Wyświetlacz sektorowy

Ponieważ system nie mierzy odległości, klasyczny wskaźnik PPI z pierścieniami dystansu byłby wprowadzeniem w błąd. Zastosowano rozwiązanie zachowujące formę radaru przy zachowaniu uczciwości pomiaru:

**Oś promieniowa reprezentuje prędkość, nie odległość.**

```
       środek = 0 m/s
       pierścienie: 1 — 2 — 3 — 4 m/s
       kąt wycinka = sektor
       jasność = amplituda
```

W każdym z ośmiu wycinków po 45° rysowane jest widmo FFT wzdłuż promienia. Obiekt wolny daje plamę blisko środka, szybki — dalej od środka. Przeliczenie osi: 70,2 Hz na 1 m/s.

### Panel spektrogramu

Wybrany sektor prezentowany jako spektrogram: czas poziomo, częstotliwość pionowo, amplituda kolorem. Umożliwia bezpośrednie porównanie sygnatur różnych obiektów.

### Wskaźnik pomocniczy

Pierścień diod WS2812 pod modułami, jasność proporcjonalna do energii w sektorze. Czytelny z odległości, nie wymaga patrzenia w ekran, działa jako natychmiastowa informacja zwrotna dla obserwatora.

---

## 9. Zgodność z przepisami

Moduł HB100 pracuje na 10,525 GHz jako urządzenie małej mocy (LPRD). W Unii Europejskiej obowiązuje norma zharmonizowana **EN 300 440**, ograniczająca moc do 25 mW EIRP.

Moduł z fabryczną anteną patch mieści się w tym limicie i nie wymaga indywidualnego pozwolenia radiowego.

**Warunek zachowania zgodności:** nie wolno dołączać anteny o większym zysku (czaszy, tuby, siatki reflektorowej) ani zewnętrznego wzmacniacza mocy. Każdy z tych zabiegów zwiększa EIRP ponad dopuszczalny poziom.

Nota aplikacyjna zaznacza, że mimo zwolnienia z licencji użytkownika mogą obowiązywać wymagania dotyczące homologacji sprzętu, a przepisy krajowe mogą różnić się w zakresie dopuszczalnego EIRP i emisji niepożądanych.

Poziom promieniowania modułu jest wielokrotnie niższy od progów bezpieczeństwa dla ogółu ludności.

---

## 10. Procedura uruchomienia

### Etap 1 — pojedynczy kanał

Zmontować **jedną** płytkę i doprowadzić do pełnej sprawności przed powieleniem.

1. **Zasilanie:** zmierzyć 5 V na module i 3,3 V na nóżce 8 układu U1.
2. **Odniesienie:** na nóżce 7 powinno być 1,65 V.
3. **Punkt pracy:** na nóżce 1 w spoczynku 1,2–2,1 V. Wartość 0 V lub 3,3 V oznacza nasycenie — sprawdzić odniesienie na nóżce 3.
4. **Test ruchu:** wyjście przez kondensator 1 µF do wejścia mikrofonowego komputera, program Audacity. Ruch ręki 30 cm przed modułem musi dać wyraźny przebieg kilkudziesięciu Hz.
5. **Test pasma:** zbliżyć obracające się śmigło lub wiatraczek na pół metra. W analizie widmowej muszą pojawić się prążki w zakresie 2–4 kHz. Ich brak oznacza, że C2 ma złą wartość.

### Etap 2 — diagnostyka

| Objaw | Prawdopodobna przyczyna |
|---|---|
| Brak jakiegokolwiek sygnału | Zmierzyć napięcie stałe na pinie IF (oczekiwane ±0,01–0,2 V). Zero oznacza, że moduł nie oscyluje. |
| Brum 50/100 Hz dominujący | Zasilić z powerbanku, skrócić ścieżkę IF–C1 |
| Sygnał przycięty | Zmniejszyć R3 do 470 kΩ |
| Szum zmienia się przy dotknięciu przewodów | Błąd w topologii masy — zweryfikować gwiazdę |

### Etap 3 — powielenie i integracja

Dopiero po zamknięciu etapów 1–2: wykonać pozostałe siedem płytek, zmontować pierścień z przegrodami, podłączyć multiplekser i ESP32.

**Test przesłuchu:** zasłonić siedem modułów materiałem pochłaniającym, sprawdzić, czy ósmy widzi ruch tylko w swoim sektorze.

**Test sumarycznego szumu:** odłączyć siedem modułów, zostawić jeden. Wyraźny spadek szumu oznacza, że prądy zasilania pozostałych modułów przenikają do toru sygnału.

---

## 11. Ograniczenia i możliwe rozwinięcia

### Ograniczenia obecnej konstrukcji

- brak pomiaru odległości (fala ciągła bez modulacji)
- brak rozróżnienia kierunku ruchu (brak kanałów I/Q)
- zasięg na drona ograniczony do kilku metrów przez małą skuteczną powierzchnię odbicia
- rozdzielczość kątowa ograniczona szerokością wiązki modułu

### Kierunki rozwoju

**Kanały I/Q** — moduł CDM324 lub HB100 z zewnętrznym mieszaczem kwadraturowym dałby znak prędkości, umożliwiając rozróżnienie zbliżania od oddalania i barwne kodowanie na wyświetlaczu.

**Modulacja FMCW** — dodanie modulacji częstotliwości pozwoliłoby mierzyć odległość. Wymaga oscylatora przestrajanego napięciem, czego HB100 nie posiada (nota stwierdza wprost brak elementów regulacyjnych).

**Moduły mmWave** — układy pokroju IWR6843 dostarczają odległość, prędkość i kąt jednocześnie, z gotowym przetwarzaniem sprzętowym. Jest to droga do realnego wykrywania dronów na dystansie 50–150 m.

**Klasyfikacja automatyczna** — obecnie rozróżnienie klas obiektów wymaga oceny wzrokowej spektrogramu. Prosty klasyfikator oparty na cechach widma (szerokość, symetria, obecność wstęg bocznych) mógłby to zautomatyzować.

---

## 12. Źródła

- ST Electronics / AgilSense, *MSAN-001: X-Band Microwave Motion Sensor Module Application Note*, wersja 1.02 — dokumentacja modułu HB100, wzór Dopplera, układ wzmacniacza odniesienia, dane o RSS i zakłóceniach.
- Microchip Technology, *MCP6001/1R/1U/2/4: 1 MHz, Low-Power Op Amp*, DS20001733 — rozkład nóżek i parametry układu MCP6002.
- EN 300 440 — norma zharmonizowana dla urządzeń bliskiego zasięgu w paśmie 1–40 GHz.

---

*Dokumentacja opisuje stan projektu na etapie zweryfikowanego schematu elektrycznego jednego kanału. Schemat w formacie KiCad stanowi załącznik.*