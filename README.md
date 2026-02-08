#include <Wire.h>
#include <Keypad.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <WiFiClientSecure.h>
#include <HTTPUpdate.h>
#include <time.h> // 時刻用ライブラリ

#define I2C_ADDR_LCD 0x3E
const char* VERSION = "V12.1";
// (各種URLは継承)
const char* UPDATE_URL = "https://raw.githubusercontent.com/shouchankeichan13-jpg/IPT-update/main/README.md";
const char* INFO_URL   = "https://script.google.com/macros/s/AKfycbwdqaDagGwJWI6n3wDBC_AJEYqHBrPbGMY55ZX5R8D0b7XkrMZagtUVByh1SY4qIggsWg/exec";

byte rowPins[4] = {27, 33, 32, 14}; 
byte colPins[3] = {15, 13, 12};     
char keys[4][3] = {{'3','2','1'},{'6','5','4'},{'9','8','7'},{'#','0','*'}};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, 4, 3);

enum Mode { HOME, MENU_MAIN, WIFI_SCANNING, WIFI_LIST, WIFI_INPUT, UPDATE_RUNNING };
Mode currentMode = HOME;
int menuIdx = 0;
String infoMsg = "IPT SYSTEM V12.1 READY... ";
unsigned long lastScroll = 0, lastClock = 0;
int scrollIdx = 0;
bool needsRefresh = true;

// 時刻設定
const char* ntpServer = "ntp.nict.jp"; // 日本標準時サーバー
const long  gmtOffset_sec = 9 * 3600;  // 日本はUTC+9時間
const int   daylightOffset_sec = 0;

void lcd_cmd(byte c) { Wire.beginTransmission(I2C_ADDR_LCD); Wire.write(0x00); Wire.write(c); Wire.endTransmission(); delay(2); }
void lcd_data(byte d) { Wire.beginTransmission(I2C_ADDR_LCD); Wire.write(0x40); Wire.write(d); Wire.endTransmission(); }
void lcd_print(const char* s) { while (*s) lcd_data(*s++); }

// --- 帝国時刻表示関数 ---
void displayClock() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) return;
  
  char timeStr[9];
  sprintf(timeStr, "%02d:%02d:%02d", timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec);
  
  lcd_cmd(0x80 + 8); // 1行目の右側に配置
  lcd_print(timeStr);
}

void setup() {
  Wire.begin(26, 25);
  delay(500); lcd_cmd(0x38); lcd_cmd(0x39); lcd_cmd(0x14); lcd_cmd(0x73); 
  lcd_cmd(0x56); lcd_cmd(0x6C); lcd_cmd(0x38); lcd_cmd(0x0C); lcd_cmd(0x01);
  
  lcd_cmd(0x80); lcd_print("IPT OS");
  WiFi.mode(WIFI_STA);
  
  // 起動時に時刻設定の準備
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
}

void loop() {
  char key = keypad.getKey();

  // --- HOME画面の定期更新（時計とテロップ） ---
  if (currentMode == HOME) {
    // 1秒ごとに時計を更新
    if (millis() - lastClock > 1000) {
      displayClock();
      lastClock = millis();
    }
    // テロップ（2行目）
    if (millis() - lastScroll > 450) {
      lcd_cmd(0xC0);
      for (int i = 0; i < 16; i++) {
        int idx = (scrollIdx + i) % infoMsg.length();
        lcd_data((byte)infoMsg[idx]);
      }
      scrollIdx = (scrollIdx + 1) % infoMsg.length();
      lastScroll = millis();
    }
  }

  // (以下、キー入力・Wi-Fiスキャン・OTAロジックを統合)
  // ...[前回のV12.0コードと同じ]...
  
  if (key != NO_KEY) {
    if (key == '*') {
      lcd_cmd(0x01); currentMode = (currentMode == HOME) ? MENU_MAIN : HOME;
      if(currentMode == HOME) { lcd_cmd(0x80); lcd_print("IPT OS"); }
      needsRefresh = true;
    }
    // (メニュー操作ロジック)
  }
}
