
# Dokumentace k projektu: Měření kvality ovzduší v jeskyních pomocí ESP32 a LoRa/MQTT

**Autoři:** Jakub Lichnovský, Kryštof Moravec, Tomáš Lazar

---

## 1. 🎯 Cíl projektu

Cílem projektu je navrhnout a realizovat zařízení pro měření kvality ovzduší v nepřístupném prostředí, konkrétně v hlubokých jeskyních, kde se může hromadit oxid uhličitý (CO₂) a jiné plyny.

Projekt využívá:
- senzor MQ-135,
- převodník ADS1115,
- mikrokontroler ESP32.

Výsledky se:
- zobrazují na OLED displeji,
- odesílají přes MQTT na server.

> Jeskyně neumožňují běžné připojení (Wi-Fi, LTE), proto je zvažováno použití **LoRa** pro přenos dat na delší vzdálenosti s nízkou spotřebou energie.

---

## 2. 🔌 Použité komponenty

### Hardware
- ESP32 WROOM-32D
- MQ-135 (senzor kvality ovzduší)
- ADS1115 (I2C převodník)
- OLED displej SH1106 (SPI, 128x64 px, 1.3”)
- LoRa modul (např. SX1276 – volitelně)
- Napájení přes USB nebo baterii

### Software a služby
- Arduino IDE
- Mosquitto (MQTT broker)
- Telegraf (sběr a přenos dat)
- InfluxDB 2.x (časová databáze)
- Grafana (vizualizace)
- Ubuntu Server (virtuální prostředí)

---

## 3. 💾 Kód pro ESP32

Kód najdete zde: ![Main.ino](/./code/main.ino)

### Použité knihovny:
- `<Wire.h>` – I2C komunikace
- `<SPI.h>` – SPI pro OLED
- `<U8g2lib.h>` – OLED displej SH1106
- `<Adafruit_ADS1X15.h>` – ADS1115
- `<WiFi.h>` – Wi-Fi připojení ESP32
- `<PubSubClient.h>` – MQTT komunikace

---

## 4. 🔁 Komunikace mezi jednotlivými prvky

### ESP32
- Čte hodnoty z MQ-135 pomocí ADS1115
- Vypočítá CO₂ (ppm)
- Určí stav: `Dobre`, `Prumerne`, `Spatne`
- Zobrazuje na OLED
- Odesílá zprávu každých 5 s

**Příklad zprávy:**
```json
{
  "co2": 842,
  "stav": "Prumerne"
}
```

### Mosquitto
- Naslouchá na tématu `senzory/ovzdusi`
- Předává data do Telegrafu

### Telegraf
- Přijímá zprávy z MQTT
- Převádí JSON na časové řady a ukládá do InfluxDB

### InfluxDB
- Bucket `mqtt_data`
- Ukládá: čas, hodnota CO₂, stav

### Grafana
- Zobrazuje data v reálném čase

---

## 5. 📊 Schéma zapojení

Viz obrázek ![Schéma zapojení](images/schema.md) nebo níže popis:

### I2C sběrnice (ADS1115):
- SDA → GPIO21
- SCL → GPIO22

### SPI sběrnice (OLED SH1106):
- SCK → GPIO18
- MOSI / SDA → GPIO23
- CS → GPIO5
- DC → GPIO2
- RES / RST → GPIO4

### Napájení:
- ADS1115 VDD → 5V
- ADS1115 GND → GND
- MQ-135 VCC → 5V
- MQ-135 GND → GND

### Signál:
- AOUT MQ-135 → A0 na ADS1115

---

## 6. 📡 LoRa modul (plánované rozšíření)

### Proč LoRa?
Wi-Fi v jeskyních nefunguje. LoRa umožní přenos na dlouhé vzdálenosti s nízkou spotřebou.

### Možnosti využití:
- ESP32 pošle zprávu přes LoRa
- Gateway ji zachytí a pošle na MQTT
- Zbytek procesu zůstává stejný

---

## 7. ⚙️ Konfigurace Telegrafu

Soubor `/etc/telegraf/telegraf.d/mqtt.conf`:
```toml
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
```

---

## 8. 📈 Příklad Flux dotazu pro Grafanu

```flux
from(bucket: "mqtt_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "ovzdusi" and r._field == "co2")
```

---

## 9. 🚀 Možná rozšíření

- Teplota, vlhkost, pohyb
- Alarm při překročení CO₂
- SD karta pro offline logování
- Export do Google Sheets
- Plná LoRa komunikace

---

## 10. ✅ Shrnutí

Projekt propojuje senzory, mikrokontroler a server do funkčního systému. Hodnoty se zobrazují na OLED i v Grafaně. Díky návrhu s LoRa je systém připraven pro reálné použití v nepřístupných místech.

Systém je **otevřený, modulární a rozšiřitelný**.
