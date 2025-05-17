Dokumentace k projektu: Měření kvality ovzduší v jeskyních pomocí ESP32 a LoRa/MQTT
Jakub Lichnovský, Kryštof Moravec, Tomáš Lazar

1. Cíl projektu
Cílem projektu je navrhnout a realizovat zařízení pro měření kvality ovzduší v nepřístupném prostředí, konkrétně v hlubokých jeskyních, kde se může hromadit oxid uhličitý (CO₂) a jiné plyny. Projekt využívá senzor MQ-135, převodník ADS1115 a mikrokontroler ESP32. Hodnoty se zobrazují lokálně na OLED displeji a zároveň jsou odesílány na server pomocí MQTT protokolu.

Jeskyně jsou specifickým prostředím – nejen kvůli riziku nahromaděných plynů, ale i kvůli absenci běžného bezdrátového připojení (Wi-Fi, LTE). Proto jsme zvažovali, že by bylo vhodné přenos dat řešit pomocí LoRa (Long Range) modulu, který umožňuje bezdrátový přenos na dlouhé vzdálenosti s velmi nízkou spotřebou energie.
 
2. Použité komponenty
Hardware:
•	ESP32 WROOM-32D
•	MQ-135 (senzor kvality ovzduší)
•	ADS1115 (I2C převodník pro přesné čtení analogových hodnot)
•	OLED displej SH1106 (SPI, 128x64 px, 1.3”)
•	LoRa modul (např. SX1276 – volitelně pro nasazení v terénu)
•	Napájení přes USB nebo baterii

Software a služby:
•	Arduino IDE (kód pro ESP32)
•	Mosquitto (MQTT broker)
•	Telegraf (sběr a přenos dat)
•	InfluxDB 2.x (časová databáze)
•	Grafana (vizualizace)
•	Ubuntu Server (běží na virtuálním zařízení)
 
3. Kód pro ESP32
Celý kód použitý s ESP32 najdete v přiloženém souboru „pj_projekt_kod“.
Použité knihovnu ve funkčním kódu a jejich verze:
•	<Wire.h> - I2C komunikace 
•	<SPI.h> - SPI pro OLED 
•	<U8g2lib.h> - Knihovna pro OLED displej 
•	<Adafruit_ADS1X15.h> - Knihovna pro ADS1115 převodník 
•	<WiFi.h> - Wi-Fi připojení ESP32 
•	<PubSubClient.h> 
 
4. Komunikace mezi jednotlivými prvky
ESP32
•	Čte data z MQ-135 přes ADS1115 (I2C)
•	Vypočítává odhad CO₂ (v ppm) a určuje stav ovzduší: “Dobre”, “Prumerne”, “Spatne”
•	Zobrazuje hodnoty na OLED displeji
•	Každých 5 sekund posílá data do MQTT brokeru (nebo přes LoRa)

Příklad odesílané zprávy:
{
  "co2": 842,
  "stav": "Prumerne"
}
Mosquitto (MQTT broker)
•	Naslouchá tématu senzory/ovzdusi
•	Přijímá zprávy z ESP32
•	Předává je dál Telegrafu

Telegraf
•	Přijímá zprávy z MQTT
•	Ukládá je do InfluxDB 2.x
•	Konvertuje JSON zprávy na časové řady

InfluxDB
•	Ukládá hodnoty z měření do bucketu mqtt_data
•	V každém zápisu je: čas, hodnota CO₂, stav (“Dobre”, “Spatne”, …)

Grafana
•	Zobrazuje hodnoty z InfluxDB v reálném čase
•	Uživatel může sledovat CO₂ ve formě grafu a ukazatele kvality
 
5. Schéma zapojení – popis
 
I2C sběrnice (ADS1115):
    •    SDA (datová linka) → GPIO21 na ESP32
    •    SCL (hodinová linka) → GPIO22 na ESP32

SPI sběrnice (OLED SH1106):
    •    SCK (SPI clock) → GPIO18 na ESP32
    •    MOSI / SDA (SPI data) → GPIO23 na ESP32
    •    CS (chip select) → GPIO5 na ESP32
    •    DC (data/command přepínač) → GPIO2 na ESP32
    •    RES / RST (reset displeje) → GPIO4 na ESP32

Napájení:
    •    ADS1115 VDD → 5V na ESP32 (lepší stabilita než 3.3V)
    •    ADS1115 GND → GND na ESP32
    •    MQ-135 VCC → 5V na ESP32
    •    MQ-135 GND → GND na ESP32

Signál z MQ-135:
    •    AOUT (analogový výstup) → připojeno na A0 vstup ADS1115

6. LoRa modul (plánované rozšíření)

Proč LoRa?

V jeskyních nelze použít běžné Wi-Fi připojení. Proto bychom v budoucnu použili LoRa modul, který umožňuje bezdrátový přenos dat na delší vzdálenosti s minimální spotřebou energie.

Způsob použití:
•	ESP32 odešle zprávu přes LoRa (místo MQTT)
•	LoRa gateway mimo jeskyni zprávu přijme a předá do MQTT brokeru
•	Dál vše funguje stejně
 
7. Konfigurace Telegrafu (mqtt.conf)
[[inputs.mqtt_consumer]]
  servers = ["tcp://localhost:1883"]
  topics = ["senzory/ovzdusi"]
  data_format = "json"
  name_override = "ovzdusi"
  qos = 0

[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "••••••••••••••••"
  organization = "skolni_projekt"
  bucket = "mqtt_data"

8. Příklad Flux dotazu pro Grafanu
from(bucket: "mqtt_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "ovzdusi" and r._field == "co2")
 
9. Možná rozšíření
•	Připojení dalších senzorů (např. teplota, vlhkost, pohyb)
•	Alarm při překročení limitu CO₂
•	Offline ukládání dat do SD karty
•	Přenos přes LoRa (viz výše)
•	Export dat do CSV nebo do Google Sheets
 
10. Shrnutí
Tento projekt propojuje senzorová data, mikrokontroler a serverovou infrastrukturu v reálném systému určeném pro sledování kvality vzduchu v jeskyních. Data se zobrazují jak na místě (OLED), tak na dálku (Grafana).

Přidaná hodnota projektu spočívá v tom, že řeší problém v reálném prostředí, kde není k dispozici klasická síťová infrastruktura – a navrhuje využití LoRa technologie jako možného řešení bezdrátového přenosu z podzemí.

Celý systém je navržen jako otevřený, modulární a snadno rozšiřitelný pro další měření nebo prostředí.

