# Fault-Detection-and-Alarm-System
#include <Wire.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <ESPAsyncWebServer.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

Adafruit_MPU6050 mpu;

// ---------------- WIFI ----------------
const char* ssid = "Lakshhh";
const char* password = "LP432006";

// ---------------- THINGSPEAK ----------------
String apiKey = "XE1KRF33UGSBPJYV";  
const char* tsURL = "http://api.thingspeak.com/update";

// ---------------- BUZZER ----------------
#define BUZ 25

// ---------------- WEB SERVER ----------------
AsyncWebServer server(80);

// ---------------- SENSOR VALUES ----------------
float A = 0, pitch = 0;
String statusTxt = "NORMAL";

// ---------------- FALL COUNTERS ----------------
int fallCounter = 0;
int warnCounter = 0;

// ---------------- HTML PAGE ----------------
const char webpage_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
<title>Fall Detection Dashboard</title>
<style>
body { font-family: Arial; text-align: center; padding: 20px; }
.box { background:#eee; padding:20px; border-radius:10px; margin:auto; width:300px; }
</style>
</head>
<body>
<h2>MPU6050 Fall Detection</h2>

<div class="box">
<p><b>Total Acceleration A:</b> <span id="acc">0</span></p>
<p><b>Pitch Angle:</b> <span id="pitch">0</span></p>
<p><b>Status:</b> <span id="status">NORMAL</span></p>
</div>

<script>
setInterval(() => {
fetch('/data').then(r => r.json()).then(d => {
document.getElementById("acc").innerText = d.A;
document.getElementById("pitch").innerText = d.pitch;
document.getElementById("status").innerText = d.status;
});
}, 1000);
</script>

</body>
</html>
)rawliteral";


// ---------------- SEND TO THINGSPEAK ----------------
void sendToThingSpeak(float A, float pitch) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;

    String url = tsURL;
    url += "?api_key=" + apiKey;
    url += "&field1=" + String(A);
    url += "&field2=" + String(pitch);

    http.begin(url);
    http.GET();
    http.end();

    Serial.println("Uploaded to ThingSpeak");
  }
}


// ----------------------------------------------------
void setup() {
  Serial.begin(115200);
  pinMode(BUZ, OUTPUT);
  digitalWrite(BUZ, LOW);

  Wire.begin(21, 22);

  if (!mpu.begin()) {
    Serial.println("MPU6050 NOT FOUND!");
    while (1);
  }

  // WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi ");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected!");

  // Web server routes
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *req){
    req->send_P(200, "text/html", webpage_html);
  });

  server.on("/data", HTTP_GET, [](AsyncWebServerRequest *req){
    String json = "{\"A\":" + String(A) + 
                  ",\"pitch\":" + String(pitch) + 
                  ",\"status\":\"" + statusTxt + "\"}";
    req->send(200, "application/json", json);
  });

  server.begin();
}


// ----------------------------------------------------
unsigned long lastTS = 0;

void loop() {
  sensors_event_t accel, gyro, temp;
  mpu.getEvent(&accel, &gyro, &temp);

  float ax = accel.acceleration.x;
  float ay = accel.acceleration.y;
  float az = accel.acceleration.z;

  pitch = atan2(ax, sqrt(ay*ay + az*az)) * 57.2958;
  A = sqrt(ax*ax + ay*ay + az*az);

  // ---- FALL LOGIC ----
  if (A > 20 || abs(pitch) > 65) {
    fallCounter++;
    warnCounter = 0;
  } else if (A > 13 || abs(pitch) > 40) {
    warnCounter++;
    fallCounter = 0;
  } else {
    fallCounter = warnCounter = 0;
  }

  if (fallCounter >= 2) {
    statusTxt = "FALL OCCURS";
    digitalWrite(BUZ, HIGH);
    delay(300);
    digitalWrite(BUZ, LOW);
  }
  else if (warnCounter >= 3) {
    statusTxt = "CHANCES OF FALL";
    digitalWrite(BUZ, HIGH);
    delay(120);
    digitalWrite(BUZ, LOW);
  }
  else {
    statusTxt = "NORMAL";
  }

  // ---- UPLOAD TO THINGSPEAK EVERY 15 SEC ----
  if (millis() - lastTS > 15000) {
    sendToThingSpeak(A, pitch);
    lastTS = millis();
  }

  delay(200);
}
