# WDGWMarauderV8

Firmware wardrivingowy dla **Marauder V8 (ESP32-C5)** — z opcjonalnym klastrem
pięciu nodów **Seeed Studio XIAO ESP32-C5**, które zbierają równolegle i raportują
do Maraudera po ESP-NOW.

Zbiera sieci Wi-Fi 2.4 + 5 GHz oraz urządzenia BLE z pozycją GPS, zapisuje na kartę
microSD w formacie **WigleWifi-1.6** i wysyła na **[wdgwars.pl](https://wdgwars.pl)**.

**🇬🇧 English version: [README.md](README.md)**

To **nie jest** stock ESP32Marauder. To osobny firmware pisany pod tę płytkę.

---

## Co potrafi

**Zbieranie**
- Wi-Fi w trybie promiscuous na 2.4 i 5 GHz, z adaptacyjnym czasem nasłuchu na kanał
- BLE (aktywny skan), wykrywanie **kamer Flock** po Wi-Fi i po BLE
- Wykrywanie trackerów **Find My / AirTag** i wymuszanie dzwonka (antystalking)
- Zbieranie **bez zasięgu GPS** (metro, garaż, galeria) — trafienia czekają
  zaparkowane i dostają pozycję przez interpolację, gdy fix wróci

**Zapis i wysyłka**
- microSD, format WigleWifi-1.6 z kolumną częstotliwości
- Bufor z podwójnym przełączaniem; zapis na kartę **poza blokadą**, więc zbieranie
  nie gubi ramek podczas zapisu
- Upload na wdgwars.pl po TLS z przypiętym certyfikatem; wysłane pliki oznaczone
- Przeglądanie logów na urządzeniu, wysyłka lub kasowanie pojedynczego pliku

**Klaster (opcjonalny)**
- Marauder przestaje skanować i **dowodzi** flotą do 18 nodów
- Kanały dzielone według **zmierzonego ruchu**, nie po równo — gęsty kanał dostaje
  node na wyłączność i stoi na nim nieruchomo, słysząc każdy bikon
- Jeden node pełni rolę **BLE + Flock** (C5 ma jedno radio — albo Wi-Fi, albo BLE)
- Rdzeń stempluje trafienia **pozycją z chwili, gdy node je usłyszał**, a nie z chwili
  odbioru — bez tego przy 50 km/h błąd sięgałby 300 m
- **Aktualizacja nodów po eterze** z karty Maraudera, bez kabla
- **Parowanie z akceptacją na ekranie** — każdy rig ma własny, losowy klucz

---

## Sprzęt

| Rola | Płytka | Uwagi |
|---|---|---|
| Rdzeń / samodzielny wardriver | **Marauder V8 z ESP32-C5** | TFT + touch, microSD, GPS, MAX17048 |
| Node klastra (opcjonalnie) | **Seeed Studio XIAO ESP32-C5** | 8 MB flash, PSRAM; bez wyświetlacza i karty |

Klaster jest opcjonalny — Marauder działa samodzielnie i bez ani jednego noda.

Zasilanie: pięć nodów w ciągłym odbiorze to **~600–750 mA** plus Marauder. Hub USB bez
własnego zasilacza powoduje resety, a node resetujący się od podnapięcia wygląda na
ekranie **identycznie** jak node, który zgubił rdzeń.

---

## Instalacja — flasher w przeglądarce (najprościej)

Nie wymaga niczego poza przeglądarką opartą o Chrome.

1. Wejdź na **[esptool.spacehuhn.com](https://esptool.spacehuhn.com/)**
2. Podłącz płytkę kablem USB i kliknij **Connect**, wybierz port
3. Dodaj **jeden** plik z offsetem **`0x0`**:

| Urządzenie | Plik |
|---|---|
| Marauder V8 | `firmware/marauder-v8-c5/WDGWMarauderV8-marauder-merged.bin` |
| XIAO ESP32-C5 (node) | `firmware/node-xiao-c5/WDGWMarauderV8-node-merged.bin` |

4. Kliknij **Program** i poczekaj do końca
5. Odłącz i podłącz zasilanie ponownie

Obrazy scalone zawierają bootloader, tablicę partycji i aplikację — dlatego offset to
`0x0` i nie ma nic więcej do podawania.

> **Uwaga przy ponownym wgrywaniu noda kablem.** Node, który choć raz wziął aktualizację
> po eterze, startuje z **drugiej partycji**. Obraz scalony zapisuje pierwszą, więc bez
> wyczyszczenia flasha układ dalej uruchomi **stary** firmware, mimo że flashowanie
> zgłosi sukces. W flasherze webowym zaznacz **Erase device** przed programowaniem.

---

## Instalacja — `esptool` z linii poleceń

Wymaga `esptool` w wersji **5.x**. Wersja 4.8.1 z Homebrew **zawiesza się na ESP32-C5** —
użyj tej z pakietu Arduino ESP32 albo `pip install --upgrade esptool`.

**Marauder V8:**

```bash
esptool --chip esp32c5 --port /dev/ttyUSB0 --baud 921600 write-flash -z \
  0x2000  firmware/marauder-v8-c5/bootloader.bin \
  0x8000  firmware/marauder-v8-c5/partitions.bin \
  0x10000 firmware/marauder-v8-c5/app.bin
```

**Node XIAO ESP32-C5:**

```bash
esptool --chip esp32c5 --port /dev/ttyACM0 --baud 921600 erase-flash
esptool --chip esp32c5 --port /dev/ttyACM0 --baud 921600 write-flash -z \
  0x2000  firmware/node-xiao-c5/bootloader.bin \
  0x8000  firmware/node-xiao-c5/partitions.bin \
  0x10000 firmware/node-xiao-c5/app.bin
```

`erase-flash` przy nodzie jest istotne — patrz uwaga o partycjach wyżej.

Porty: na Linuksie zwykle `/dev/ttyUSB0` (Marauder, mostek USB-UART) i `/dev/ttyACM0`
(XIAO, natywne USB). Na macOS `/dev/cu.usbserial-*` i `/dev/cu.usbmodem*`.
XIAO **zawsze** enumeruje się pod tą samą nazwą, więc przy kilku sztukach po nazwie
portu ich nie odróżnisz — sprawdź `esptool ... read-mac`.

**Weryfikacja noda po wgraniu** — po resecie na konsoli (115200) pojawia się:

```
[node] WDGNODEFW:0004 | firmware v4
```

Jeśli tej linii nie ma, node uruchomił stary obraz z drugiej partycji.

---

## Konfiguracja

Skopiuj **`wdgwars.cfg.sample`** do **katalogu głównego karty microSD** i zmień nazwę na
**`wdgwars.cfg`**. Plik zostaje na twojej karcie — firmware nigdy go nie wysyła ani nie
wypisuje na konsolę (hasła pokazuje tylko jako długość).

Minimum, żeby ruszyć:

```ini
ssid=NazwaTwojegoWiFi
pass=HasloDoWiFi
key=twoj_klucz_api_64_znaki_hex
```

Klucz API wygenerujesz na wdgwars.pl w swoim profilu — to dokładnie 64 znaki
szesnastkowe. Wiąże wysłane sieci z twoim kontem, więc użyj własnego.

Bez pliku urządzenie **zbiera i zapisuje normalnie** — nie działa tylko wysyłka.

---

## Klaster — jak uruchomić

1. Wgraj firmware **noda** na każdą XIAO
2. Na Marauderze: **MENU → CLUSTER**
3. Nody zgłoszą się same. Pojawi się pasek:

```
5 node(s) want to join:
1CD4  0178  1BBC  3924  8020
[ ACCEPT ALL ]        * = re-adopt
```

4. Sprawdź, czy końcówki adresów zgadzają się z twoimi sztukami, i naciśnij **ACCEPT ALL**

Marauder losuje wtedy **własny klucz** sprzętowym generatorem i rozdaje go przyjętym
nodom. Od tej chwili flota jest twoja.

**BACK** opuszcza ekran, ale flota pracuje dalej. **STOP** kończy sesję i zamyka plik
logu, dzięki czemu widać go w SYNC.

Gwiazdka przy adresie znaczy „mam klucz, ale nic mi nie odpowiada" — tak node prosi
o ponowne przyjęcie, gdy rdzeń zmienił klucz. Nie trzeba kabla.

### Firmware nodów: pierwsze wgranie, potem aktualizacje

**Za pierwszym razem każdy node wymaga kabla.** Nie da się tego obejść: fabryczna XIAO
nie ma firmware'u, który mógłby przyjąć aktualizację. Wgraj każdą tak, jak opisano
w sekcji *Instalacja* — flasherem w przeglądarce albo `esptool`, obojętnie.

**Potem już nigdy.** Aktualizacje idą po eterze z karty Maraudera.

#### Jak wrzucić obraz na kartę

Pobierz **`node_fw.bin`** z
[Releases](https://github.com/LOCOSP/WDGWMarauderV8/releases) i skopiuj do
**katalogu głównego karty microSD**:

```
/node_fw.bin          ← tutaj
/wdgwars.cfg
/wdgw/                ← tu leżą logi, obrazu tu NIE ma
```

Nazwa musi brzmieć dokładnie `node_fw.bin` i plik musi leżeć w katalogu głównym,
nie w `wdgw/`. To jedyne miejsce, w które firmware zagląda.

Marauder sprawdza kartę **przy każdym uruchomieniu**. Jeśli znajdzie obraz, na ekranie
startowym pojawia się linia `node fw v4 on card`, a ekran CLUSTER dopisuje uwagę, gdy
któryś node jest w tyle. Nie musisz tego szukać.

> Nie chcesz wyjmować karty? Jest droga przez kabel: komenda `nodefw <bajty>` na konsoli
> (115200), potem wysyłasz surowy plik. Trwa jakieś dwie i pół minuty na megabajt
> i **blokuje panel** na ten czas — ekran o tym uprzedza. Wyjęcie karty jest szybsze.

#### Przebieg aktualizacji

1. **MENU → CLUSTER** — poczekaj, aż nody się zameldują, i wyjdź przez **BACK**
   (nie czerwonym STOP, bo ten rozwiązuje flotę)
2. **MENU → NODE FW → CHECK** — pokaże wersję obrazu i wersję każdego noda
3. **UPDATE ALL**

Co zobaczysz, po kolei:

| Etap | Co znaczy |
|---|---|
| `asking nodes who needs it...` | node słucha kanału sterującego tylko przez chwilę, więc rdzeń czeka, aż każdy się odezwie |
| `sending 42%` | obraz idzie rozgłoszeniem — pięć nodów kosztuje tyle co jeden |
| `filling gaps` | rozgłoszenia są bez potwierdzeń, więc nody proszą teraz o kawałki, których nie dostały |
| `updated` przy nodzie | zapisany i zatwierdzony |
| `all 5 now run v4` | **zweryfikowane po restarcie** — wersja, którą same zgłosiły |

**UPDATE ALL jest wyszarzony**, gdy nie ma czego robić: brak obrazu, brak nodów, transfer
w toku albo wszystkie już aktualne. Naciśnij mimo to, a napisze który z tych powodów.

Możesz wyjść z ekranu w trakcie — transfer leci dalej.

#### Dlaczego to nie zamuruje noda

Node składa obraz w PSRAM i sprawdza **SHA-256 całości**, zanim w ogóle dotknie pamięci —
niepełny albo uszkodzony transfer po prostu się nie udaje, a node dalej chodzi na tym, co
miał. Po zatwierdzeniu nowy firmware startuje **na probacji**: jeśli w dwie minuty nie
dogada się z rdzeniem, przestawia partycję rozruchową z powrotem i wraca do poprzedniej
wersji.

To pokrywa przypadek, którego suma kontrolna nie wyłapie: obraz spójny, ale zepsuty.

---

## Ekrany

Ekran główny zbiera. Reszta siedzi pod **MENU** — dwa rzędy opcji, a na dole
**INFO | SET | BACK**.

### Trzy ekrany wymagają wcześniejszego uruchomienia innego

Każdy się na tym raz potyka, więc piszę to przed wszystkim innym. Trzy ekrany pokazują
**zamrożoną listę zebraną przez inny ekran** i celowo nie zbierają jej same. Wejście na
zimno wygląda jak awaria — a to nie awaria, tylko czekanie.

```
   BT SCAN      →  BACK  →   FOX HUNT
   AIRTAG SCAN  →  BACK  →   RING TAG
   CLUSTER      →  BACK  →   NODE FW
```

| Chcesz | Najpierw | Potem |
|---|---|---|
| Namierzyć urządzenie Bluetooth | **BT SCAN** — poczekaj, aż lista się zapełni | **BACK**, potem **FOX HUNT**, wybierz cel |
| Zadzwonić trackerem | **AIRTAG SCAN** — daj mu znaleźć tagi | **BACK**, potem **RING TAG**, wybierz tag |
| Zaktualizować nody | **CLUSTER** — poczekaj, aż flota się zamelduje | **BACK**, potem **NODE FW → CHECK → UPDATE ALL** |

**Dlaczego tak.** Skanowanie i wybieranie nie mogą dziać się naraz: lista przestawiałaby
się pod palcem, gdy sygnały pojawiają się i znikają, a na tym układzie skan działający
w tle rywalizuje o jedno radio z tym, co robisz. Zamrożenie listy daje stabilny wybór.

**Wychodź przez BACK, nie STOP.** Na ekranie CLUSTER czerwony **STOP** kończy sesję —
rozwiązuje flotę i zamyka plik logu. **BACK** tylko opuszcza ekran, a wszystko pracuje
dalej. I o to chodzi przed wejściem w NODE FW: ten ekran potrzebuje działających nodów,
żeby zobaczyć ich wersje.

Jeśli ekran wyboru pokaże pustą listę, napisze, który ekran uruchomić najpierw — zamiast
tkwić w nieskończoność na „scanning…".

### SCAN — wardriving

Zbiera Wi-Fi i BLE z pozycją GPS i dopisuje wiersze WigleWifi-1.6 na kartę. Zakres
ustawia się komendą `sub wifi|ble|both`; **both** jest domyślne i na tym układzie
najstabilniejsze.

Pod ziemią, w garażu, w galerii — wszędzie tam, gdzie nie widać nieba — trafienia są
**parkowane, a nie wyrzucane**, i dostają pozycję przez interpolację, gdy fix wróci.
Wiersz z 0,0 portal i tak odrzuci, a przy okazji MAC wpadłby do tablicy duplikatów
i ta sieć nie zostałaby zapisana **już nigdy** po wyjeździe na powierzchnię.

### SYNC — logi i wysyłka

Lista logów z karty, najnowsze na górze, wysłane oznaczone. Tapnięcie pliku daje
**UPLOAD** i **DELETE** — sprzątanie bez wyjmowania karty. Plik aktualnie zapisywany
jest opisany jako nagrywany i nie da się go wysłać.

### BT SCAN — urządzenia Bluetooth

Aktywny skan BLE z listą na żywo: adres, nazwa jeśli rozgłaszana, siła sygnału.
To z tej listy wybiera potem **FOX HUNT**.

### FOX HUNT — znajdź konkretne urządzenie

Namierzanie po sile sygnału. Wybierasz urządzenie i chodzisz — odczyt rośnie, gdy się
zbliżasz. Przydaje się do zlokalizowania beacona, zgubionego taga albo ustalenia, które
urządzenie w pomieszczeniu jest które.

> **Najpierw skan, potem wybór.** FOX HUNT **celowo nie skanuje** — pokazuje listę, którą
> zebrał **BT SCAN**. Kolejność: **BT SCAN → BACK → FOX HUNT**. Wejście bez wcześniejszego
> skanu daje pustą listę i to jest podpowiedź, nie usterka.

### AIRTAG SCAN — trackery

Wykrywa trackery klasy Find My / AirTag w otoczeniu. Filtrowanie jest tu sednem: sam
producent `0x004C` pasuje do **każdego** uczestnika sieci Find My, łącznie z telefonami
i Makami, co zasypuje listę. Firmware filtruje po kategorii w bajcie statusu oraz po
UUID usług — Apple `0xFD44`, Samsung SmartTag `0xFD5A`, DULT `0xFCB2`, Google, Tile,
Chipolo — i odrzuca wszystko poniżej −85 dBm, bo w antystalkingu liczy się to, co jedzie
**z tobą**.

Tagi odłączone od właściciela mają znacznik **SEP**. To on decyduje, czy dzwonienie
w ogóle zadziała.

### RING TAG — zmuś tracker do dzwonienia

Wybierasz tag z tego, co znalazł AIRTAG SCAN, i każesz mu zadzwonić — żeby zlokalizować
coś podrzuconego do torby albo do auta.

> Ta sama zasada: **AIRTAG SCAN → BACK → RING TAG**. Lista jest zamrożona ze skanu.

**Co zadzwoni, a co nie — to konstrukcja Apple, nie ograniczenie tego firmware'u:**

- **Dzwonek właściciela** (to, co robi aplikacja Find My) jest chroniony sekretem
  z parowania. Z ESP32 niewykonalny, przez nikogo.
- **Dzwonek nie-właściciela** (ten antystalkingowy) działa **tylko wtedy, gdy tag jest
  odłączony od wszystkich urządzeń Apple swojego właściciela**.

Gdy właściciel jest w pobliżu, zapis **udaje się na poziomie protokołu**, a tag i tak
milczy i odpowiada `0xFFFF`. To najbardziej mylący objaw w całej tej funkcji: wszystko
wygląda na sukces i nic nie piszczy.

Żeby zadzwonić własnym, sparowanym tagiem: wyłącz Bluetooth w iPhonie i Macu, odczekaj
15–30 minut, aż pokaże się jako **SEP**, i dopiero wtedy dzwoń. Etui AirPods dzwoni bez
oporu i podczas testów bywa fałszywym tropem.

### CLUSTER i NODE FW

Flota i jej aktualizacje — opisane wyżej.

### SET i INFO

Ustawienia i informacje o urządzeniu: wersja firmware'u, stan GPS, bateria, karta,
pamięć.

---

## Aktualizacja sparowanego zestawu

**Wgranie obrazu scalonego kasuje parowanie.** Ten plik obejmuje pamięć od adresu `0x0`
razem z partycją NVS i wypełnia ją pustymi bajtami — czyli zamazuje klucz floty, który
Marauder wylosował przy przyjmowaniu nodów. Nody zachowują swój, nic się nie
uwierzytelnia, a ekran klastra świeci pustką. Wygląda dokładnie jak zepsute wydanie.

Przy **pierwszej** instalacji to nie przeszkadza, a nawet jest pożądane. Przy
**aktualizacji** wgraj samą aplikację i wszystko przeżyje:

```bash
esptool --chip esp32c5 --port /dev/ttyUSB0 --baud 921600 write-flash -z \
  0x10000 firmware/marauder-v8-c5/app.bin
```

`app.bin` leży powyżej NVS, więc klucz floty, konfiguracja i parowania zostają nietknięte.

**Jeśli już wgrałeś obraz scalony i nody zniknęły:** nic nie przepadło i kabel nie jest
potrzebny. Każdy node w ciągu mniej więcej minuty zauważy, że nic z tego, co słyszy, nie
jest poprawnie podpisane, i zacznie prosić o ponowne przyjęcie. Wejdź w **CLUSTER**,
poczekaj na listę i naciśnij **ACCEPT ALL**. Zestaw wylosuje nowy klucz i rozda go od nowa.

Ekran klastra mówi to teraz wprost, gdy nie ma klucza, zamiast zostawiać pustą listę.

---

## Bezpieczeństwo — co chroni, a co nie

Piszę wprost, bo publikowane binarki da się przeszukać.

**Klucz floty jest losowany na twoim urządzeniu** i nie ma go w tym repozytorium ani
w żadnym obrazie. Podpisuje każdą ramkę ESP-NOW — każdy przydział, każdy rekord, każdy
kawałek firmware'u.

**Sprostowanie (2026-08-06):** wcześniejsza wersja tego pliku twierdziła, że ruch
sterujący jest dodatkowo **szyfrowany AES-CCM**. Nie jest. Klucz trafia do ESP-NOW jako
PMK, ale szyfrowanie per-peer jest świadomie wyłączone: ESP-NOW szyfruje najwyżej
**sześciu** peerów, mieszcząc dwudziestu — włączenie go ograniczyłoby flotę do sześciu
nodów. Wygrała wielkość floty. Ruch jest więc **podpisany, ale jawny**, a zdanie zostało
sprostowane, nie po cichu usunięte.

**Co z tego wynika:** nikt w zasięgu nie wgra ci firmware'u na nody ani nie podstawi
fałszywych sieci do logu — a to była najpoważniejsza konsekwencja wcześniejszego,
wkompilowanego sekretu.

**Czego to nie daje:**
- Rozgłoszenia (kawałki aktualizacji, zgłoszenia nodów) są **podpisane, ale jawne** —
  ESP-NOW nie potrafi szyfrować rozgłoszeń
- Sekret używany do **samego dołączania** jest publiczny i widoczny w obrazie
  (`wdgwars-fleet-2026`). Przenosi wyłącznie „czy mogę dołączyć?" i nie daje niczego
  poza prawem do pojawienia się na twoim ekranie z prośbą o akceptację
- Klucz floty przelatuje w eterze **jawnie w jednej ramce**, w momencie gdy naciskasz
  ACCEPT. Okno trwa ułamek sekundy i otwiera je twoja świadoma decyzja
- Podpis to suma kontrolna z kluczem, **nie podpis kryptograficzny**. Chroni przed
  podszyciem i przypadkowym pomieszaniem flot, nie przed przeciwnikiem z budżetem

**Twoje dane:** `wdgwars.cfg` z kluczem API i hasłami zostaje na karcie. Nie trafia do
logów, nie jest wysyłany i nie jest wypisywany na konsolę.

---

## Format danych

WigleWifi-1.6 — ten, którego oczekuje wdgwars.pl (**nie** 1.4, różnica to kolumna
częstotliwości):

```
MAC,SSID,AuthMode,FirstSeen,Channel,Frequency,RSSI,Lat,Lon,AltitudeMeters,AccuracyMeters,Type
54:DB:A2:1A:D7:DC,HALNy-2.4G,[WPA2-PSK-CCMP][ESS],2026-07-30 02:41:43,1,2412,-88,50.1234,17.5678,123,8.0,WIFI
82:19:4F:FE:81:43,,[BLE],2026-07-30 02:41:43,0,0,-95,50.1234,17.5678,123,8.0,BLE
```

---

## Gdy coś nie działa

| Objaw | Najpierw sprawdź |
|---|---|
| Node „ucichł" | zasilanie. Podnapięcie wygląda tak samo jak utrata rdzenia |
| Node ma starą wersję mimo wgrania kablem | uruchomił drugą partycję — wyczyść flash i wgraj ponownie |
| Liczniki rosną, a karta pusta | czy sesja logu jest otwarta (na ekranie plik jest nazwany) |
| Upload odrzucony | `202` i `409` z serwera to **sukces**, nie błąd |
| Brak zapisu po dłuższym czasie | karta — komenda `sdtest` na konsoli |

Konsola szeregowa (115200) ma pełen zestaw komend: `status`, `logstats`, `dumplog`,
`cluster`, `pair`, `nodeota`, `scanstat`, `sdtest`, `help`.

Zasada, która oszczędziła najwięcej czasu przy tym projekcie: **potwierdzenie zapisu to
nie potwierdzenie działania.** Sprawdzaj skutek, nie deklarację — po aktualizacji baner
wersji, po zapisie `logstats`, po wgraniu odczyt z układu.

---

## Wydawanie

Nazwy plików, format sum kontrolnych i znaczniki wersji są **umową** z flasherem
wdgwars.pl, który serwuje te binarki bezpośrednio. Zajrzyj do
**[RELEASING.md](RELEASING.md)**, zanim cokolwiek z tego zmienisz.

Sprawdzenie pobranego pliku jedną komendą, w katalogu z pobranymi plikami:

```bash
sha256sum -c SHA256SUMS.txt      # na macOS: shasum -a 256 -c
```

To dowodzi, że plik dotarł nieuszkodzony. **Nie** dowodzi, że wydanie jest godne zaufania —
suma leży obok binarki, więc obie pochodzą z tego samego miejsca.

## Podziękowania

Wzorce i inspiracje: [justcallmekoko/ESP32Marauder](https://github.com/justcallmekoko/ESP32Marauder),
[dark3d/ESP32DualBandWardriver](https://github.com/dark3d), C5Lab/projectZero,
[pr3y/Bruce](https://github.com/pr3y/Bruce).
Flasher webowy: [Spacehuhn Technologies](https://esptool.spacehuhn.com/).

---

## Licencja

**Wydawane wyłącznie jako binarki.** Kod źródłowy nie jest publikowany.

Copyright © 2026 LOCOSP / [wdgwars.pl](https://wdgwars.pl). Wszelkie prawa zastrzeżone.

Możesz pobierać i używać tych obrazów na własnym sprzęcie. Nie zezwala się na
redystrybucję, odsprzedaż, dekompilację ani na przedstawianie tej pracy jako własnej.

Firmware służy do **legalnego** wardrivingu — pasywnego nasłuchu publicznie rozgłaszanych
ramek — oraz do wykrywania trackerów w celach antystalkingowych. Odpowiedzialność za
zgodność z prawem miejsca użycia spoczywa na użytkowniku.
