# Měření kvality ovzduší v jeskyních pomocí ESP32

![Plakát projektu](images/F7DC485F-09BB-440B-AF06-2DACB5608FFC.png)

## 📡 O projektu
Tento projekt se zaměřuje na měření kvality ovzduší v náročném prostředí – konkrétně v hlubokých jeskyních, kde není dostupné Wi-Fi. Pomocí senzoru MQ-135 a převodníku ADS1115 zařízení měří koncentraci CO₂ a zobrazuje stav vzduchu na OLED displeji.

Data jsou dále odesílána přes MQTT na server, kde jsou uložena v InfluxDB a vizualizována v Grafaně. Pro provoz v terénu je navrženo použití LoRa technologie.

## 👨‍💻 Autoři
Jakub Lichnovský, Kryštof Moravec, Tomáš Lazar

---

## 🔧 Použité komponenty
- ESP32-WROOM-32D
- MQ-135 senzor
- ADS1115 I2C převodník
- OLED displej SH1106 (SPI, 128x64 px)
- (volitelně) LoRa modul
- Arduino IDE
- MQTT broker: Mosquitto
- Databáze: InfluxDB 2.x
- Vizualizace: Grafana
- Server: Ubuntu (VM)

---

## 📷 Ukázky funkčního zařízení

![OLED kvalita](images/IMG_6700.jpeg)
![OLED CO2](images/IMG_6701.jpeg)

---

## 🔌 Schéma zapojení
![Schéma zapojení](images/schematic_PJ.png)

---

## 📁 Struktura repozitáře
```
/src        → Arduino kód pro ESP32
/docs       → PDF dokumentace, plakát
/images     → Schémata, OLED fotky, vizualizace
```

---

## 🛠️ Spuštění projektu

### 1. Nahrání do ESP32
- Arduino IDE
- Knihovny: `Wire`, `SPI`, `U8g2lib`, `Adafruit_ADS1X15`, `WiFi`, `PubSubClient`
- Úprava Wi-Fi údajů v `main.ino`

### 2. Serverová část
- `sudo apt install mosquitto mosquitto-clients`
- Instalace InfluxDB, Telegraf, Grafana
- Konfigurace `mqtt.conf` pro Telegraf
- Vytvoření dashboardu v Grafaně

---

## 📈 Dotaz pro Grafanu (Flux)
```flux
from(bucket: "mqtt_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "ovzdusi" and r._field == "co2")
```

---

## 💡 Možná rozšíření
- Přenos přes LoRa
- Připojení dalších senzorů (teplota, vlhkost)
- Alarm při špatné kvalitě
- Offline logování na SD kartu

---

## ✅ Shrnutí
Projekt úspěšně propojuje hardwarovou a softwarovou část IoT aplikace, využívá cloudové technologie pro ukládání a vizualizaci dat, a je připraven k nasazení v reálném prostředí.

