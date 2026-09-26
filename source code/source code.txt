// ============================================================
// ESP32 MCP DASHBOARD (OPTIMIZED & STABLE)
// Sensor Dashboard + TFT + RGB + WiFi + MCP
// + MICRO SD FIRMWARE UPDATE
// + WEB WIFI / MCP URL SETUP
// + DOUBLE-TAP B&W/COLOR MODE TOGGLE
// + SINGLE-TAP SD DATA LOGGING WITH NTP TIME SYNC
// + NON-BLOCKING CALIBRATION & RECURSIVE MUTEX
//
// Developer: Munna Kumar
// ============================================================

#include <WiFi.h>
#include <WebSocketMCP.h>
#include <DHT.h>
#include <time.h>

#include <Adafruit_GFX.h>
#include <Adafruit_ST7735.h>
#include <SPI.h>

#include <ArduinoJson.h>
#include <Preferences.h>
#include <WebServer.h>
#include <DNSServer.h>

#include <FS.h>
#include <SD.h>
#include <Update.h>

#include <freertos/FreeRTOS.h>
#include <freertos/task.h>
#include <freertos/semphr.h>

// Recursive Mutex for safe multi-core hardware sharing
SemaphoreHandle_t hardwareMutex;

// ============================================================
// MICRO SD + FIRMWARE UPDATE & LOGGING
// ============================================================

#define SD_CS 13

#define UPGRADE_FILE        "/upgrade.bin"
#define UPGRADE_DONE_FILE   "/upgrade.done"
#define DOWNGRADE_FILE      "/downgrade.bin"
#define DOWNGRADE_DONE_FILE "/downgrade.done"

bool sdInitialized = false;

// ============================================================
// PIN DEFINITIONS (UNCHANGED)
// ============================================================

#define RGB_R 4
#define RGB_G 25
#define RGB_B 26

#define BLINK_LIGHT 2
#define SETTINGS_BUTTON 15

#define DHT_PIN 27
#define DHT_TYPE DHT22

#define MQ135_PIN 34
#define MQ6_PIN   35
#define MQ3_PIN   32
#define MQ9_PIN   33
#define MQ8_PIN   39

// ============================================================
// MQ SENSOR THRESHOLDS
// ============================================================

#define MQ135_MODERATE 1000
#define MQ135_POOR     2000
#define MQ6_LPG        1500
#define MQ6_GAS        2500
#define MQ3_ALCOHOL    1200
#define MQ3_HIGH       2000
#define MQ9_CO         1200
#define MQ9_HIGH       2000
#define MQ8_H2         1000
#define MQ8_HIGH       1800

// ============================================================
// TFT DISPLAY
// ============================================================

#define TFT_CS   5
#define TFT_RST  14
#define TFT_DC   12

Adafruit_ST7735 tft(TFT_CS, TFT_DC, TFT_RST);

// ============================================================
// OBJECTS
// ============================================================

DHT dht(DHT_PIN, DHT_TYPE);
Preferences preferences;
WebServer server(80);
DNSServer dnsServer;
WebSocketMCP mcpClient;

// ============================================================
// ACCESS POINT & NTP
// ============================================================

const char* AP_SSID = "MCP-ESP32 Setup";
IPAddress apIP(192, 168, 4, 1);
IPAddress gateway(192, 168, 4, 1);
IPAddress subnet(255, 255, 255, 0);

const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 19800; // IST
const int   daylightOffset_sec = 0;

// ============================================================
// SAVED SETTINGS & SENSOR VALUES
// ============================================================

String savedSSID = "";
String savedPassword = "";
String savedMCPURL = "";

int mq135Value = 0, mq6Value = 0, mq3Value = 0, mq9Value = 0, mq8Value = 0;
float temperature = NAN, humidity = NAN;

const int CLEAN_AIR_REFERENCE = 400;
int base135 = 0, base6 = 0, base3 = 0, base9 = 0, base8 = 0;

// ============================================================
// STATUS, THEME & SYSTEM FLAGS
// ============================================================

bool mcpConnected = false;
bool blinkLightState = false;
String currentRgbColor = "OFF";
bool isRgbBlinking = false; 
bool isBWMode = false;

unsigned long lastSensorUpdate = 0;
const unsigned long SENSOR_INTERVAL = 2000;

bool setupWebServerRunning = false;
bool captivePortalRunning = false;
bool inSetupMode = false; // Prevents sensor updates during setup
bool mainScreenInitialized = false;

// Non-blocking Calibration Variables
bool isCalibrating = false;
int calibStep = 0;
long tot135 = 0, tot6 = 0, tot3 = 0, tot9 = 0, tot8 = 0;
unsigned long lastCalibTick = 0;

// ============================================================
// TFT CHANGE TRACKING & COLORS
// ============================================================

int lastDisplayedMQ135Value = -1, lastDisplayedMQ6Value = -1, lastDisplayedMQ3Value = -1, lastDisplayedMQ9Value = -1, lastDisplayedMQ8Value = -1;
float lastDisplayedTemperature = NAN, lastDisplayedHumidity = NAN;
String lastDisplayedMQ135Status = "", lastDisplayedMQ6Status = "", lastDisplayedMQ3Status = "", lastDisplayedMQ9Status = "", lastDisplayedMQ8Status = "";
bool lastDisplayedMcpConnected = false, lastDisplayedWifiConnected = false;
String lastDisplayedRgbColor = "";

#define COLOR_BG       ST77XX_BLACK
#define COLOR_TEXT     ST77XX_WHITE
#define COLOR_ICON     ST77XX_CYAN
#define COLOR_GOOD     ST77XX_GREEN
#define COLOR_WARNING  ST77XX_YELLOW
#define COLOR_BAD      ST77XX_RED

#define ICON_COLOR_THERMO   ST77XX_RED
#define ICON_COLOR_DROP     ST77XX_BLUE
#define ICON_COLOR_AIR      ST77XX_GREEN
#define ICON_COLOR_FLAME    0xFD20 
#define ICON_COLOR_GLASS    ST77XX_MAGENTA
#define ICON_COLOR_SKULL    ST77XX_YELLOW
#define ICON_COLOR_ATOM     ST77XX_CYAN

// ============================================================
// BITMAP ICONS (8x8)
// ============================================================
const unsigned char icon_thermo[] PROGMEM = { 0x18, 0x24, 0x24, 0x24, 0x42, 0x5A, 0x5A, 0x3C }; 
const unsigned char icon_drop[] PROGMEM   = { 0x18, 0x3C, 0x7E, 0xFF, 0xFF, 0xFF, 0x7E, 0x3C }; 
const unsigned char icon_air[] PROGMEM    = { 0x00, 0x7C, 0x02, 0xFE, 0x80, 0x7C, 0x00, 0x00 }; 
const unsigned char icon_flame[] PROGMEM  = { 0x10, 0x30, 0x78, 0x7C, 0xFE, 0xFE, 0x7C, 0x38 }; 
const unsigned char icon_glass[] PROGMEM  = { 0xFF, 0x7E, 0x3C, 0x18, 0x18, 0x18, 0x3C, 0x7E }; 
const unsigned char icon_skull[] PROGMEM  = { 0x3C, 0x7E, 0xDB, 0xFF, 0xFF, 0x5A, 0x3C, 0x00 }; 
const unsigned char icon_atom[] PROGMEM   = { 0x18, 0x42, 0x99, 0xBD, 0xBD, 0x99, 0x42, 0x18 }; 

uint16_t theme(uint16_t color) {
  if (!isBWMode) return color;
  if (color == COLOR_BG) return ST77XX_BLACK;
  return ST77XX_WHITE; 
}

// ============================================================
// FUNCTION DECLARATIONS
// ============================================================

void drawMainScreen();
void updateSensors();
void startCalibration();
void processCalibration();
void setupWebRoutes();
String getPortalPage();
void applyColorPins(String color);
void setRGBColor(String color);
void logDataToSD();
String getTimeString();
void registerMcpTools();

// ============================================================
// STATUS LOGIC
// ============================================================

String getMQ135Status(int value) { if (value < MQ135_MODERATE) return "GOOD"; else if (value < MQ135_POOR) return "MODERATE"; return "POOR"; }
String getMQ6Status(int value) { if (value < MQ6_LPG) return "NORMAL"; else if (value < MQ6_GAS) return "LPG WARN"; return "GAS LEAK"; }
String getMQ3Status(int value) { if (value < MQ3_ALCOHOL) return "NORMAL"; else if (value < MQ3_HIGH) return "ALC WARN"; return "HIGH ALC"; }
String getMQ9Status(int value) { if (value < MQ9_CO) return "NORMAL"; else if (value < MQ9_HIGH) return "CO WARN"; return "HIGH CO"; }
String getMQ8Status(int value) { if (value < MQ8_H2) return "NORMAL"; else if (value < MQ8_HIGH) return "H2 WARN"; return "HIGH H2"; }

uint16_t getStatusColor(String status) {
  if (status == "GOOD" || status == "NORMAL") return COLOR_GOOD;
  if (status == "MODERATE" || status == "LPG WARN" || status == "ALC WARN" || status == "CO WARN" || status == "H2 WARN") return COLOR_WARNING;
  return COLOR_BAD;
}

// ============================================================
// DISPLAY FUNCTIONS
// ============================================================

void drawSensorIcon(int x, int y, int type) {
  tft.drawRect(x, y, 10, 10, theme(COLOR_ICON));
  if (type == 0) tft.drawBitmap(x + 1, y + 1, icon_air, 8, 8, theme(ICON_COLOR_AIR));
  else if (type == 1) tft.drawBitmap(x + 1, y + 1, icon_flame, 8, 8, theme(ICON_COLOR_FLAME));
  else if (type == 2) tft.drawBitmap(x + 1, y + 1, icon_glass, 8, 8, theme(ICON_COLOR_GLASS));
  else if (type == 3) tft.drawBitmap(x + 1, y + 1, icon_skull, 8, 8, theme(ICON_COLOR_SKULL));
  else if (type == 4) tft.drawBitmap(x + 1, y + 1, icon_atom, 8, 8, theme(ICON_COLOR_ATOM));
}

void resetDisplayTracking() {
  lastDisplayedMQ135Value = -1; lastDisplayedMQ6Value = -1; lastDisplayedMQ3Value = -1; lastDisplayedMQ9Value = -1; lastDisplayedMQ8Value = -1;
  lastDisplayedTemperature = NAN; lastDisplayedHumidity = NAN;
  lastDisplayedMQ135Status = ""; lastDisplayedMQ6Status = ""; lastDisplayedMQ3Status = ""; lastDisplayedMQ9Status = ""; lastDisplayedMQ8Status = "";
  lastDisplayedMcpConnected = !mcpConnected; lastDisplayedWifiConnected = !(WiFi.status() == WL_CONNECTED); lastDisplayedRgbColor = "__FORCE_UPDATE__";
}

void drawStaticMainScreen() {
  tft.fillScreen(theme(COLOR_BG));
  tft.setTextWrap(false); tft.setTextSize(1); tft.setTextColor(theme(COLOR_TEXT));
  tft.setCursor(2, 2); tft.print("ESP32 MCP DASHBOARD");
  tft.drawBitmap(3, 16, icon_thermo, 8, 8, theme(ICON_COLOR_THERMO));
  tft.drawBitmap(88, 16, icon_drop, 8, 8, theme(ICON_COLOR_DROP));
  drawSensorIcon(2, 31, 0); drawSensorIcon(2, 48, 1); drawSensorIcon(2, 65, 2); drawSensorIcon(2, 82, 3); drawSensorIcon(2, 99, 4); 
}

void updateDisplayItem(int x, int y, int w, int h, const String& label, int value, const String& status) {
  tft.fillRect(x, y, w, h, theme(COLOR_BG));
  tft.setTextSize(1); tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(x, y + 2);
  tft.print(label); tft.print(value); tft.print(" ");
  tft.setTextColor(theme(getStatusColor(status))); tft.print(status);
}

void drawMainScreen() {
  if (inSetupMode || isCalibrating) return;

  if (!mainScreenInitialized) {
    drawStaticMainScreen(); resetDisplayTracking(); mainScreenInitialized = true;
  }

  if (isnan(temperature) != isnan(lastDisplayedTemperature) || (!isnan(temperature) && temperature != lastDisplayedTemperature)) {
    tft.fillRect(15, 14, 70, 12, theme(COLOR_BG)); tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(15, 16);
    tft.print("T:"); if (isnan(temperature)) tft.print("--"); else { tft.print(temperature, 1); tft.print("C"); }
    lastDisplayedTemperature = temperature;
  }

  if (isnan(humidity) != isnan(lastDisplayedHumidity) || (!isnan(humidity) && humidity != lastDisplayedHumidity)) {
    tft.fillRect(100, 14, 60, 12, theme(COLOR_BG)); tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(100, 16);
    tft.print("H:"); if (isnan(humidity)) tft.print("--"); else { tft.print(humidity, 0); tft.print("%"); }
    lastDisplayedHumidity = humidity;
  }

  String s135 = getMQ135Status(mq135Value);
  if (mq135Value != lastDisplayedMQ135Value || s135 != lastDisplayedMQ135Status) { updateDisplayItem(15, 30, 145, 12, "AQ: ", mq135Value, s135); lastDisplayedMQ135Value = mq135Value; lastDisplayedMQ135Status = s135; }
  
  String s6 = getMQ6Status(mq6Value);
  if (mq6Value != lastDisplayedMQ6Value || s6 != lastDisplayedMQ6Status) { updateDisplayItem(15, 47, 145, 12, "LPG: ", mq6Value, s6); lastDisplayedMQ6Value = mq6Value; lastDisplayedMQ6Status = s6; }
  
  String s3 = getMQ3Status(mq3Value);
  if (mq3Value != lastDisplayedMQ3Value || s3 != lastDisplayedMQ3Status) { updateDisplayItem(15, 64, 145, 12, "Alc: ", mq3Value, s3); lastDisplayedMQ3Value = mq3Value; lastDisplayedMQ3Status = s3; }
  
  String s9 = getMQ9Status(mq9Value);
  if (mq9Value != lastDisplayedMQ9Value || s9 != lastDisplayedMQ9Status) { updateDisplayItem(15, 81, 145, 12, "CO: ", mq9Value, s9); lastDisplayedMQ9Value = mq9Value; lastDisplayedMQ9Status = s9; }
  
  String s8 = getMQ8Status(mq8Value);
  if (mq8Value != lastDisplayedMQ8Value || s8 != lastDisplayedMQ8Status) { updateDisplayItem(15, 98, 145, 12, "H2: ", mq8Value, s8); lastDisplayedMQ8Value = mq8Value; lastDisplayedMQ8Status = s8; }

  bool currentWifiConnected = (WiFi.status() == WL_CONNECTED);
  if (mcpConnected != lastDisplayedMcpConnected || currentWifiConnected != lastDisplayedWifiConnected || currentRgbColor != lastDisplayedRgbColor) {
    tft.fillRect(2, 115, 158, 13, theme(COLOR_BG)); tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(2, 117); tft.print("MCP:");
    if (mcpConnected) { tft.setTextColor(theme(COLOR_GOOD)); tft.print("OK"); } else { tft.setTextColor(theme(COLOR_BAD)); tft.print("OFF"); }
    tft.setTextColor(theme(COLOR_TEXT)); tft.print(" WiFi:");
    if (currentWifiConnected) { tft.setTextColor(theme(COLOR_GOOD)); tft.print("OK"); } else { tft.setTextColor(theme(COLOR_BAD)); tft.print("OFF"); }
    tft.setTextColor(theme(COLOR_TEXT)); tft.print(" RGB:");
    if (currentRgbColor != "OFF") tft.setTextColor(theme(COLOR_GOOD)); else tft.setTextColor(theme(COLOR_TEXT));
    tft.print(currentRgbColor); if (isRgbBlinking) { tft.setTextColor(theme(COLOR_WARNING)); tft.print(" (*)"); }
    lastDisplayedMcpConnected = mcpConnected; lastDisplayedWifiConnected = currentWifiConnected; lastDisplayedRgbColor = currentRgbColor;
  }
}

// ============================================================
// SENSORS & RGB
// ============================================================

void updateSensors() {
  if (inSetupMode || isCalibrating) return;
  mq135Value = constrain(analogRead(MQ135_PIN) - base135 + CLEAN_AIR_REFERENCE, 0, 4095);
  mq6Value   = constrain(analogRead(MQ6_PIN) - base6 + CLEAN_AIR_REFERENCE, 0, 4095);
  mq3Value   = constrain(analogRead(MQ3_PIN) - base3 + CLEAN_AIR_REFERENCE, 0, 4095);
  mq9Value   = constrain(analogRead(MQ9_PIN) - base9 + CLEAN_AIR_REFERENCE, 0, 4095);
  mq8Value   = constrain(analogRead(MQ8_PIN) - base8 + CLEAN_AIR_REFERENCE, 0, 4095);

  float newTemp = dht.readTemperature(); float newHum = dht.readHumidity();
  if (!isnan(newTemp)) temperature = newTemp;
  if (!isnan(newHum)) humidity = newHum;
}

void applyColorPins(String color) {
  digitalWrite(RGB_R, LOW); digitalWrite(RGB_G, LOW); digitalWrite(RGB_B, LOW);
  if (color == "RED") digitalWrite(RGB_R, HIGH);
  else if (color == "GREEN") digitalWrite(RGB_G, HIGH);
  else if (color == "BLUE") digitalWrite(RGB_B, HIGH);
  else if (color == "YELLOW") { digitalWrite(RGB_R, HIGH); digitalWrite(RGB_G, HIGH); }
  else if (color == "CYAN") { digitalWrite(RGB_G, HIGH); digitalWrite(RGB_B, HIGH); }
  else if (color == "MAGENTA" || color == "PINK") { digitalWrite(RGB_R, HIGH); digitalWrite(RGB_B, HIGH); }
  else if (color == "WHITE") { digitalWrite(RGB_R, HIGH); digitalWrite(RGB_G, HIGH); digitalWrite(RGB_B, HIGH); }
}

void setRGBColor(String color) {
  color.trim(); color.toUpperCase();
  if (color == "RED" || color == "GREEN" || color == "BLUE" || color == "YELLOW" || color == "CYAN" || color == "MAGENTA" || color == "PINK" || color == "WHITE") {
      currentRgbColor = color;
  } else { currentRgbColor = "OFF"; }
  isRgbBlinking = false; applyColorPins(currentRgbColor); lastDisplayedRgbColor = "__FORCE_UPDATE__";
  if (!inSetupMode && !isCalibrating) drawMainScreen();
}

// ============================================================
// NON-BLOCKING CALIBRATION
// ============================================================

void startCalibration() {
  isCalibrating = true; calibStep = 0; tot135 = 0; tot6 = 0; tot3 = 0; tot9 = 0; tot8 = 0;
  tft.fillScreen(theme(COLOR_BG)); mainScreenInitialized = false;
  tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(35, 20); tft.println("CALIBRATING");
  lastCalibTick = millis();
}

void processCalibration() {
  if (!isCalibrating) return;
  if (millis() - lastCalibTick >= 100) {
    lastCalibTick = millis();
    tot135 += analogRead(MQ135_PIN); tot6 += analogRead(MQ6_PIN); tot3 += analogRead(MQ3_PIN); tot9 += analogRead(MQ9_PIN); tot8 += analogRead(MQ8_PIN);
    
    calibStep++;
    int percent = (calibStep * 100) / 20;
    int barWidth = (percent * 126) / 100;
    
    tft.drawRect(15, 65, 130, 10, theme(COLOR_TEXT)); tft.fillRect(17, 67, barWidth, 6, theme(COLOR_GOOD));
    tft.fillRect(65, 85, 35, 10, theme(COLOR_BG)); tft.setCursor(65, 85); tft.print(percent); tft.print("%");
    
    if (calibStep >= 20) {
      base135 = tot135 / 20; base6 = tot6 / 20; base3 = tot3 / 20; base9 = tot9 / 20; base8 = tot8 / 20;
      preferences.begin("mcp_dash", false);
      preferences.putInt("base_135", base135); preferences.putInt("base_6", base6); preferences.putInt("base_3", base3); preferences.putInt("base_9", base9); preferences.putInt("base_8", base8);
      preferences.end();

      tft.fillScreen(theme(COLOR_BG)); tft.setTextColor(theme(COLOR_GOOD)); tft.setCursor(35, 45); tft.println("CALIBRATED!");
      
      xSemaphoreGiveRecursive(hardwareMutex);
      vTaskDelay(1500 / portTICK_PERIOD_MS);
      xSemaphoreTakeRecursive(hardwareMutex, portMAX_DELAY);
      
      isCalibrating = false; mainScreenInitialized = false; drawMainScreen();
    }
  }
}

// ============================================================
// MCP TOOLS (PROTECTED)
// ============================================================

WebSocketMCP::ToolResponse setRgbLightColorTool(const String& args) {
  DynamicJsonDocument doc(512); deserializeJson(doc, args);
  if (!doc["color"]) return WebSocketMCP::ToolResponse(false, "Missing color");
  
  xSemaphoreTakeRecursive(hardwareMutex, portMAX_DELAY);
  setRGBColor(doc["color"].as<String>());
  String outColor = currentRgbColor;
  xSemaphoreGiveRecursive(hardwareMutex);
  
  DynamicJsonDocument result(256); result["success"] = true; result["rgb_color"] = outColor;
  String output; serializeJson(result, output); return WebSocketMCP::ToolResponse(output);
}

WebSocketMCP::ToolResponse toggleRgbBlinkTool(const String& args) {
  DynamicJsonDocument doc(512); deserializeJson(doc, args);
  
  xSemaphoreTakeRecursive(hardwareMutex, portMAX_DELAY);
  if (currentRgbColor == "OFF") {
    xSemaphoreGiveRecursive(hardwareMutex);
    return WebSocketMCP::ToolResponse(false, "Turn on a color first before blinking.");
  }
  if (doc.containsKey("blink")) isRgbBlinking = doc["blink"].as<bool>(); else isRgbBlinking = !isRgbBlinking;
  if (!isRgbBlinking) applyColorPins(currentRgbColor);
  lastDisplayedRgbColor = "__FORCE_UPDATE__";
  if (!inSetupMode && !isCalibrating) drawMainScreen();
  bool blinkStatus = isRgbBlinking; String colorStatus = currentRgbColor;
  xSemaphoreGiveRecursive(hardwareMutex);

  DynamicJsonDocument result(256); result["success"] = true; result["is_blinking"] = blinkStatus; result["current_color"] = colorStatus;
  String output; serializeJson(result, output); return WebSocketMCP::ToolResponse(output);
}

WebSocketMCP::ToolResponse getSensorDataTool(const String& args) {
  xSemaphoreTakeRecursive(hardwareMutex, portMAX_DELAY);
  DynamicJsonDocument result(1024);
  result["temperature"] = temperature; result["humidity"] = humidity;
  result["mq135"]["value"] = mq135Value; result["mq135"]["status"] = getMQ135Status(mq135Value);
  result["mq6"]["value"] = mq6Value; result["mq6"]["status"] = getMQ6Status(mq6Value);
  result["mq3"]["value"] = mq3Value; result["mq3"]["status"] = getMQ3Status(mq3Value);
  result["mq9"]["value"] = mq9Value; result["mq9"]["status"] = getMQ9Status(mq9Value);
  result["mq8"]["value"] = mq8Value; result["mq8"]["status"] = getMQ8Status(mq8Value);
  xSemaphoreGiveRecursive(hardwareMutex);

  String output; serializeJson(result, output); return WebSocketMCP::ToolResponse(output);
}

void registerMcpTools() {
  mcpClient.registerTool("set_rgb_light_color", "Set RGB light color.", "{\"color\":\"string\"}", setRgbLightColorTool);
  mcpClient.registerTool("toggle_rgb_blink", "Enable/disable blink.", "{\"blink\":\"boolean\"}", toggleRgbBlinkTool);
  mcpClient.registerTool("get_sensor_data", "Get all sensor data.", "{}", getSensorDataTool);
}

void onConnectionStatus(bool connected) {
  xSemaphoreTakeRecursive(hardwareMutex, portMAX_DELAY);
  mcpConnected = connected;
  if (connected) registerMcpTools();
  if (!inSetupMode && !isCalibrating) drawMainScreen();
  xSemaphoreGiveRecursive(hardwareMutex);
}

// ============================================================
// SETTINGS
// ============================================================

void saveSettings() {
  preferences.begin("mcp_dash", false);
  preferences.putString("ssid", savedSSID); preferences.putString("pass", savedPassword); preferences.putString("mcp_url", savedMCPURL);
  preferences.end();
}
void loadSettings() {
  preferences.begin("mcp_dash", true);
  savedSSID = preferences.getString("ssid", ""); savedPassword = preferences.getString("pass", ""); savedMCPURL = preferences.getString("mcp_url", "");
  base135 = preferences.getInt("base_135", 0); base6 = preferences.getInt("base_6", 0); base3 = preferences.getInt("base_3", 0); base9 = preferences.getInt("base_9", 0); base8 = preferences.getInt("base_8", 0);
  preferences.end();
}
void eraseSettings() {
  preferences.begin("mcp_dash", false); preferences.clear(); preferences.end();
  savedSSID = ""; savedPassword = ""; savedMCPURL = ""; base135 = 0; base6 = 0; base3 = 0; base9 = 0; base8 = 0;
}
bool checkEraseSettings() {
  if (digitalRead(SETTINGS_BUTTON) != LOW) return false;
  tft.fillScreen(theme(COLOR_BG)); tft.setTextColor(theme(COLOR_WARNING)); tft.setCursor(25, 35); tft.println("ERASE SETTINGS?");
  unsigned long startTime = millis();
  while (digitalRead(SETTINGS_BUTTON) == LOW) {
    if ((millis() - startTime) >= 3000) {
      eraseSettings(); tft.fillScreen(theme(COLOR_BG)); tft.setTextColor(theme(COLOR_GOOD)); tft.setCursor(35, 55); tft.println("SETTINGS ERASED!"); delay(1500);
      while (digitalRead(SETTINGS_BUTTON) == LOW) delay(10); return true;
    } delay(10);
  }
  tft.fillScreen(theme(COLOR_BG)); tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(45, 60); tft.println("CANCELLED"); delay(800);
  return false;
}

// ============================================================
// SD UPDATE 
// ============================================================

void showSDUpdateProgress(size_t current, size_t total) {
  int percent = (current * 100) / total; int barWidth = (percent * 126) / 100;
  tft.fillRect(17, 67, barWidth, 11, theme(COLOR_GOOD));
  tft.fillRect(65, 90, 40, 15, theme(COLOR_BG)); tft.setCursor(65, 90); tft.setTextColor(theme(COLOR_TEXT)); tft.print(percent); tft.print("%");
}

bool checkAndUpdateFromSD() {
  if (!sdInitialized) return false;
  bool hasUpgrade = SD.exists(UPGRADE_FILE); bool hasDowngrade = SD.exists(DOWNGRADE_FILE);
  if (!hasUpgrade && !hasDowngrade) return false; 

  String targetFile = hasUpgrade ? UPGRADE_FILE : DOWNGRADE_FILE;
  String doneFile = hasUpgrade ? UPGRADE_DONE_FILE : DOWNGRADE_DONE_FILE;
  
  File updateBin = SD.open(targetFile);
  if (!updateBin || updateBin.isDirectory()) { if(updateBin) updateBin.close(); return false; }
  
  size_t fileSize = updateBin.size();
  if (fileSize > 0) {
    tft.fillScreen(theme(COLOR_BG)); tft.setTextSize(2); tft.setTextColor(theme(COLOR_WARNING));
    String modeStr = hasUpgrade ? "UPGRADING" : "DOWNGRADING";
    int xPos = (160 - (modeStr.length() * 12)) / 2; tft.setCursor(xPos > 0 ? xPos : 0, 25); tft.print(modeStr);
    tft.drawRect(15, 65, 130, 15, theme(COLOR_TEXT));

    if (Update.begin(fileSize, U_FLASH)) {
      Update.onProgress([](size_t progress, size_t total) { showSDUpdateProgress(progress, total); });
      size_t written = Update.writeStream(updateBin);
      if (Update.end() && Update.isFinished()) {
        updateBin.close(); SD.rename(targetFile, doneFile);
        tft.fillScreen(theme(COLOR_BG)); tft.setTextColor(theme(COLOR_GOOD)); tft.setCursor(35, 55); tft.print("SUCCESS!"); delay(2000);
        ESP.restart(); 
      }
    } 
  }
  updateBin.close(); 
  return false;
}

// ============================================================
// LOGGING (OPTIMIZED)
// ============================================================
void logDataToSD() {
  if (!sdInitialized) { 
    Serial.println("SD Not Initialized"); 
    return; 
  }
  
  // Check if the directory exists; if not, create it
  if (!SD.exists("/datalogs")) {
    SD.mkdir("/datalogs");
  }
  
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) { 
    Serial.println("Failed to obtain time"); 
    return; 
  }
  
  char fileName[64];
  // Set the file path using the timestamp within the folder
  strftime(fileName, sizeof(fileName), "/datalogs/%Y-%m-%d_%H-%M-%S.txt", &timeinfo);
  
  // Open the new file in write mode
  File file = SD.open(fileName, FILE_WRITE);
  if (!file) { 
    Serial.println("Failed to open file for writing"); 
    return; 
  }
  
  char logBuffer[256];
  snprintf(logBuffer, sizeof(logBuffer), 
           "Date: %04d-%02d-%02d | Time: %02d:%02d:%02d | Temp: %.1fC | Hum: %.0f%% | AQ: %d [%s] | LPG: %d [%s] | Alc: %d [%s] | CO: %d [%s] | H2: %d [%s]\r\n",
           timeinfo.tm_year + 1900, timeinfo.tm_mon + 1, timeinfo.tm_mday,
           timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec,
           temperature, humidity,
           mq135Value, getMQ135Status(mq135Value).c_str(),
           mq6Value, getMQ6Status(mq6Value).c_str(),
           mq3Value, getMQ3Status(mq3Value).c_str(),
           mq9Value, getMQ9Status(mq9Value).c_str(),
           mq8Value, getMQ8Status(mq8Value).c_str());

  if (file.print(logBuffer)) {
    Serial.print("Data logged to: ");
    Serial.println(fileName);
    // Show a small green dot on the TFT for confirmation
    tft.fillCircle(150, 10, 4, theme(COLOR_GOOD)); 
    delay(200); 
    tft.fillCircle(150, 10, 4, theme(COLOR_BG));
  } else {
    Serial.println("Write failed");
  }
  file.close();
}

// ============================================================
// NON-BLOCKING BUTTON HANDLER
// ============================================================
void handleButtonActions() {
  if (inSetupMode) return;
  static unsigned long lastPressTime = 0;
  static int tapCount = 0;
  static bool buttonHeld = false;
  static bool lastState = HIGH;
  bool currentState = digitalRead(SETTINGS_BUTTON);

  if (lastState == HIGH && currentState == LOW) { lastPressTime = millis(); buttonHeld = false; }
  
  if (currentState == LOW) {
    if (!buttonHeld && (millis() - lastPressTime >= 3000)) {
      buttonHeld = true; startCalibration(); tapCount = 0;
    }
  }
  
  if (lastState == LOW && currentState == HIGH) {
    if (!buttonHeld) { tapCount++; lastPressTime = millis(); }
  }

  if (currentState == HIGH && tapCount > 0 && (millis() - lastPressTime > 400)) {
    if (tapCount == 1) logDataToSD();
    else if (tapCount == 2) { isBWMode = !isBWMode; mainScreenInitialized = false; drawMainScreen(); }
    tapCount = 0; 
  }
  lastState = currentState;
}

// ============================================================
// WIFI & PORTAL LOGIC (NON-BLOCKING)
// ============================================================

void setupWebRoutes() {
  server.on("/", HTTP_GET, []() { server.send(200, "text/html", getPortalPage()); });
  server.on("/save", HTTP_POST, []() {
    String newSSID = server.arg("ssid"); String newPass = server.arg("pass"); String newMCP = server.arg("mcp_url");
    if (newSSID.length() > 0) {
      savedSSID = newSSID; savedPassword = newPass; savedMCPURL = newMCP; saveSettings(); 
      String html = "<!DOCTYPE html><html><head><meta name='viewport' content='width=device-width, initial-scale=1'><style>body{font-family:sans-serif;text-align:center;background:#203a43;color:#fff;display:flex;justify-content:center;align-items:center;height:100vh;margin:0;}</style></head><body><h2>Settings Saved! Rebooting...</h2></body></html>";
      server.send(200, "text/html", html);
      delay(1000); ESP.restart();
    } else {
      server.send(400, "text/html", "<h2>Error: SSID Empty</h2><a href='/'>Back</a>");
    }
  });
  server.onNotFound([]() { server.sendHeader("Location", "http://192.168.4.1/", true); server.send(302, "text/plain", ""); });
}

String getPortalPage() {
  String html = "<!DOCTYPE html><html><head><meta name='viewport' content='width=device-width, initial-scale=1'><style>body{font-family:sans-serif;background:#203a43;display:flex;justify-content:center;align-items:center;height:100vh;margin:0;}.card{background:#fff;padding:35px 25px;border-radius:12px;width:90%;max-width:360px;text-align:center;}input,button{width:100%;padding:14px;margin:10px 0;border-radius:8px;box-sizing:border-box;}button{background:#2c5364;color:#fff;border:none;cursor:pointer;font-weight:bold;}</style></head><body><div class='card'><h2>MCP Dashboard Setup</h2><form action='/save' method='POST'><input type='text' name='ssid' placeholder='WiFi SSID' required><input type='password' name='pass' placeholder='WiFi Password'><input type='text' name='mcp_url' placeholder='MCP Server (ws://ip:port)' value='" + savedMCPURL + "'><button type='submit'>Save & Connect</button></form></div></body></html>";
  return html;
}

bool connectToSavedWiFi() {
  if (savedSSID.length() == 0) return false;
  tft.fillScreen(theme(COLOR_BG)); tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(15, 50); tft.print("Connecting WiFi...");
  WiFi.mode(WIFI_STA); WiFi.setAutoReconnect(true); WiFi.persistent(false); WiFi.begin(savedSSID.c_str(), savedPassword.c_str());
  unsigned long startTime = millis();
  while (WiFi.status() != WL_CONNECTED) { 
    delay(500); 
    if (millis() - startTime >= 20000) {
      tft.fillScreen(theme(COLOR_BG)); tft.setCursor(30, 50); tft.setTextColor(theme(COLOR_BAD)); tft.print("WiFi Failed!"); delay(2000); return false; 
    } 
  }
  tft.fillScreen(theme(COLOR_BG)); tft.setTextColor(theme(COLOR_GOOD)); tft.setCursor(20, 50); tft.print("WiFi Connected!"); delay(1000);
  return true;
}

void startCaptivePortal() {
  WiFi.disconnect(true); delay(200); WiFi.mode(WIFI_AP); WiFi.softAPConfig(apIP, gateway, subnet); WiFi.softAP(AP_SSID, nullptr, 1, 0, 4);
  dnsServer.start(53, "*", apIP); captivePortalRunning = true; setupWebRoutes(); server.begin(); setupWebServerRunning = true;
  
  tft.fillScreen(theme(COLOR_BG)); tft.setTextColor(theme(COLOR_TEXT));
  tft.setCursor(30, 20); tft.println("MCP SETUP"); tft.setCursor(20, 38); tft.println("Connect WiFi:");
  tft.setTextColor(theme(COLOR_GOOD)); tft.setCursor(20, 55); tft.println(AP_SSID);
  tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(35, 75); tft.println("Open:"); tft.setCursor(35, 90); tft.println("192.168.4.1");
  
  inSetupMode = true; // Removes blocking while(true) loop
}

// ============================================================
// FREERTOS TASKS
// ============================================================

void TaskNetwork(void *pvParameters) {
  for (;;) {
    if (xSemaphoreTakeRecursive(hardwareMutex, portMAX_DELAY)) {
      mcpClient.loop();
      xSemaphoreGiveRecursive(hardwareMutex);
    }
    if (setupWebServerRunning) {
      if (captivePortalRunning) dnsServer.processNextRequest();
      server.handleClient();
    }
    vTaskDelay(10 / portTICK_PERIOD_MS); 
  }
}

void TaskSensors(void *pvParameters) {
  for (;;) {
    if (millis() - lastSensorUpdate >= SENSOR_INTERVAL) {
      lastSensorUpdate = millis();
      if (xSemaphoreTakeRecursive(hardwareMutex, portMAX_DELAY)) {
        updateSensors();
        if (!inSetupMode && !isCalibrating) drawMainScreen();
        xSemaphoreGiveRecursive(hardwareMutex);
      }
    }
    vTaskDelay(20 / portTICK_PERIOD_MS);
  }
}

// ============================================================
// SETUP
// ============================================================
void setup() {
  Serial.begin(115200); delay(500);
  
  hardwareMutex = xSemaphoreCreateRecursiveMutex();

  pinMode(BLINK_LIGHT, OUTPUT); pinMode(RGB_R, OUTPUT); pinMode(RGB_G, OUTPUT); pinMode(RGB_B, OUTPUT);
  pinMode(SETTINGS_BUTTON, INPUT_PULLUP); pinMode(SD_CS, OUTPUT); digitalWrite(SD_CS, HIGH);

  tft.initR(INITR_BLACKTAB); tft.setRotation(1); tft.setTextWrap(false);
  
  tft.fillScreen(theme(COLOR_BG)); tft.setTextSize(1);
  tft.setTextColor(theme(COLOR_TEXT)); tft.setCursor(48, 45); tft.println("DEVELOPER");
  tft.setTextColor(theme(COLOR_GOOD)); tft.setCursor(42, 65); tft.println("Munna Kumar"); delay(1800);

  // Initialize SD Card only once
  if (SD.begin(SD_CS)) sdInitialized = true;
  checkAndUpdateFromSD(); 
  
  dht.begin(); loadSettings();
  if (checkEraseSettings()) { startCaptivePortal(); }
  else if (savedSSID.length() == 0 || savedMCPURL.length() == 0) { startCaptivePortal(); } 
  else if (!connectToSavedWiFi()) { startCaptivePortal(); }

  if (!inSetupMode) { updateSensors(); drawMainScreen(); }
  
  if (WiFi.status() == WL_CONNECTED) {
    configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
    if (savedMCPURL.length() > 0) mcpClient.begin(savedMCPURL.c_str(), onConnectionStatus); 
  }

  xTaskCreatePinnedToCore(TaskNetwork, "NetworkTask", 10000, NULL, 1, NULL, 0); 
  xTaskCreatePinnedToCore(TaskSensors, "SensorsTask", 10000, NULL, 1, NULL, 1); 
}

// ============================================================
// MAIN LOOP (UI & BUTTON TASK)
// ============================================================
void loop() {
  
  if (xSemaphoreTakeRecursive(hardwareMutex, portMAX_DELAY)) {
    if (isCalibrating) {
      processCalibration();
    } else {
      handleButtonActions();
    }
    xSemaphoreGiveRecursive(hardwareMutex);
  }

  static unsigned long lastBlink = 0;
  if (millis() - lastBlink >= 500) {
    lastBlink = millis(); blinkLightState = !blinkLightState; digitalWrite(BLINK_LIGHT, blinkLightState);
  }

  static unsigned long lastRgbBlink = 0;
  static bool rgbBlinkState = false;
  if (isRgbBlinking && currentRgbColor != "OFF") {
    if (millis() - lastRgbBlink >= 500) {
      lastRgbBlink = millis(); rgbBlinkState = !rgbBlinkState;
      if (rgbBlinkState) applyColorPins(currentRgbColor); else applyColorPins("OFF");
    }
  }

  delay(5);
}
