Návod na spuštění a využití aplikace

1. Nahrání programu do ESP32
Požadavky:
•	Arduino IDE (verze 1.8+ nebo 2.x)
•	Knihovny:
o	<Wire.h> - 
o	<SPI.h> - 
o	<U8g2lib.h> - verze: 2.35.30
o	<Adafruit_ADS1X15.h> - verze: 2.5.0
o	<WiFi.h> -  
o	<PubSubClient.h> verze: 2.8
•	Balíček pro ESP32 (https://github.com/espressif/arduino-esp32)
V kódu upravit:
const char* ssid = "Vaše_WIFI";
const char* password = "Vaše_HESLO";
const char* mqtt_server = "172.16.144.130";
Postup:
  1.	Otevřete kód v Arduino IDE
  2.	Vyberte desku "ESP32 Dev Module"
  3.	Nahrajte do ESP32

2. Nastavení infrastruktury na serveru
Server může být Ubuntu/Debian
Balíčky:
•	Mosquitto
•	Telegraf
•	InfluxDB 2.x
•	Grafana

Mosquitto:
sudo apt update
sudo apt install -y mosquitto mosquitto-clients

InfluxDB 2.X:
wget https://dl.influxdata.com/influxdb/releases/influxdb2_2.7.4-1_amd64.deb
sudo dpkg -i influxdb2_2.7.4-1_*.deb
sudo systemctl enable influxdb
sudo systemctl start influxdb
Po instalaci InfluxDB spusťte:
influx setup
Vytvořte:
•	bucket (např. mqtt_data)
•	token
•	organizaci

Telegraf:
wget https://dl.influxdata.com/telegraf/releases/telegraf_1.30.0-1_$(dpkg --print-architecture).deb
sudo dpkg -i telegraf_1.30.0-1_*.deb
sudo systemctl start telegraf

Grafana:
wget https://dl.grafana.com/oss/release/grafana_10.2.3_amd64.deb
sudo dpkg -i grafana_10.2.3_*.deb
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
(Grafana bude dostupná na adrese: http://localhost:3000, přihlášení: admin/admin)

3. Konfigurace Telegrafu
Vytvořte soubor /etc/telegraf/telegraf.d/mqtt.conf:
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

Restartujte Telegraf:
sudo systemctl restart telegraf

4. Ověření komunikace
Otevřete terminál:
mosquitto_sub -t senzory/ovzdusi -v
V InfluxDB zkontrolujte příjem dat pomocí dotazu:
from(bucket: "mqtt_data")
  |> range(start: -15m)
  |> filter(fn: (r) => r._measurement == "ovzdusi")

5. Vytvoření dashboardu v Grafaně
  1.	Otevřete Grafanu (http://localhost:3000)
  2.	Přihlašte se (admin / admin)
  3.	Přidejte InfluxDB jako datový zdroj
  4.	Vytvořte panel s dotazem:
from(bucket: "mqtt_data")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "ovzdusi" and r._field == "co2")

Shrnutí
•	OLED zobrazuje místní hodnoty
•	MQTT odesílá data na server
•	Telegraf + InfluxDB je ukládají
•	Grafana zobrazí průběh a aktuální hodnoty

