#include <Wire.h>
#include <Keypad.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <WiFiClientSecure.h>

#define I2C_ADDR_LCD 0x3E
// --- 接続先：リポジトリ & デプロイキー ---
const char* UPDATE_URL = "https://raw.githubusercontent.com/shouchankeichan13-jpg/IPT/refs/heads/main/README.md";
const char* INFO_URL   = "https://script.google.com/macros/s/AKfycbwdqaDagGwJWI6n3wDBC_AJEYqHBrPbGMY55ZX5R8D0b7XkrMZagtUVByh1SY4qIggsWg/exec";

byte rowPins[4] = {27, 33, 32, 14}; 
byte colPins[3] = {15, 13, 12};     
char keys[4][3] = {{'3','2','1'},{'6','5','4'},{'9','8','7'},{'#','0','*'}};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, 4, 3);

enum Mode { HOME, MENU_MAIN, WIFI_SCANNING, WIFI_LIST, WIFI_INPUT, UPDATE_CHECK };
Mode currentMode = HOME;
int menuIdx = 0, wifiIdx = 0, wifiCount = 0;
String selectedSSID = "", inputPass = "", infoMsg = "IPT SYSTEM V11.5 READY... ";
unsigned long lastScroll = 0, lastInfoUpdate = 0;
int scrollIdx = 0;
bool needsRefresh = true;

const byte k_WIFI[] = {0xB2, 0xDD, 0xC0, 0xB0, 0xC8, 0xAF, 0xC4, 0x00}; // ｲﾝﾀｰﾈｯﾄ
const byte k_UPD[]  = {0xB1, 0xAF, 0xCC, 0xDF, 0xC3, 0xDE, 0xB0, 0xC4, 0x00}; // ｱｯﾌﾟﾃﾞｰﾄ

void lcd_cmd(byte c) { Wire.beginTransmission(I2C_ADDR_LCD); Wire.write(0x00); Wire.write(c); Wire.endTransmission(); delay(2); }
void lcd_data(byte d) { Wire.beginTransmission(I2C_ADDR_LCD); Wire.write(0x40); Wire.write(d); Wire.endTransmission(); }
void lcd_print(const char* s) { while (*s) lcd_data(*s++); }
void lcd_kana(const byte* m) { while (*m) lcd_data(*m++); }

// --- GitHub/GASからのデータ取得 ---
String fetchCloud(const char* url, bool follow) {
  if (WiFi.status() != WL_CONNECTED) return "";
  WiFiClientSecure client; client.setInsecure();
  HTTPClient http;
  String res = "";
  if (http.begin(client, url)) {
    if(follow) http.setFollowRedirects(HTTPC_STRICT_FOLLOW_REDIRECTS);
    if (http.GET() == HTTP_CODE_OK) res = http.getString();
    http.end();
  }
  return res;
}

void setup() {
  Wire.begin(26, 25);
  delay(500); lcd_cmd(0x38); lcd_cmd(0x39); lcd_cmd(0x14); lcd_cmd(0x73); 
  lcd_cmd(0x56); lcd_cmd(0x6C); lcd_cmd(0x38); lcd_cmd(0x0C); lcd_cmd(0x01);
  lcd_cmd(0x80); lcd_print("IPT OS V11.5");
  WiFi.mode(WIFI_STA);
}

void loop() {
  char key = keypad.getKey();

  // 定期お知らせ更新
  if (WiFi.status() == WL_CONNECTED && (millis() - lastInfoUpdate > 600000 || lastInfoUpdate == 0)) {
    String m = fetchCloud(INFO_URL, true);
    if(m != "") infoMsg = m + "      ";
    lastInfoUpdate = millis();
  }

  // ホーム画面：テロップ
  if (currentMode == HOME && millis() - lastScroll > 450) {
    lcd_cmd(0xC0);
    for (int i = 0; i < 16; i++) {
      int idx = (scrollIdx + i) % infoMsg.length();
      lcd_data((byte)infoMsg[idx]);
    }
    scrollIdx = (scrollIdx + 1) % infoMsg.length();
    lastScroll = millis();
  }

  if (key != NO_KEY) {
    if (key == '*') {
      lcd_cmd(0x01); currentMode = (currentMode == HOME) ? MENU_MAIN : HOME;
      menuIdx = 0; needsRefresh = true;
    } else {
      switch (currentMode) {
        case MENU_MAIN:
          if (key == '2') menuIdx = 0; if (key == '8') menuIdx = 1;
          if (key == '#') {
            lcd_cmd(0x01);
            if (menuIdx == 0) {
              lcd_print("SCANNING..."); WiFi.scanNetworks(true); currentMode = WIFI_SCANNING;
            } else {
              lcd_print("CHECKING..."); String p = fetchCloud(UPDATE_URL, false);
              lcd_cmd(0x01);
              if (p.indexOf("no update") != -1 || p.indexOf("V11.5") != -1) {
                lcd_print("NO UPDATE");
              } else { lcd_print("NEW VER FOUND!"); }
              delay(3000); currentMode = HOME;
            }
          }
          break;
        case WIFI_LIST:
          if (key == '2') wifiIdx = (wifiIdx > 0) ? wifiIdx - 1 : (wifiCount-1);
          if (key == '8') wifiIdx = (wifiIdx < wifiCount - 1) ? wifiIdx + 1 : 0;
          if (key == '#') { selectedSSID = WiFi.SSID(wifiIdx); inputPass = ""; currentMode = WIFI_INPUT; }
          break;
        case WIFI_INPUT:
          if (key == '#') { WiFi.begin(selectedSSID.c_str(), inputPass.c_str()); currentMode = HOME; }
          else { inputPass += key; }
          break;
      }
      needsRefresh = true;
    }
  }

  if (currentMode == WIFI_SCANNING && WiFi.scanComplete() >= 0) {
    wifiCount = WiFi.scanComplete(); currentMode = WIFI_LIST; needsRefresh = true;
  }

  if (needsRefresh && currentMode != HOME) {
    lcd_cmd(0x01);
    if (currentMode == MENU_MAIN) {
      lcd_cmd(0x80); lcd_print("MENU:"); lcd_cmd(0xC0); lcd_print("> ");
      if (menuIdx == 0) lcd_kana(k_WIFI); else lcd_kana(k_UPD);
    } else if (currentMode == WIFI_LIST) {
      lcd_cmd(0x80); lcd_print("SELECT SSID:");
      lcd_cmd(0xC0); if(wifiCount > 0) lcd_print(WiFi.SSID(wifiIdx).substring(0,16).c_str());
    } else if (currentMode == WIFI_INPUT) {
      lcd_cmd(0x80); lcd_print("PASS:"); lcd_cmd(0xC0); lcd_print(inputPass.c_str());
    }
    needsRefresh = false;
  }
}
