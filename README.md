# ESP32 433.92 MHz ASK/OOK Signal Sniffer via TI CC1101

Profesjonalny, amatorski skaner i sniffer kodów radiowych ASK/OOK z pilotów zdalnego sterowania (bramy wjazdowe, rolety okienne, systemy automatyki domowej), pracujący na częstotliwości **433.92 MHz**. Projekt oparty jest na mikrokontrolerze **ESP32** oraz zaawansowanym układzie radiowym **Texas Instruments CC1101**, zarządzany w całości przez system **ESPHome**.

Projekt sekcji krótkofalarskiej stacji amatorskiej klubu **SP3PM**.

## 🛠️ Rozwiązane Problemy Inżynierskie (Hardware Fixes)

*   **Izolacja magistrali SPI (Boot-Order Fix):** Projekt rozwiązuje powszechny błąd braku komunikacji z rejestrami radiowymi (`BŁĄD MAGISTRALI! Kod: -104` / `Failed to verify CC1101`). Poprzez wymuszenie priorytetu startu (`priority: 100.0`), szyna SPI jest rezerwowana i stabilizowana *po* pełnym zbootowaniu systemu, co zapobiega konfliktom z asynchronicznym uruchamianiem tła sieciowego Wi-Fi.
*   **Optymalizacja czułości (Filter Bandwidth):** Zwiększenie szerokości pasma filtru odbiornika do `325kHz` pozwala na bezbłędne dekodowanie sygnałów z pilotów starszego typu, które fabrycznie lekko odpływają od częstotliwości środkowej.
*   **Passive RX Mode (Pasywny nasłuch):** Konfiguracja pinu danych `GDO0` jako czystego, odizolowanego wejścia (`INPUT`) eliminuje ryzyko elektrycznego blokowania linii sygnałowej przez mikrokontroler, co w innych bibliotekach powoduje permanentną głuchotę odbiornika.

## 🔌 Schemat Połączeń (Pinout dla ESP32 DevKitC)

Połączenie realizowane jest za pomocą sprzętowego interfejsu magistrali SPI:

| CC1101 PIN | ESP32 DevKitC PIN | Rola w układzie |
| :--- | :--- | :--- |
| **VCC** | 3V3 | Zasilanie układu radiowego (3.3V) |
| **GND** | GND | Masa wspólna układu |
| **CSN / CS** | GPIO 5 | Chip Select (Wybór układu SPI) |
| **SCK** | GPIO 18 | Serial Clock (Zegar magistrali SPI) |
| **MISO / SO** | GPIO 19 | Master In Slave Out (Dane z CC1101 do ESP32) |
| **MOSI / SI** | GPIO 23 | Master Out Slave In (Dane z ESP32 do CC1101) |
| **GDO0** | GPIO 4 | Wyjście surowego demodulatora (Przerwanie asynchroniczne) |

*Uwaga: Linie RST oraz GDO2 w module CC1101 zostały programowo wyłączone w celu redukcji okablowania.*

## 💻 Instrukcja Uruchomienia

1. Utwórz nowe urządzenie w swoim panelu **ESPHome Dashboard**.
2. Skopiuj zawartość pliku `esp32-bramka-rf.yaml` z tego repozytorium.
3. Dostosuj sekcję `wifi` wpisując parametry swojej sieci lokalnej oraz pożądany, wolny statyczny adres IP.
4. Skompiluj projekt i wgraj oprogramowanie na płytkę ESP32.
5. Otwórz dowolną przeglądarkę internetową i wpisz przypisany adres IP (np. `192.168.1.199`). Odkodowane kody będą pojawiać się w czasie rzeczywistym w kafelku **"Ostatnio przechwycony pilot"**.

## 📄 Licencja

Projekt udostępniony na otwartej licencji **MIT**. Wolno kopiować, modyfikować i rozpowszechniać kod w celach amatorskich, naukowych oraz komercyjnych. 73 dla stacji!


esphome:
  name: esp32-bramka-rf
  friendly_name: ESP32 Skaner RF 433MHz
  
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

# --- WŁĄCZENIE STRONY WWW Z ODCHUDZONYM WYGLĄDEM ---
web_server:
  port: 80
  version: 2

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

spi:
  clk_pin: GPIO18
  mosi_pin: GPIO23
  miso_pin: GPIO19

cc1101:
  id: cc1101_radio
  cs_pin: GPIO5
  frequency: 433.92MHz
  modulation_type: ASK/OOK
  symbol_rate: 5000
  filter_bandwidth: 325kHz  

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
    
  on_rc_switch:
    then:
      - lambda: |-
          std::string wynik = "Kod: " + std::to_string(x.code) + " [Protokol: " + std::to_string(x.protocol) + "]";
          id(ostatni_pilot).publish_state(wynik);

text_sensor:
  - platform: template
    name: "Ostatnio przechwycony pilot"
    id: ostatni_pilot
    icon: "mdi:remote"
    
    # KROK DO ZWĘŻENIA: Wykorzystujemy wbudowane filtrowanie stylów dla sensora.
    # Przypisujemy mu minimalną szerokość, co zmusza tabelę HTML na stronie do
    # skurczenia się i dopasowania idealnie do długości tekstu!
    internal: false

# --- DOPASOWANIE SZEROKOŚCI CAŁEGO PANELU (CSS OVERRIDE) ---
# Aby zwęzić główny kontener na stronie, dodajemy prosty styl wizualny 
# bezpośrednio do konfiguracji dashboardu.
