
# Měření kvality ovzduší v jeskyních pomocí ESP32 a LoRa/MQTT

## Autoři
Jakub Lichnovský, Kryštof Moravec, Tomáš Lazar

---

## 1. Cíl projektu
Cílem projektu je navrhnout a realizovat zařízení pro měření kvality ovzduší v nepřístupném prostředí, konkrétně v hlubokých jeskyních, kde se může hromadit oxid uhličitý (CO₂) a jiné plyny. Projekt využívá senzor MQ-135, převodník ADS1115 a mikrokontroler ESP32. Hodnoty se zobrazují lokálně na OLED displeji a zároveň jsou odesílány na server pomocí MQTT protokolu.

Jeskyně jsou specifickým prostředím – nejen kvůli riziku nahromaděných plynů, ale i kvůli absenci běžného bezdrátového připojení (Wi-Fi, LTE). Proto jsme zvažovali, že by bylo vhodné přenos dat řešit pomocí LoRa (Long Range) modulu, který umožňuje bezdrátový přenos na dlouhé vzdálenosti s velmi nízkou spotřebou energie.

---

## 2. Použité komponenty

**Hardware:**
- ESP32 WROOM-32D
- MQ-135 (senzor kvality ovzduší)
- ADS1115 (I2C převodník pro přesné čtení analogových hodnot)
- OLED displej SH1106 (SPI, 128x64 px, 1.3”)
- LoRa modul (např. SX1276 – volitelně pro nasazení v terénu)
- Napájení přes USB nebo baterii

**Software a služby:**
- Arduino IDE (kód pro ESP32)
- Mosquitto (MQTT broker)
- Telegraf (sběr a přenos dat)
- InfluxDB 2.x (časová databáze)
- Grafana (vizualizace)
- Ubuntu Server (běží na virtuálním zařízení)

---

## 3. Schéma zapojení

![Schéma zapojení](schema.md)

---

## 4. OLED displej – ukázky výstupu

![OLED kvalita](oled_kvalita.md)
![OLED CO2](oled_co2.md)

---

## 5. Komunikace mezi prvky

**ESP32:**
- Čte data z MQ-135 přes ADS1115 (I2C)
- Vypočítává odhad CO₂ (v ppm) a určuje stav ovzduší: “Dobre”, “Prumerne”, “Spatne”
- Zobrazuje hodnoty na OLED displeji
- Každých 5 sekund posílá data do MQTT brokeru (nebo přes LoRa)

**Příklad odesílané zprávy:**
```json
{
  "co2": 842,
  "stav": "Prumerne"
}
```

**Mosquitto (MQTT broker):**
- Naslouchá tématu senzory/ovzdusi
- Přijímá zprávy z ESP32
- Předává je dál Telegrafu

**Telegraf:**
- Přijímá zprávy z MQTT
- Ukládá je do InfluxDB 2.x
- Konvertuje JSON zprávy na časové řady

**InfluxDB:**
- Ukládá hodnoty z měření do bucketu mqtt_data

**Grafana:**
- Zobrazuje hodnoty z InfluxDB v reálném čase
- Uživatel může sledovat CO₂ ve formě grafu a ukazatele kvality

---

## 6. Plakát projektu

![Plakát projektu](plakat.md)

---

## 7. Návod na spuštění

### Nahrání do ESP32
**Požadavky:**
- Arduino IDE (verze 1.8+ nebo 2.x)
- Knihovny: `Wire`, `SPI`, `U8g2lib`, `Adafruit_ADS1X15`, `WiFi`, `PubSubClient`
- ESP32 balíček: https://github.com/espressif/arduino-esp32

**V kódu upravit:**
```cpp
const char* ssid = "Vaše_WIFI";
const char* password = "Vaše_HESLO";
const char* mqtt_server = "172.16.144.130";
```

**Postup:**
1. Otevřete kód v Arduino IDE
2. Vyberte desku "ESP32 Dev Module"
3. Nahrajte do ESP32

---

### Server – instalace balíčků
```bash
sudo apt update
sudo apt install -y mosquitto mosquitto-clients

wget https://dl.influxdata.com/influxdb/releases/influxdb2_2.7.4-1_amd64.deb
sudo dpkg -i influxdb2_2.7.4-1_amd64.deb
sudo systemctl enable influxdb
sudo systemctl start influxdb

influx setup

wget https://dl.influxdata.com/telegraf/releases/telegraf_1.30.0-1_amd64.deb
sudo dpkg -i telegraf_1.30.0-1_amd64.deb
sudo systemctl start telegraf

wget https://dl.grafana.com/oss/release/grafana_10.2.3_amd64.deb
sudo dpkg -i grafana_10.2.3_amd64.deb
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

---

### Konfigurace Telegrafu
Soubor `/etc/telegraf/telegraf.d/mqtt.conf`:
```toml
[[inputs.mqtt_consumer]]
  servers = ["tcp://localhost:1883"]
  topics = ["senzory/ovzdusi"]
  data_format = "json"
  name_override = "ovzdusi"

[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "ZDE_VLOŽ_TOKEN"
  organization = "skolni_projekt"
  bucket = "mqtt_data"
```

Restart Telegraf:
```bash
sudo systemctl restart telegraf
```

---

### Ověření a Grafana

**MQTT kontrola:**
```bash
mosquitto_sub -t senzory/ovzdusi -v
```

**Flux dotaz:**
```flux
from(bucket: "mqtt_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "ovzdusi" and r._field == "co2")
```

---

## 8. Možná rozšíření
- Připojení dalších senzorů (např. teplota, vlhkost, pohyb)
- Alarm při překročení limitu CO₂
- Offline ukládání dat do SD karty
- Přenos přes LoRa (viz níže)

---

## 9. LoRa – plánované rozšíření

**Proč LoRa?**
V jeskyních nelze použít běžné Wi-Fi připojení. Proto bychom v budoucnu použili LoRa modul, který umožňuje bezdrátový přenos dat na delší vzdálenosti s minimální spotřebou energie.

**Způsob použití:**
- ESP32 odešle zprávu přes LoRa (místo MQTT)
- LoRa gateway mimo jeskyni zprávu přijme a předá do MQTT brokeru
- Dál vše funguje stejně

---

## 10. Shrnutí

Projekt propojuje senzorová data, mikrokontroler a serverovou infrastrukturu v reálném systému určeném pro sledování kvality vzduchu v jeskyních. Data se zobrazují jak na místě (OLED), tak na dálku (Grafana).

Přidaná hodnota projektu spočívá v tom, že řeší problém v reálném prostředí, kde není k dispozici klasická síťová infrastruktura – a navrhuje využití LoRa technologie jako možného řešení bezdrátového přenosu z podzemí.

Celý systém je navržen jako otevřený, modulární a snadno rozšiřitelný.

---

Pro celou dokumentaci otevřete ![Dokumentace](dokumentace.md)
Pro celý návod na spuštění otevřete ![Návod](navod_na_spusteni.md)
