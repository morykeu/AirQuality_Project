// ====== Knihovny pro hardware a komunikaci ======
#include <Wire.h>                     // I2C komunikace
#include <SPI.h>                      // SPI pro OLED
#include <U8g2lib.h>                  // Knihovna pro OLED displej
#include <Adafruit_ADS1X15.h>         // Knihovna pro ADS1115 převodník
#include <WiFi.h>                     // Wi-Fi připojení ESP32
#include <PubSubClient.h>            // MQTT klient

// ====== Připojení k Wi-Fi síti ======
const char* ssid = "TVÁ_WIFI";                  // Název Wi-Fi sítě
const char* password = "TVÉ_HESLO";             // Heslo k Wi-Fi
const char* mqtt_server = "172.16.144.130";     // IP adresa MQTT serveru

WiFiClient espClient;
PubSubClient client(espClient);

// ====== Inicializace OLED displeje (SPI, SH1106) ======
// Piny: CS = 5, DC = 2, RST = 4
U8G2_SH1106_128X64_NONAME_F_4W_HW_SPI oled(U8G2_R0, 5, 2, 4);

// ====== ADS1115 převodník (přes I2C) ======
Adafruit_ADS1115 ads;

// ====== Proměnné pro zobrazení a měření ======
unsigned long lastSwitch = 0;     // Čas posledního přepnutí displeje
bool showCO2 = false;             // Stav přepínače OLED výstupu
bool adsReady = false;            // Informace, zda je ADS1115 nalezen
float co2ppm = 0;                 // Vypočtená CO₂ koncentrace
String kvalita = "---";           // Slovní hodnocení kvality ovzduší
unsigned long lastMqttSend = 0;   // Čas posledního odeslání MQTT zprávy

// ====== Funkce pro připojení k Wi-Fi ======
void setup_wifi() {
  WiFi.begin(ssid, password);
  Serial.print("Připojování k WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi připojeno");
}

// ====== Funkce pro připojení k MQTT brokeru ======
void reconnect() {
  while (!client.connected()) {
    Serial.print("Připojování k MQTT...");
    if (client.connect("ESP32_OVZDUSI")) {
      Serial.println(" připojeno.");
    } else {
      Serial.print(" chyba, kód=");
      Serial.print(client.state());
      delay(2000);
    }
  }
}

// ====== Nastavení při startu zařízení ======
void setup() {
  Serial.begin(115200);                // Seriová komunikace

  oled.begin();                        // Spuštění OLED displeje
  oled.clearBuffer();
  oled.setFont(u8g2_font_ncenB08_tr);
  oled.drawStr(0, 24, "Inicializace OLED");
  oled.sendBuffer();
  delay(500);

  Wire.begin(21, 22);                  // Inicializace I2C sběrnice (SDA, SCL)
  Wire.setClock(100000);              // Rychlost I2C
  adsReady = ads.begin(0x4B);         // Pokus o nalezení ADS1115 na adrese 0x4B

  setup_wifi();                        // Připojení k Wi-Fi
  client.setServer(mqtt_server, 1883); // Nastavení IP a portu MQTT brokeru
}

// ====== Hlavní smyčka programu ======
void loop() {
  // Připojení k MQTT brokeru (pokud došlo k odpojení)
  if (!client.connected()) {
    reconnect();
  }
  client.loop(); // MQTT klient musí být pravidelně volán

  // === MĚŘENÍ CO₂ pomocí ADS1115 ===
  int16_t adcValue = -1;
  float voltage = 0;
  co2ppm = 0;

  if (adsReady) {
    adcValue = ads.readADC_SingleEnded(0);             // Čtení z kanálu A0
    voltage = adcValue * (3.3 / 32767.0);               // Přepočet na napětí
    co2ppm = map(adcValue, 0, 32767, 400, 5000);        // Hrubý převod na ppm

    // Slovní hodnocení kvality
    if (adcValue <= 5000) {
      kvalita = "Dobre";
    } else if (adcValue <= 15000) {
      kvalita = "Prumerne";
    } else {
      kvalita = "Spatne";
    }

    // Výpis do seriového monitoru
    Serial.print("CO2: ");
    Serial.print(co2ppm);
    Serial.print(" ppm | Kvalita: ");
    Serial.println(kvalita);
  } else {
    kvalita = "---";
    Serial.println("ADS1115 nenalezen!");
  }

  // === ODESLÁNÍ MQTT zprávy každých 5 sekund ===
  if (millis() - lastMqttSend > 5000 && adsReady) {
    lastMqttSend = millis();
    String payload = "{\"co2\": " + String((int)co2ppm) + ", \"stav\": \"" + kvalita + "\"}";
    client.publish("senzory/ovzdusi", payload.c_str());
    Serial.println("MQTT zpráva odeslána: " + payload);
  }

  // === Přepínání OLED výpisu (každé 2 s) ===
  if (millis() - lastSwitch > 2000) {
    showCO2 = !showCO2;
    lastSwitch = millis();
  }

  // === Výpis na OLED displej ===
  oled.clearBuffer();
  oled.setFont(u8g2_font_ncenB08_tr);

  if (showCO2) {
    oled.drawStr(0, 14, "CO2 koncentrace:");
    oled.drawStr(0, 40, adsReady ? (String((int)co2ppm) + " ppm").c_str() : "--- ppm");
  } else {
    oled.drawStr(0, 14, "Kvalita ovzdusi:");
    oled.drawStr(0, 40, kvalita.c_str());
  }

  if (!adsReady) {
    oled.setFont(u8g2_font_5x7_tr);
    oled.drawStr(2, 62, "ADS error");
  }

  oled.sendBuffer();
  delay(100);
}
