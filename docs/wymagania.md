# WrenchLog – wymagania (POC)

## 1. Cel

WrenchLog to aplikacja mobilna, która odczytuje aktualny przebieg samochodu przez adapter OBD2
(Bluetooth LE) i na podstawie zapisanych interwałów przypomina o zbliżających się serwisach
(oleje, filtry itp.).

**Cel POC:** sprawdzić, czy da się niezawodnie odczytać przebieg z Pajero przez vLinker MS
w aplikacji .NET MAUI i policzyć na tej podstawie terminy serwisów.

## 2. Kontekst

| Element | Wartość |
|---|---|
| Pojazd | Mitsubishi Pajero IV 3.2 DI-D (4M41), 2015, skrzynia automatyczna |
| Adapter | Vgate vLinker MS – zgodny z ELM327, Bluetooth 5.2 / BLE, Android i iOS, usypia się po 30 min |
| Protokół | ISO 15765-4 CAN, 500 kb/s, 11-bit (`ATSP6`) |
| Technologia | .NET MAUI (C#) |
| Platformy | Android (POC), iOS (później – wymaga Maca lub buildu w chmurze) |
| Środowisko | Windows + Visual Studio, telefon z Androidem |
| Dane | Lokalnie na telefonie (SQLite) |

### 2.1. Odczyt przebiegu (zweryfikowany w Car Scanner)

| Parametr | Wartość |
|---|---|
| Nagłówek (adres ECU) | `79E` → `ATSH79E` |
| Polecenie | `21AD` (usługa 0x21 – ReadDataByLocalIdentifier, ID 0xAD) |
| Oczekiwana odpowiedź | `61 AD A B C …` |
| Formuła | `A + B·256 + C·65536` (3 bajty, little-endian) |
| Zakres | 0 – 1 000 000 km |

Sekwencja: `ATZ` → `ATE0` → `ATSP6` → `ATSH79E` → `21AD`.

Odczyt wymaga włączonego zapłonu.

### 2.2. Dlaczego BLE, a nie klasyczny Bluetooth (SPP)

iOS nie udostępnia aplikacjom klasycznego Bluetooth (SPP) dla urządzeń bez certyfikatu MFi.
BLE (CoreBluetooth) jest dostępne bez ograniczeń. vLinker MS obsługuje BLE, więc na obu
platformach używamy jednej ścieżki komunikacji – BLE.

## 3. Wymagania funkcjonalne POC

| ID | Wymaganie |
|---|---|
| P1 | Skanowanie BLE, wybór adaptera, połączenie, zapamiętanie wybranego adaptera. |
| P1a | Gdy adaptera nie ma w skanowaniu lub nie odpowiada – komunikat „włącz zapłon / adapter może być uśpiony”. |
| P1b | Automatyczne wykrywanie usługi i charakterystyk BLE (zapis + powiadomienia); wykryte UUID widoczne w konsoli diagnostycznej. |
| P2 | Odczyt przebiegu na żądanie (przycisk), parsowanie odpowiedzi wg formuły z 2.1. |
| P2a | Walidacja: wartość w zakresie 0–1 000 000 km i nie mniejsza od ostatniego zapisanego przebiegu. |
| P3 | Konsola diagnostyczna: log surowych komend i odpowiedzi, wysłanie własnej komendy. |
| P4 | Ręczne wpisanie przebiegu (wyjście awaryjne). |
| P5 | Lista pozycji serwisowych z szablonu (rozdz. 5); edycja interwałów; wpisanie ostatniego wykonania (km + data). |
| P6 | Status każdej pozycji: pozostało km / dni; OK / zbliża się / po terminie. |
| P7 | „Wykonano serwis” – zapis km + daty, reset licznika pozycji. |
| P8 | Lokalny zapis danych (SQLite). |
| P9 | Po odczycie przebiegu – alert w aplikacji, jeśli któraś pozycja zbliża się lub jest po terminie. |

### 3.1. Reguły statusu

- Interwał w km i/lub w miesiącach; liczy się to, co nastąpi **pierwsze**.
- **Zbliża się:** pozostało < 1 000 km lub < 30 dni (progi konfigurowalne w kodzie).
- **Po terminie:** przekroczony interwał km lub czasu.

## 4. Wymagania niefunkcjonalne

| ID | Wymaganie |
|---|---|
| N1 | Działanie offline. |
| N2 | Odczyt przebiegu < ~10 s od nawiązania połączenia. |
| N3 | Tylko odczyt – aplikacja nie wysyła do pojazdu żadnych poleceń zapisu ani kasowania. |
| N4 | Interfejs po polsku. |
| N5 | Logika (protokół ELM327, parsowanie, statusy) w bibliotece niezależnej od MAUI, pokryta testami jednostkowymi. |

## 5. Szablon serwisowy – Pajero IV 3.2 DI-D, automat

Typowe interwały – do weryfikacji z książką serwisową. Przy ciężkich warunkach (holowanie,
teren, brody) interwały zwykle skraca się o połowę.

| Pozycja | Interwał km | Interwał czasu |
|---|---|---|
| Olej silnikowy + filtr oleju | 15 000 | 12 mies. |
| Filtr powietrza | 30 000 | 24 mies. |
| Filtr paliwa | 30 000 | 24 mies. |
| Filtr kabinowy | 15 000 | 12 mies. |
| Olej w skrzyni automatycznej (ATF) | 60 000 | 48 mies. |
| Olej w reduktorze (skrzynia rozdzielcza) | 45 000 | 36 mies. |
| Olej w moście przednim | 45 000 | 36 mies. |
| Olej w moście tylnym | 45 000 | 36 mies. |

## 6. Architektura (szkic)

- **WrenchLog.Core** – biblioteka .NET: protokół ELM327, parsowanie odpowiedzi, model danych,
  liczenie statusów serwisów.
- **WrenchLog.Core.Tests** – testy xUnit.
- **WrenchLog.App** – aplikacja .NET MAUI: ekrany, MVVM (CommunityToolkit.Mvvm),
  BLE (Plugin.BLE), SQLite.

## 7. Poza zakresem POC (kolejne etapy)

- Automatyczny odczyt przebiegu w tle.
- Powiadomienia systemowe (także wg daty).
- Prognoza terminu serwisu na podstawie średniego dziennego przebiegu.
- Historia kosztów, warsztaty, notatki.
- Wiele pojazdów i konfigurowalne profile odczytu przebiegu.
- Eksport / import danych, synchronizacja w chmurze.
- Wersja na iOS.

## 8. Otwarte kwestie

- Surowa odpowiedź adaptera na `21AD` wraz z przebiegiem z licznika – do danych testowych.
- UUID usługi i charakterystyk BLE vLinkera MS – do odczytania (nRF Connect lub konsola diagnostyczna).
