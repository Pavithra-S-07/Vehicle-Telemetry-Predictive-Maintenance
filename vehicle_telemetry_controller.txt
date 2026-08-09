#include <Arduino.h>
#include <Wire.h>
#include <WiFi.h>
#include <ThingSpeak.h>
#include <TinyGPSPlus.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

#include <SPI.h>
#include <mcp2515.h>

#include <LiquidCrystal_I2C.h>

//////////////////// FUNCTION PROTOTYPES ////////////////////
void readCAN();
void readGPS();
void readMPU();
void connectWiFi();
void checkThingSpeak();
void sendToThingSpeak();
void sendAlertSMS();
void bootScreen();
void updateLCD();
void printSerial();
String getLocationURL();

//////////////////// LCD ////////////////////
LiquidCrystal_I2C lcd(0x27, 16, 2);

//////////////////// WIFI ////////////////////
const char* WIFI_SSID = "abcd";
const char* WIFI_PASS = "9876543210";

unsigned long CHANNEL_ID = 3390678;
const char* WRITE_API_KEY = "YOUR_API_KEY";

WiFiClient client;

bool wifiOK = false;
bool thingSpeakOK = false;

//////////////////// GPS ////////////////////
TinyGPSPlus gps;
HardwareSerial SerialGPS(2);

#define GPS_RX 16
#define GPS_TX 17

//////////////////// GSM ////////////////////
HardwareSerial SerialGSM(1);

#define GSM_RX 26
#define GSM_TX 27

const char* ALERT_PHONE = "PHONE_NUMBER";

//////////////////// CAN ////////////////////
struct can_frame canMsg;
MCP2515 mcp2515(5);

struct EngineData {
  uint16_t rpm = 0;
  uint8_t speed = 0;
  uint8_t coolant = 0;
  uint8_t throttle = 0;
} engine;

//////////////////// MPU ////////////////////
Adafruit_MPU6050 mpu;

float AX = 0, AY = 0, AZ = 0;
float mpuTemp = 0;

int axisThreshold = 8;

bool tiltAlert = false;
bool alertSent = false;

//////////////////// TIMERS ////////////////////
unsigned long lastUpload = 0;
unsigned long lastLCD = 0;
unsigned long lastSerial = 0;

//////////////////// GPS LOCATION ////////////////////
String getLocationURL() {
  //  if (!gps.location.isValid()) return "GPS_NO_FIX";

  String url = "https://www.google.com/maps/search/?api=1&query=";
  url += String(gps.location.lat(), 6);
  url += ",";
  url += String(gps.location.lng(), 6);
  return url;
}

//////////////////// CAN ////////////////////
void readCAN() {
  if (mcp2515.readMessage(&canMsg) == MCP2515::ERROR_OK) {
    engine.rpm      = (canMsg.data[0] << 8) | canMsg.data[1];
    engine.speed    = canMsg.data[2];
    engine.coolant  = canMsg.data[3];
    engine.throttle = canMsg.data[4];
  }
}

//////////////////// GPS ////////////////////
void readGPS() {
  while (SerialGPS.available()) {
    gps.encode(SerialGPS.read());
  }
}

//////////////////// MPU ////////////////////
void readMPU() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  AX = a.acceleration.x;
  AY = a.acceleration.y;
  AZ = a.acceleration.z;

  mpuTemp = temp.temperature;

  tiltAlert = (abs(AX) >= axisThreshold || abs(AY) >= axisThreshold);
}

//////////////////// WIFI ////////////////////
void connectWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASS);

  unsigned long t = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - t < 8000) {
    delay(300);
  }

  wifiOK = (WiFi.status() == WL_CONNECTED);
}

//////////////////// THINGSPEAK ////////////////////
void checkThingSpeak() {
  ThingSpeak.begin(client);
  thingSpeakOK = true;
}

void sendToThingSpeak() {
  ThingSpeak.setField(1, engine.speed);
  ThingSpeak.setField(2, engine.rpm);
  ThingSpeak.setField(3, engine.coolant);
  ThingSpeak.setField(4, engine.throttle);
  ThingSpeak.setField(5, AX);
  ThingSpeak.setField(6, AY);
  ThingSpeak.setField(7, tiltAlert ? 1 : 0);
  ThingSpeak.setField(8, mpuTemp);

  ThingSpeak.writeFields(CHANNEL_ID, WRITE_API_KEY);
}

//////////////////// GSM SMS ////////////////////
void sendAlertSMS() {

  Serial.println("[GSM] Sending SMS...");

  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("SENDING SMS");
  lcd.setCursor(0,1);
  lcd.print("PLEASE WAIT");

  SerialGSM.println("AT+CMGF=1");
  delay(500);

  SerialGSM.print("AT+CMGS=\"");
  SerialGSM.print(ALERT_PHONE);
  SerialGSM.println("\"");
  delay(1000);

  SerialGSM.print("TELEMETRY ALERT: TILT DETECTED\n\n");

  SerialGSM.print("Speed: ");
  SerialGSM.print(engine.speed);
  SerialGSM.print("\nRPM: ");
  SerialGSM.print(engine.rpm);
  SerialGSM.print("\nCoolant: ");
  SerialGSM.print(engine.coolant);
  SerialGSM.print("\nTemp: ");
  SerialGSM.print(mpuTemp);
  SerialGSM.print("\n\nLocation:\n");
  SerialGSM.print(getLocationURL());

  SerialGSM.write(26);

  delay(3000);

  Serial.println("[GSM] SMS SENT");

  updateLCD();
}

//////////////////// LCD BOOT ////////////////////
void bootScreen() {

  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("TELEMETRY CTRL");
  lcd.setCursor(0,1);
  lcd.print("UNIT-STARTING...");
  delay(1500);

  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("WiFi:");
  lcd.print(wifiOK ? "OK" : "FAIL");

  lcd.setCursor(0,1);
  lcd.print("Cloud:");
  lcd.print(thingSpeakOK ? "OK" : "FAIL");
  delay(1500);

  lcd.clear();
  lcd.print("SYSTEM READY");
  delay(1000);
}

//////////////////// LCD LIVE ////////////////////
void updateLCD() {

  lcd.clear();

  lcd.setCursor(0,0);
  lcd.print("AX:");
  lcd.print(AX,1);
  lcd.print(" AY:");
  lcd.print(AY,1);

  lcd.setCursor(0,1);
  lcd.print("T:");
  lcd.print(mpuTemp,1);
  lcd.print("*C");
}

//////////////////// SERIAL DEBUG ////////////////////
void printSerial() {

  Serial.println("\n========= TELEMETRY SYSTEM =========");

  Serial.print("WiFi: ");
  Serial.println(wifiOK ? "CONNECTED" : "FAILED");

  Serial.print("ThingSpeak: ");
  Serial.println(thingSpeakOK ? "OK" : "FAIL");

  Serial.println("\nCAN DATA:");
  Serial.print("RPM: "); Serial.println(engine.rpm);
  Serial.print("Speed: "); Serial.println(engine.speed);
  Serial.print("Coolant: "); Serial.println(engine.coolant);
  Serial.print("Throttle: "); Serial.println(engine.throttle);

  Serial.println("\nMPU DATA:");
  Serial.print("AX: "); Serial.print(AX);
  Serial.print(" AY: "); Serial.print(AY);
  Serial.print(" AZ: "); Serial.println(AZ);
  Serial.print("Temp: "); Serial.println(mpuTemp);
  Serial.print("Tilt: "); Serial.println(tiltAlert ? "YES" : "NO");

  Serial.println("\nGPS DATA:");
  if (gps.location.isValid()) {
    Serial.print("Lat: "); Serial.println(gps.location.lat(), 6);
    Serial.print("Lon: "); Serial.println(gps.location.lng(), 6);
    Serial.print("URL: "); Serial.println(getLocationURL());
  } else {
    Serial.println("NO FIX");
  }

  Serial.println("====================================\n");
}

//////////////////// SETUP ////////////////////
void setup() {

  Serial.begin(115200);
  Wire.begin(21,22);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0,0);
  lcd.print("TELEMETRY CTRL");

  SerialGPS.begin(9600, SERIAL_8N1, GPS_RX, GPS_TX);
  SerialGSM.begin(9600, SERIAL_8N1, GSM_RX, GSM_TX);

  if (!mpu.begin()) {
    Serial.println("MPU FAILED");
    while(1);
  }

  SPI.begin();
  mcp2515.reset();
  mcp2515.setBitrate(CAN_500KBPS, MCP_8MHZ);
  mcp2515.setNormalMode();

  connectWiFi();
  checkThingSpeak();

  bootScreen();

  Serial.println("SYSTEM READY");
}

//////////////////// LOOP ////////////////////
void loop() {

  readCAN();
  readGPS();
  readMPU();

  if (tiltAlert && !alertSent) {
    sendAlertSMS();
    alertSent = true;
  }

  if (!tiltAlert) alertSent = false;

  if (millis() - lastUpload > 10000) {
    sendToThingSpeak();
    lastUpload = millis();
  }

  if (millis() - lastLCD > 2000) {
    updateLCD();
    lastLCD = millis();
  }

  if (millis() - lastSerial > 1000) {
    printSerial();
    lastSerial = millis();
  }

  delay(200);
}