# 📡 ESP32 Sub-GHz (433.92 MHz) ASK/OOK Signal Sniffer via TI CC1101

Profesjonalny, amatorski skaner i sniffer kodów radiowych ASK/OOK z pilotów zdalnego sterowania (bramy wjazdowe/garażowe, rolety okienne, systemy inteligentnego domu), pracujący na częstotliwości **433.92 MHz**. Projekt oparty jest na popularnym mikrokontrolerze **ESP32** oraz zaawansowanym układzie radiowym **Texas Instruments CC1101**, zarządzany w całości przez system **ESPHome**.

Projekt sekcji krótkofalarskiej stacji amatorskiej klubu **SP3PM**.

---

## 🛠️ Rozwiązane Problemy Inżynierskie (Hardware Fixes)

Podczas implementacji w czystym środowisku C++ (Arduino IDE/PlatformIO) projekt napotykał na twarde bariery sprzętowe, które zostały skutecznie zażegnane po migracji do architektury ESPHome:

*   **Izolacja magistrali SPI (Boot-Order Fix):** Projekt definitywnie rozwiązuje krytyczny błąd braku komunikacji z rejestrami radiowymi (`BŁĄD MAGISTRALI! Kod: -104` / `Failed to verify CC1101`). Poprzez wymuszenie odpowiedniego priorytetu startu (`priority: 100.0`), szyna SPI jest rezerwowana i stabilizowana *po* pełnym zbootowaniu systemu, co zapobiega konfliktom z asynchronicznym uruchamianiem stosu sieciowego Wi-Fi w tle.
*   **Optymalizacja czułości (Filter Bandwidth):** Zwiększenie szerokości pasma filtru odbiornika do **325 kHz** pozwala na bezbłędne dekodowanie sygnałów z pilotów starszego lub tańszego typu, których rezonatory kwarcowe fabrycznie lekko odpływają od częstotliwości środkowej.
*   **Passive RX Mode (Pasywny nasłuch):** Konfiguracja pinu danych `GDO0` jako czystego, odizolowanego wejścia (`INPUT`) eliminuje ryzyko elektrycznego blokowania linii sygnałowej przez mikrokontroler (częsty problem w innych bibliotekach powodujący permanentną głuchotę odbiornika).

---

## 🔌 Schemat Połączeń (Pinout dla ESP32 DevKitC)

Połączenie między mikrokontrolerem a modułem CC1101 realizowane jest za pomocą sprzętowego interfejsu magistrali SPI:

| CC1101 PIN | ESP32 DevKitC PIN | Rola w układzie |
| :--- | :--- | :--- |
| **VCC** | 3V3 | Zasilanie układu radiowego (3.3V) |
| **GND** | GND | Masa wspólna układu |
| **CSN / CS** | GPIO 5 | Chip Select (Wybór układu SPI) |
| **SCK** | GPIO 18 | Serial Clock (Zegar magistrali SPI) |
| **MISO / SO** | GPIO 19 | Master In Slave Out (Dane z CC1101 do ESP32) |
| **MOSI / SI** | GPIO 23 | Master Out Slave In (Dane z ESP32 do CC1101) |
| **GDO0** | GPIO 4 | Wyjście surowego demodulatora (Przerwanie asynchroniczne) |

*Uwaga: Linie RST oraz GDO2 w module CC1101 zostały programowo skonfigurowane jako wyłączone w celu redukcji zbędnego okablowania.*

---

## 💻 Kompletny Kod Konfiguracyjny (`esp32-bramka-rf.yaml`)

Poniższy kod należy wkleić bezpośrednio do swojego projektu w panelu ESPHome:

```yaml
esphome:
  name: esp32-bramka-rf
  friendly_name: ESP32 Skaner RF 433MHz
  
  # SPRZĘTOWY FIX: Bezpieczny moment startu radia po pełnej inicjalizacji szyny SPI
  on_boot:
    priority: 100.0
    then:
      - cc1101.begin_rx: cc1101_radio

esp32:
  board: esp32dev
  framework:
    type: arduino

wifi:
  ssid: "ssid_wifi"
  password: "passwd"
  
  manual_ip:
    static_ip: 192.168.1.199
    gateway: 192.168.1.1     
    subnet: 255.255.255.0    

# Autonomiczny serwer WWW do podglądu wyników pod adresem 192.168.1.199
web_server:
  port: 80
  version: 2

# Filtrowanie logów zapewniające czystość konsoli diagnostycznej
logger:
  level: DEBUG                
  logs:
    web_server: INFO
    web_server_idf: INFO
    component: INFO
    scheduler: INFO
    app: INFO
    remote_receiver: INFO     
    remote.raw: INFO
    remote.rc_switch: DEBUG   

# Konfiguracja sprzętowa szyny SPI
spi:
  clk_pin: GPIO18
  mosi_pin: GPIO23
  miso_pin: GPIO19

# Parametryzacja transceivera TI CC1101 dla modulacji ASK/OOK
cc1101:
  id: cc1101_radio
  cs_pin: GPIO5
  frequency: 433.92MHz
  modulation_type: ASK/OOK
  symbol_rate: 5000
  filter_bandwidth: 325kHz  

# Konfiguracja asynchronicznego odbiornika sygnałów z pilota (Passive RX Mode)
remote_receiver:
  id: rf_receiver
  pin:
    number: GPIO4   
  tolerance: 60%    
  filter: 50us      
  idle: 10ms        
  buffer_size: 8192B 
  
  dump:
    - rc_switch
    
  # Konwersja odebranego kodu na format dziesiętny i wysłanie na stronę WWW
  on_rc_switch:
    then:
      - lambda: |-
          std::string wynik = "Kod: " + std::to_string(x.code) + " [Protokol: " + std::to_string(x.protocol) + "]";
          id(ostatni_pilot).publish_state(wynik);

# Definicja kafelka wyświetlającego odkodowane dane bezpośrednio w przeglądarce
text_sensor:
  - platform: template
    name: "Ostatnio przechwycony pilot"
    id: ostatni_pilot
    icon: "mdi:remote"
```

---

## 🚀 Instrukcja Uruchomienia

1.  Utwórz nowe urządzenie w swoim panelu **ESPHome Dashboard**.
2.  Skopiuj powyższą konfigurację YAML i dostosuj sekcję `wifi` (wpisując parametry swojej sieci lokalnej oraz pożądany, wolny statyczny adres IP).
3.  Skompiluj projekt i wgraj oprogramowanie na płytkę ESP32 za pomocą kabla USB lub bezprzewodowo (OTA).
4.  Otwórz dowolną przeglądarkę internetową i wpisz przypisany adres IP: `192.168.1.199`. 
5.  Skieruj pilota w stronę stacji i kliknij przycisk. Odkodowane dane (czysty kod dziesiętny oraz numer protokołu) pojawią się natychmiast w kafelku **"Ostatnio przechwycony pilot"**.

---

## 📄 Licencja (MIT License)

Copyright (c) 2026 **SP3PM Amateur Radio Club**

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---
**73 dla wszystkich stacji i powodzenia w nasłuchach!**

