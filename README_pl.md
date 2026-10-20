# 🎵 RPi Zero Spotify Streamer ze sprzętowym interfejsem (pizero-spotify-tft-player)

[![en](https://img.shields.io/badge/lang-en-blue.svg)](README.md)
*Przeczytaj to po [angielsku](README.md)*

Bezobsługowy (headless) odtwarzacz sieciowy Spotify Connect oraz wyświetlacz metadanych zbudowany na bazie **Raspberry Pi Zero W** i odchudzonego systemu **DietPi OS**. Układ wykorzystuje zewnętrzny przetwornik **DAC I2S** do sprzętowego dekodowania dźwięku oraz ekran **TFT ST7789V 240x320 px (SPI)** sterowany autorskim skryptem w języku Python, który wyświetla okładki albumów i informacje o utworach w czasie rzeczywistym.

---

## ⚙️ 1. Architektura Sprzętowa

* **Komputer jednopłytkowy:** Raspberry Pi Zero W (architektura ARMv6)
* **Przetwornik Audio:** PCM5102A (lub kompatybilny moduł DAC I2S)
* **Wyświetlacz:** 2.0" TFT LCD, 240x320 px, kontroler ST7789V (magistrala SPI, złącze 8-pinowe: `GND`, `VCC`, `SCL`, `SDA`, `RST`, `DC`, `CS`, `BL`)
* **System operacyjny:** DietPi (środowisko Debian Bookworm / Python 3.13)

**Uwagi dotyczące podłączenia pinów (Pinout):**
* **DAC I2S** wykorzystuje piny `GPIO 18` (BCLK), `GPIO 19` (LRCLK) oraz `GPIO 21` (DOUT).
* **Ekran ST7789V** korzysta ze sprzętowego interfejsu SPI0 (`MOSI`/`SDA`, `SCLK`/`SCL`, `CE0`/`CS`) oraz pinów sterujących `GPIO 24` (`DC`) i `GPIO 25` (`RST`).
* **Podświetlenie (`BL`):** Podłączone bezpośrednio do linii 3.3V (stałe podświetlenie) lub pozostawione niepodłączone. Sterowanie programowe w kodzie zostało przekierowane na nieużywany tzw. "ślepy pin" (`GPIO 23`), aby uniknąć konfliktu sprzętowego z zegarem DAC na pinie `GPIO 18`.

---

## 🛠️ 2. Przygotowanie Systemu i Sieci

Przed instalacją pakietów skonfiguruj warstwy sprzętowe i sieciowe za pomocą narzędzia `dietpi-config`:

1. **Aktywacja interfejsów sprzętowych:**
   * Przejdź do **Advanced Options** -> włącz magistralę **SPI**.
   * Przejdź do **Audio Options** -> **Soundcard** -> wybierz profil **`rpi-dac`** (lub **`hifiberry-dac`**).
2. **Wyłączenie protokołu IPv6 (Eliminuje błąd odrzucenia połączenia / OS Error 111):**
   * Przejdź do **Network Options: Adapters** -> przełącz opcję **IPv6** na **`[Off]`**.
3. **Wyłączenie oszczędzania energii Wi-Fi (Eliminuje zrywanie muzyki / OS Error 104):**
   * Wyłącz usypianie karty sieciowej w bieżącej sesji:
     ```bash
     sudo iw dev wlan0 set power_save off
     iw dev wlan0 get power_save
     ```
   * Utrwal ustawienie po restarcie, edytując plik `/etc/rc.local`:
     ```bash
     sudo nano /etc/rc.local
     ```
     Dopisz poniższą komendę tuż nad linijką `exit 0`:
     ```bash
     iw dev wlan0 set power_save off
     ```
4. **Weryfikacja karty dźwiękowej ALSA:**
   * Zrestartuj malinę (`sudo reboot`) i sprawdź indeks przypisany karcie DAC:
     ```bash
     aplay -l
     ```
   * Zanotuj numer karty (np. `card 0` odpowiada adresowi sprzętowemu `hw:0`).

---

## 📦 3. Zależności i Środowisko Python

### Pakiety systemowe (APT)
Zainstaluj biblioteki systemowe wymagane przez moduł graficzny `Pillow`, nowoczesne sterowniki pinów GPIO (`lgpio`), demona rozgłaszania mDNS (`avahi-daemon`) oraz pakiet certyfikatów SSL:

```bash
sudo apt update
sudo apt install -y libxcb1 libxcb-render0 libxcb-shape0 libxcb-xfixes0 \
                    liblgpio1 liblgpio-dev avahi-daemon ca-certificates
sudo apt install --reinstall ca-certificates
```

### Wirtualne środowisko Python (PIP)
Utwórz izolowane środowisko uruchomieniowe. **Ważne:** Pakiet `cbor2` musi zostać ograniczony poniżej wersji `6.0.0`, aby pominąć wymóg kompilacji kodu w języku Rust, co kończy się błędem na jednordzeniowym procesorze ARMv6. Biblioteka `rpi-lgpio` zastępuje przestarzały moduł `RPi.GPIO`.

```bash
python3 -m venv ~/spotify_player_env
source ~/spotify_player_env/bin/activate

pip install "cbor2<6.0.0" spotipy luma.lcd Pillow requests spidev rpi-lgpio
```

---

## 🔊 4. Konfiguracja Odtwarzacza (`spotifyd`)

Z uwagi na porzucenie wsparcia dla architektury ARMv6 przez nowsze wydania pakietu `raspotify`, projekt wykorzystuje lekki silnik **`spotifyd`** dostępny w repozytorium DietPi.

1. **Instalacja `spotifyd`:**
   Uruchom `dietpi-software`, wyszukaj pakiet `spotifyd`, zaznacz go spacją i zainstaluj.
2. **Konfiguracja ZeroConf oraz buforowania audio:**
   Otwórz plik konfiguracyjny demona:
   ```bash
   sudo nano /mnt/dietpi_userdata/spotifyd/spotifyd.conf
   ```
   Wprowadź poniższe parametry (pomiń sekcje `username` oraz `password` – autoryzacja odbywa się bezdotykowo przez sieć Wi-Fi za pośrednictwem usługi Avahi oraz oficjalnej aplikacji Spotify na telefonie):
   ```ini
   [global]
   device_name = "Odtwarzacz"
   bitrate = 320
   backend = "alsa"
   device = "hw:0"
   volume_controller = "softvol"
   normalisation_pregain = -5
   volume_ramp_time = 500
   ```
3. **Czyszczenie pamięci podręcznej i uruchomienie usługi:**
   ```bash
   sudo systemctl stop spotifyd
   sudo rm -rf /mnt/dietpi_userdata/spotifyd/cache/
   sudo systemctl enable spotifyd
   sudo systemctl start spotifyd
   ```

---

## 🖥️ 5. Skrypt Wyświetlacza i Autoryzacja OAuth (`player.py`)

1. Pobierz plik `player.py` z tego repozytorium i umieść w katalogu `/root/`.
2. Otwórz plik `player.py` w edytprze tekstowym i podmień zmienne `CLIENT_ID` oraz `CLIENT_SECRET` na swoje klucze wygenerowane w panelu [Spotify Developer Dashboard](https://developer.spotify.com/dashboard). Pamiętaj, aby w ustawieniach aplikacji na stronie Spotify dodać adres zwrotny `http://127.0.0.1:8888/callback`.
3. Przeprowadź jednorazową autoryzację uruchamiająć skrypt ręcznie:
```
source ~/spotify_player_env/bin/activate
python3 ~/player.py
```
4. Skopiuj wygenerowany link do przeglądarki, zatwierdż uprawnienia. Kiedy przeglądarka zgłosi błąd połączenia (to normalne), skopiuj cały URL z paska i wklej z powrotem do konsoli maliny. Token zostanie trwale zapisany w `~/.spotify_cache`. Przerwij działanie skryptu (`Ctrl+C`).

---

## 🚀 6. Automatyzacja Autostartu (`systemd`)

Aby skrypt wyświetlacza uruchamiał się automatycznie w tle po włączeniu zasilania:

1. Utwórz plik nowej usługi systemowej:
   ```bash
   sudo nano /etc/systemd/system/spotify-display.service
   ```
2. Wklej poniższą konfigurację:
   ```ini
   [Unit]
   Description=Spotify TFT Display Service
   After=network.target spotifyd.service

   [Service]
   Type=simple
   User=root
   WorkingDirectory=/root
   ExecStart=/root/spotify_player_env/bin/python3 /root/player.py
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```
3. Przeładuj demona i aktywuj usługę:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable spotify-display.service
   sudo systemctl start spotify-display.service
   ```

---

## 🔧 7. Rozwiązywanie Problemów (Baza Wiedzy)

| Objaw / Komunikat błędu | Przyczyna techniczna | Zastosowane rozwiązanie |
| :--- | :--- | :--- |
| **Błąd instalacji `cbor2` w `pip`** | Nowsze wersje `cbor2` wymagają kompilatora Rust, który zawiesza się na architekturze ARMv6. | Wymuszenie starszej wersji pakietu: `pip install "cbor2<6.0.0"`. |
| **`ValueError: Channel must be an integer`** (`gpio_LIGHT=None`) | Rygorystyczna walidacja typów w `rpi-lgpio` odrzuca wartość `None` przy próbie wyłączenia podświetlenia przez `luma.lcd`. | Przypisanie wolnego, fizycznego pinu jako "ślepego wyjścia" (`gpio_LIGHT=23`), co zwalnia `GPIO 18` dla DACa. |
| **`Couldn't write token to cache at: .cache`** | Problem z relatywnymi ścieżkami zapisu w bibliotece `spotipy` podczas pracy z usługami systemowymi. | Jawne użycie klasy `CacheFileHandler` ze ścieżką bezwzględną `os.path.expanduser('~/.spotify_cache')`. |
| **Ukośnie pocięty obraz i szum z pikseli na dole ekranu** | Kontroler ST7789V domyślnie pracuje w orientacji poziomej (320x240); wysłanie klatki 240x320 powoduje zawijanie pamięci RAM (memory wrapping). | Inicjalizacja ekranu jako `width=320, height=240`, rysowanie na płótnie `240x320` i programowy obrót klatki przez `.transpose(Image.ROTATE_90)` (lub `ROTATE_270` dla odwrócenia góra-dół). |
| **`Failed to resolve Access Point` / `os error 111`** | `spotifyd` próbuje zestawiać sesję przez nieobsługiwany stos IPv6 lub korzysta z zablokowanych adresów w pamięci podręcznej. | Wyłączenie IPv6 w `dietpi-config`, reinstalacja `ca-certificates` oraz usunięcie katalogu `/mnt/dietpi_userdata/spotifyd/cache/`. |
| **`Bad credentials` w logach `spotifyd`** | Błędy parsowania znaków specjalnych w pliku konfiguracyjnym lub logowanie przez konta OAuth (Google/FB). | Usunięcie loginu i hasła z `spotifyd.conf` na rzecz rozgłaszania ZeroConf przez `avahi-daemon` (wymaga konta Spotify Premium). |
| **`Connection reset by peer (os error 104)`** | Układ Wi-Fi w RPi Zero W przechodzi w stan uśpienia między paczkami danych, gubiąc pakiety podtrzymujące sesję (heartbeat). | Trwałe wyłączenie zarządzania energią komendą `iw dev wlan0 set power_save off`. |
| **`Unable to load audio item: MercuryError`** | Chwilowe przepełnienie bufora lub przycięcie jednordzeniowego procesora przy przełączaniu utworów. | Dodanie parametrów `normalisation_pregain = -5` oraz `volume_ramp_time = 500` w `spotifyd.conf`. |

---

## 📄 Licencja i Autor

Projekt stworzony przez **Rafała**. Udostępniony na licencji MIT.