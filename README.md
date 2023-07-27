/*
Important: This code is only for the DIY OUTDOOR OPEN AIR Presoldered Kit with the ESP-C3.

It is a high quality outdoor air quality sensor with dual PM2.5 modules and can send data over Wifi.

Kits are available: https://www.airgradient.com/open-airgradient/kits/

This code creates a HTTP client for scraping of sensor data by Prometheus

The codes needs the following libraries installed:
“WifiManager by tzapu, tablatronix” tested with version 2.0.11-beta
“pms” by Markusz Kakl version 1.1.0 (needs to be patched for 5003T model)

For built instructions and how to patch the PMS library: https://www.airgradient.com/open-airgradient/instructions/diy-open-air-presoldered-v11/

Note that below code only works with both PM sensor modules connected.

If you have any questions please visit our forum at https://forum.airgradient.com/

CC BY-SA 4.0 Attribution-ShareAlike 4.0 International License

Contributed by bartstar, 7/27/2023

*/

#include "PMS.h"
#include <HardwareSerial.h>
#include <Wire.h>
#include <HTTPClient.h>
#include <WiFiManager.h>

#define DEBUG true

HTTPClient client;

const int port = 9926;
const char* device = "AirGradient DIY Outdoor";
const char* ssid = "Snickers ";
String deviceId;
String idString;

PMS pms1(Serial0);
PMS::DATA data1;

float pm1Value01=0;
float pm1Value25=0;
float pm1Value10=0;
float pm1PCount=0;
float pm1temp = 0;
float pm1hum = 0;

PMS pms2(Serial1);
PMS::DATA data2;

float pm2Value01=0;
float pm2Value25=0;
float pm2Value10=0;
float pm2PCount=0;
float pm2temp = 0;
float pm2hum = 0;

int countPosition = 0;
int targetCount = 20;


String APIROOT = "http://hw.airgradient.com/";
WebServer server(port);

int loopCount = 0;

void IRAM_ATTR isr() {
  debugln("pushed");
}

// select board LOLIN C3 mini to flash
void setup() {
  Serial.begin(115200);
  // see https://github.com/espressif/arduino-esp32/issues/6983
  Serial.setTxTimeoutMs(0);   // <<<====== solves the delay issue

  debug("starting ...");
  debug("Serial Number: "+ getNormalizedMac());

  // default hardware serial, PMS connector on the right side of the C3 mini on the Open Air
  Serial0.begin(9600);

  // second hardware serial, PMS connector on the left side of the C3 mini on the Open Air
  Serial1.begin(9600, SERIAL_8N1, 0, 1);

  // led
  pinMode(10, OUTPUT);

  // push button
  pinMode(9, INPUT_PULLUP);
  attachInterrupt(9, isr, FALLING);

  pinMode(2, OUTPUT);
  digitalWrite(2, LOW);

  // give the PMSs some time to start
  countdown(3);

  connectToWifi();
  sendPing();
  switchLED(false);
  Serial.println("");
  Serial.print("Connected to ");
  Serial.println(ssid);
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());
  Serial.print("MAC Address: ");
  Serial.println(WiFi.macAddress());
  idString = "{id=\"" + String(device) + "\",mac=\"" + String(WiFi.macAddress()) + "\"}";
  server.on("/", HandleRoot);
  server.on("/metrics", HandleRoot);
  server.onNotFound(HandleNotFound);
  server.enableDelay(false);
  server.begin();
  Serial.println("HTTP server started at ip " + WiFi.localIP().toString() + ":" + String(port));
  server.send(200, "text/plain","HTTP send works");
}

void loop() {
  if(WiFi.status()== WL_CONNECTED) {
    if (pms1.readUntil(data1, 2000) && pms2.readUntil(data2, 2000)) {
      pm1Value01=pm1Value01+data1.PM_AE_UG_1_0;
      pm1Value25=pm1Value25+data1.PM_AE_UG_2_5;
      pm1Value10=pm1Value10+data1.PM_AE_UG_10_0;
      pm1PCount=pm1PCount+data1.PM_RAW_0_3;
      pm1temp=pm1temp+data1.AMB_TMP;
      pm1hum=pm1hum+data1.AMB_HUM;
      pm2Value01=pm2Value01+data2.PM_AE_UG_1_0;
      pm2Value25=pm2Value25+data2.PM_AE_UG_2_5;
      pm2Value10=pm2Value10+data2.PM_AE_UG_10_0;
      pm2PCount=pm2PCount+data2.PM_RAW_0_3;
      pm2temp=pm2temp+data2.AMB_TMP;
      pm2hum=pm2hum+data2.AMB_HUM;
      countPosition++;
      if (countPosition==targetCount) {
        pm1Value01 = pm1Value01 / targetCount;
        pm1Value25 = pm1Value25 / targetCount;
        pm1Value10 = pm1Value10 / targetCount;
        pm1PCount = pm1PCount / targetCount;
        pm1temp = pm1temp / targetCount;
        pm1hum = pm1hum / targetCount;
        pm2Value01 = pm2Value01 / targetCount;
        pm2Value25 = pm2Value25 / targetCount;
        pm2Value10 = pm2Value10 / targetCount;
        pm2PCount = pm2PCount / targetCount;
        pm2temp = pm2temp / targetCount;
        pm2hum = pm2hum / targetCount;
        postToServer(pm1Value01, pm1Value25,pm1Value10,pm1PCount, pm1temp,pm1hum,pm2Value01, pm2Value25,pm2Value10,pm2PCount, pm2temp,pm2hum);
        int i = 0;
        while (i < 10) {
          i++;
          server.handleClient();
          countdown(1);
        }
        countPosition=0;
        pm1Value01=0;
        pm1Value25=0;
        pm1Value10=0;
        pm1PCount=0;
        pm1temp=0;
        pm1hum=0;
        pm2Value01=0;
        pm2Value25=0;
        pm2Value10=0;
        pm2PCount=0;
        pm2temp=0;
        pm2hum=0;
      }
    }

  }
  countdown(2);
}

void debug(String msg) {
  if (DEBUG)
  Serial.print(msg);
}

void debug(int msg) {
  if (DEBUG)
  Serial.print(msg);
}

void debugln(String msg) {
  if (DEBUG)
  Serial.println(msg);
}

void debugln(int msg) {
  if (DEBUG)
  Serial.println(msg);
}

void switchLED(boolean ledON) {
 if (ledON) {
     digitalWrite(10, HIGH);
  } else {
    digitalWrite(10, LOW);
  }
}

void sendPing(){
      String payload = "{\"wifi\":" + String(WiFi.RSSI())
    + ", \"boot\":" + loopCount
    + "}";
    sendPayload(payload);
}

void postToServer(int pm1Value01, int pm1Value25, int pm1Value10, int pm1PCount, float pm1temp, float pm1hum,int pm2Value01, int pm2Value25, int pm2Value10, int pm2PCount, float pm2temp, float pm2hum) {
  String payload = "{\"wifi\":" + String(WiFi.RSSI())
  + ", \"pm01\":" + String((pm1Value01+pm2Value01)/2)
  + ", \"pm02\":" + String((pm1Value25+pm2Value25)/2)
  + ", \"pm10\":" + String((pm1Value10+pm2Value10)/2)
  + ", \"pm003_count\":" + String((pm1PCount+pm2PCount)/2)
  + ", \"atmp\":" + String((pm1temp+pm2temp)/20)
  + ", \"rhum\":" + String((pm1hum+pm2hum)/20)
  + ", \"boot\":" + loopCount
   + ", \"channels\": {"
      + "\"1\":{"
        + "\"pm01\":" + String(pm1Value01)
        + ", \"pm02\":" + String(pm1Value25)
        + ", \"pm10\":" + String(pm1Value10)
        + ", \"pm003_count\":" + String(pm1PCount)
        + ", \"atmp\":" + String(pm1temp/10)
        + ", \"rhum\":" + String(pm1hum/10)
        + "}"
      + ", \"2\":{"
      + " \"pm01\":" + String(pm1Value01)
      + ", \"pm02\":" + String(pm2Value25)
      + ", \"pm10\":" + String(pm2Value10)
      + ", \"pm003_count\":" + String(pm2PCount)
      + ", \"atmp\":" + String(pm2temp/10)
      + ", \"rhum\":" + String(pm2hum/10)
      + "}"
    + "}"
  + "}";
  Serial.println(payload);
  sendPayload(payload);
}

String GenerateMetrics(float pm125, float pm225, float temp1, float temp2, float rhum1, float rhum2) {
  String message = "";
  int stat = (pm125 + pm225)/2;
  message += "# HELP pm25 PM2.5 Particulate Matter Concentration US AQI\n";
  message += "# TYPE pm25 gauge\n";
  message += "pm25";
  message += idString + " ";
  message += String(PM_TO_AQI_US(stat));
  message += "\n\n";
  int stat_t = (temp1 + temp2)/20;
  message += "# HELP atmp Temp-F\n";
  message += "# TYPE atmp gauge\n";
  message += "atmp";
  message += idString + " ";
  message += String(stat_t);
  message += "\n\n";
  int stat_rh = (rhum1 + rhum2)/20;
  message += "# HELP rhum Rel Humidity (%)\n";
  message += "# TYPE rhum gauge\n";
  message += "rhum";
  message += idString + " ";
  message += String(stat_rh);
  message += "\n\n";
  Serial.print(message);
  return message;
}

void sendPayload(String payload) {
      if(WiFi.status()== WL_CONNECTED){
//      switchLED(true); // Removed turning on LED
      String url = APIROOT + "sensors/airgradient:" + getNormalizedMac() + "/measures";
      debugln(url);
      debugln(payload);
      client.setConnectTimeout(5 * 1000);
      client.begin(url);
      client.addHeader("content-type", "application/json");
      int httpCode = client.POST(payload);
      debugln(httpCode);
      client.end();
      resetWatchdog();
//      switchLED(false);
    }
    else {
      debug("post skipped, not network connection");
    }
    loopCount++;
}

void countdown(int from) {
  debug("\n");
  while (from > 0) {
    debug(String(from--));
    debug(" ");
    delay(1000);
  }
  debug("\n");
}

void resetWatchdog() {
    digitalWrite(2, HIGH);
    delay(20);
    digitalWrite(2, LOW);
}

// Wifi Manager
 void connectToWifi() {
   WiFiManager wifiManager;
   switchLED(true);
   //WiFi.disconnect(); //to delete previous saved hotspot
   String HOTSPOT = "AG-" + String(getNormalizedMac());
   wifiManager.setTimeout(180);


   if (!wifiManager.autoConnect((const char * ) HOTSPOT.c_str())) {
    switchLED(false);
     Serial.println("failed to connect and hit timeout");
     delay(6000);
   }

}

String getNormalizedMac() {
  String mac = WiFi.macAddress();
  mac.replace(":", "");
  mac.toLowerCase();
  return mac;
}

void HandleRoot() {
  server.send(200, "text/plain", GenerateMetrics(pm1Value25, pm2Value25, pm1temp, pm2temp, pm1hum, pm2hum) );
}

void HandleNotFound() {
  String message = "File Not Found\n\n";
  message += "URI: ";
  message += server.uri();
  message += "\nMethod: ";
  message += (server.method() == HTTP_GET) ? "GET" : "POST";
  message += "\nArguments: ";
  message += server.args();
  message += "\n";
  for (uint i = 0; i < server.args(); i++) {
    message += " " + server.argName(i) + ": " + server.arg(i) + "\n";
  }
  server.send(404, "text/html", message);
}

// Calculat PM2.5 US AQI
int PM_TO_AQI_US(int pm_25) {
  if (pm_25 < 12.0) return ((50 - 0) / (12.0 - 0.0) * (pm_25 - 0.0) + 0);
  else if (pm_25 < 35.4) return ((100-50) / (35.4 - 12.0) * (pm_25 - 12.0) + 50);
  else if (pm_25 < 55.4) return ((150-100) / (55.4 - 35.4) * (pm_25 - 35.4) + 100);
  else if (pm_25 < 150.4) return ((200 - 150) / (150.4 - 55.4) * (pm_25 - 55.4) + 150);
  else if (pm_25 < 250.4) return ((300 - 200) / (250.4 - 150.4) * (pm_25 - 150.4) + 200);
  else if (pm_25 < 350.4) return ((400 - 300) / (350.4 - 250.4) * (pm_25 - 250.4) + 300);
  else if (pm_25 < 500.4) return ((500 - 400) / (500.4 - 350.4) * (pm_25 - 350.4) + 400);
  else return 500;
}
