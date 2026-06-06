/* =============================================================================================
 * 🚀 SMART SATELLITE SYSTEM - v12.5 (RANDOM NIGHT SKY EDITION)
 * ESP32C3 SUPER MINI + OLED 128x32 Pixel + SENSOR SHT3X
 * [PROJECT MANAGER]  : TRAN NAM
 * SOURCE DESIGN      : HuyVector & Gemini AI Collaborator
 * ĐỒ HỌA NÂNG CẤP     : Sao đêm nhảy vị trí NGẪU NHIÊN (Random) khi có mây trôi | Mượt mà 100%
 * =============================================================================================
 */

#include <Wire.h>                 
#include <Adafruit_GFX.h>         
#include <Adafruit_SSD1306.h>     
#include <Adafruit_SHT31.h>       
#include <WebServer.h>            
#include <DNSServer.h>            
#include <WiFiManager.h>          
#include <time.h>                 
#include <HTTPClient.h>           
#include <ArduinoJson.h>          

#define SCREEN_WIDTH 128          
#define SCREEN_HEIGHT 32          

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
Adafruit_SHT31 sht31 = Adafruit_SHT31();

// --- CẢM BIẾN SHT31 ---
float filteredTemp = NAN, filteredHum = NAN; 
float rawTemp = NAN, rawHum = NAN;           
const float alpha = 0.1;                      
unsigned long lastSensorRetry = 0, lastSensorRead = 0;
bool isSensorOnline = false;                 

// --- HARDWARE LED ---
#define LED_PIN_6 6               
#define LED_PIN_5 5               
#define LED_PIN_4 4               
#define LED_DAY_BRIGHTNESS   10   
#define LED_NIGHT_BRIGHTNESS 5    
uint8_t ledBrightness = 10;       

// --- KIỂM SOÁT WIFI ---
bool laBanDem = false;            
unsigned long lastSyncWiFi = 0;
const unsigned long syncInterval = 1800000; 
bool hasFirstSyncEver = false;             
bool reconnectingWiFi = false;             
unsigned long reconnectStart = 0;
bool portalStarted = false;                

WiFiManager wm;

// --- OPENWEATHERMAP ---
String apiKey = "faeeb2ffc93fba572b6717dcc030ec59";
String lat = "10.4851";
String lon = "105.6176";
String serverPath = "http://api.openweathermap.org/data/2.5/weather?lat=" + lat + "&lon=" + lon + "&appid=" + apiKey + "&units=metric";
String trangThaiThoiTiet = "NANG";         
bool coDuLieuOnline = false;               
int gioCapNhatAPI = 0, phutCapNhatAPI = 0;  
int gioHienTaiHeThong = 12;                

// --- TIMEOUT MẤT MẠNG (45 PHÚT) ---
unsigned long thoiGianCapNhatMangCuoi = 0;  
const unsigned long timeoutMatMang = 2700000; 

// --- LOGIC CHỮ CUỘN TẦNG 2 ---
int X_chuChay = 32;                       
unsigned long lastScrollTime = 0;
unsigned long waitStartTime = 0;
bool dangChoNghi20s = false;               
String chuoiChayTongHop = "";              
String chuoiChayOwmDungSan = "P.MY NGAI-P.CAO LANH, DONG THAP"; 

// --- TỌA ĐỘ 2 ĐÁM MÂY BAY LƯỚT LỆCH PHA TẦNG 3 ---
int X_may1 = -16; 
int X_may2 = -36; 
unsigned long lastScrollTimeMay = 0;

const char* const daysOfWeekShort[] PROGMEM = {"SUN", "MON", "TUE", "WED", "THU", "FRI", "SAT"};
const char* const monthsShort[] PROGMEM = {"JAN", "FEB", "MAR", "APR", "MAY", "JUN", "JUL", "AUG", "SEP", "OCT", "NOV", "DEC"};

// --- ANTI-BURN-IN GLOBAL SHIFT ---
static int8_t burnShiftX = 0, burnShiftY = 0; 
static int8_t textShiftX = 0;                  
static int8_t iconShiftX = 0, iconShiftY = 0;  
static unsigned long lastShiftTime = 0;        

// --- BIẾN LƯU TỌA ĐỘ RANDOM CHO SAO ĐÊM ---
static int starRandX1 = 6;
static int starRandY1 = 100;
static int starRandX2 = 22;
static int starRandY2 = 115;
static unsigned long lastStarRandomTime = 0;

// --- BITMAPS CẢI TIẾN CHUẨN ĐỒ HỌA MỚI ---
const uint8_t icon_Trang_Khuyet_Nho[] PROGMEM = {
  0x03, 0x80, 0x07, 0x00, 0x0e, 0x00, 0x0c, 0x00, 0x0c, 0x00, 0x0e, 0x00, 0x07, 0x00, 0x03, 0x80
};

const uint8_t icon_Set_Cai_Tien[] PROGMEM = {
  0x0c, 0x18, 0x30, 0x60, 0xff, 0x38, 0x30, 0x60, 0xc0, 0x80, 0x00, 0x00
};

const uint8_t icon_May_Pixel_Theo_Anh[] PROGMEM = {
  0x07, 0x00, 0x18, 0xe0, 0x23, 0x10, 0x44, 0x08, 0x88, 0x04, 0x90, 0x06,
  0x80, 0x01, 0x40, 0x01, 0x22, 0x02, 0x1d, 0x1c, 0x00, 0xe0
};

void recoverI2CBus() {
  Wire.end(); delay(5);
  pinMode(8, OUTPUT); pinMode(9, INPUT_PULLUP); delayMicroseconds(2);
  for (int i = 0; i < 9; i++) { digitalWrite(8, HIGH); delayMicroseconds(5); digitalWrite(8, LOW); delayMicroseconds(5); }
  Wire.begin(8, 9); Wire.setClock(400000); delay(5);
}

void layThoiTietVeTinhOnline() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http; http.begin(serverPath); http.setTimeout(3000); 
    int httpResponseCode = http.GET();
    if (httpResponseCode == 200) {
      String payload = http.getString();
      DynamicJsonDocument doc(1024); DeserializationError err = deserializeJson(doc, payload);
      if (err) { coDuLieuOnline = false; http.end(); return; }
      
      int weatherId = doc["weather"][0]["id"];
      if (weatherId >= 200 && weatherId <= 232) trangThaiThoiTiet = "DONG_SET";
      else if (weatherId >= 300 && weatherId <= 531) trangThaiThoiTiet = "MUA";
      else if (weatherId >= 801 && weatherId <= 804) trangThaiThoiTiet = "MAY";
      else trangThaiThoiTiet = "NANG";

      struct tm timeinfo;
      if (getLocalTime(&timeinfo)) { 
        gioCapNhatAPI = timeinfo.tm_hour; 
        phutCapNhatAPI = timeinfo.tm_min; 
        gioHienTaiHeThong = timeinfo.tm_hour; 
      }
      coDuLieuOnline = true; thoiGianCapNhatMangCuoi = millis();
      
      char formatCheck[32];
      snprintf(formatCheck, sizeof(formatCheck), " (OWM SYNC AT: %02d:%02d)", gioCapNhatAPI, phutCapNhatAPI);
      chuoiChayOwmDungSan = String(F("P.MY NGAI-P.CAO LANH, DONG THAP")) + String(formatCheck);
      
    } else { coDuLieuOnline = false; }
    http.end();
  } else { coDuLieuOnline = false; }
}

void trySyncTime() {
  if (WiFi.status() == WL_CONNECTED) {
    configTime(7 * 3600, 0, "asia.pool.ntp.org", "time.google.com");
    struct tm timeinfo;
    if (getLocalTime(&timeinfo, 1500)) {
      hasFirstSyncEver = true; lastSyncWiFi = millis(); gioHienTaiHeThong = timeinfo.tm_hour;
      layThoiTietVeTinhOnline(); portalStarted = false; 
      WiFi.disconnect(false); WiFi.mode(WIFI_OFF); 
    }
  }
}

void drawCyberLaserLines(int y1, int y2) {
  display.drawFastHLine(0, y1, 32, SSD1306_WHITE);
  display.drawFastHLine(0, y2, 32, SSD1306_WHITE);
}

// --- HÀM VẼ ICON TẦNG 3 (ĐÃ CẬP NHẬT RANDOM SAO ĐÊM) ---
void veChuyenTrangIconFooter(int x_icon_base, int y_icon, bool checkDem, unsigned long currentMillis) {
  int x = x_icon_base + iconShiftX; 
  int y = y_icon + iconShiftY;
  unsigned long pulseModulo = currentMillis % 15000;

  // 7 giây đầu hiển thị cảm biến Nhiệt độ / Độ ẩm bình thường
  if (pulseModulo < 7000) {
    if (isSensorOnline && !isnan(filteredTemp) && !isnan(filteredHum)) {
      display.setCursor(1 + textShiftX, 102); 
      display.print((int)filteredTemp); display.print((char)247); display.print(F("C")); 
      display.setCursor(0 + textShiftX, 116); 
      display.print(F("H:")); display.print((int)filteredHum); display.print(F("%")); 
    } else {
      display.setCursor(4 + textShiftX, 102); display.print(F("--")); display.print((char)247); display.print(F("C"));
      display.setCursor(0 + textShiftX, 116); display.print(F("H:--%"));
    }
    return;
  }

  // --- TRẠNG THÁI NĂNG (ĐÊM: TRĂNG NHỎ + SAO NHÁY CHỚP) ---
  if (trangThaiThoiTiet == "NANG") {
    if (checkDem) {
      int x_trang = 12 + iconShiftX; int y_trang = y - 4; // Căn giữa chuẩn
      display.drawBitmap(x_trang, y_trang, icon_Trang_Khuyet_Nho, 8, 8, SSD1306_WHITE); 
      if ((currentMillis / 500) % 2 == 0) {
        display.drawPixel(x_trang - 4, y_trang - 2, SSD1306_WHITE); 
        display.drawPixel(x_trang + 10, y_trang + 6, SSD1306_WHITE); 
        display.drawPixel(x_trang + 3, y_trang + 9, SSD1306_WHITE); 
      }
    } else {
      display.fillCircle(x, y, 3, SSD1306_WHITE); 
      int d = 5 + ((currentMillis / 200) % 2) * 2;
      display.drawPixel(x, y - d, SSD1306_WHITE);   display.drawPixel(x, y + d, SSD1306_WHITE);
      display.drawPixel(x - d, y, SSD1306_WHITE);   display.drawPixel(x + d, y, SSD1306_WHITE);
      display.drawPixel(x - (d-1), y - (d-1), SSD1306_WHITE); display.drawPixel(x + (d-1), y - (d-1), SSD1306_WHITE);
      display.drawPixel(x - (d-1), y + (d-1), SSD1306_WHITE); display.drawPixel(x + (d-1), y + (d-1), SSD1306_WHITE);
    }
  }
  // --- TRẠNG THÁI MÂY (ĐÊM: BỎ TRĂNG - MÂY TRÔI ĐÈ SAO RANDOM) ---
  else if (trangThaiThoiTiet == "MAY") {
    if (checkDem) {
      // Cứ sau mỗi 1.4 giây (sau khi sao tắt và bật lại), đổi vị trí ngẫu nhiên cho sinh động
      if (currentMillis - lastStarRandomTime >= 1400) {
        lastStarRandomTime = currentMillis;
        starRandX1 = random(2, 14);  // Random trong khu vực nửa trái vùng quét dọc
        starRandY1 = random(98, 108);
        starRandX2 = random(18, 30); // Random trong khu vực nửa phải vùng quét dọc
        starRandY2 = random(110, 122);
      }

      // Nhịp hiển thị nhấp nháy cho sao
      if ((currentMillis / 700) % 2 == 0) {
        display.drawPixel(starRandX1 + iconShiftX, starRandY1 + iconShiftY, SSD1306_WHITE);  
        display.drawPixel(starRandX2 + iconShiftX, starRandY2 + iconShiftY, SSD1306_WHITE); 
      }
      // Vẽ mây đè lên sao
      display.drawBitmap(X_may1 + iconShiftX, y - 6, icon_May_Pixel_Theo_Anh, 16, 11, SSD1306_WHITE);
      display.drawBitmap(X_may2 + iconShiftX, y + 2, icon_May_Pixel_Theo_Anh, 16, 11, SSD1306_WHITE);
    } else {
      display.drawBitmap(X_may1 + iconShiftX, y - 6, icon_May_Pixel_Theo_Anh, 16, 11, SSD1306_WHITE);
      display.drawBitmap(X_may2 + iconShiftX, y + 2, icon_May_Pixel_Theo_Anh, 16, 11, SSD1306_WHITE);
    }
  }
  else if (trangThaiThoiTiet == "MUA") {
    display.drawBitmap(x - 8, y - 6, icon_May_Pixel_Theo_Anh, 16, 11, SSD1306_WHITE);
    int quat_roi = (currentMillis / 80) % 6; 
    display.drawPixel(x - 5, y + 6 + quat_roi, SSD1306_WHITE);
    display.drawPixel(x - 1, y + 7 + (quat_roi % 4), SSD1306_WHITE);
    display.drawPixel(x + 3, y + 5 + quat_roi, SSD1306_WHITE);
  }
  else if (trangThaiThoiTiet == "DONG_SET") {
    display.drawBitmap(x - 8, y - 5, icon_May_Pixel_Theo_Anh, 16, 11, SSD1306_WHITE);
    if ((currentMillis / 120) % 3 == 0) display.drawBitmap(x - 4, y + 6, icon_Set_Cai_Tien, 8, 12, SSD1306_WHITE); 
  }
}

void setup() {
  Serial.begin(115200); Wire.begin(8, 9); Wire.setTimeout(50); randomSeed(esp_random());
  ledcAttach(LED_PIN_5, 5000, 8); ledcAttach(LED_PIN_4, 5000, 8); ledcAttach(LED_PIN_6, 5000, 8);
  ledcWrite(LED_PIN_5, 0); ledcWrite(LED_PIN_4, 0); ledcWrite(LED_PIN_6, 0);  

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { for (;;); }
  display.clearDisplay(); display.setRotation(1); display.setTextColor(SSD1306_WHITE);

  display.setCursor(1, 10);  display.print(F("SMART"));
  display.setCursor(1, 25);  display.print(F("SAT"));      
  display.setCursor(1, 40);  display.print(F("SYS"));      
  display.setCursor(1, 55);  display.print(F("v12.2"));    
  display.drawFastHLine(0, 75, 32, SSD1306_WHITE);
  display.setCursor(1, 85);  display.print(F("READY"));     
  display.display(); delay(1500);

  if (sht31.begin(0x44)) isSensorOnline = true; else isSensorOnline = false;
  wm.setConfigPortalBlocking(false); wm.setConfigPortalTimeout(60); wm.setConnectTimeout(150);
  WiFi.mode(WIFI_STA); WiFi.begin(); lastSyncWiFi = millis();
}

void loop() {
  if (WiFi.getMode() != WIFI_OFF) wm.process();
  unsigned long now = millis();

  if (now - lastScrollTimeMay >= 190) {
    lastScrollTimeMay = now; 
    X_may1++; if (X_may1 > 32) X_may1 = -16; 
    X_may2++; if (X_may2 > 32) X_may2 = -16; 
  }

  if (!hasFirstSyncEver && (WiFi.status() != WL_CONNECTED) && (now > 5000) && (WiFi.getMode() != WIFI_OFF) && (!wm.getConfigPortalActive())) {
    if (!portalStarted) { wm.startConfigPortal("Smart_Satellite"); portalStarted = true; }
  }
  if (!hasFirstSyncEver && (WiFi.status() == WL_CONNECTED)) trySyncTime();
  if (hasFirstSyncEver && !reconnectingWiFi && (now - lastSyncWiFi > syncInterval)) { reconnectingWiFi = true; reconnectStart = now; WiFi.mode(WIFI_STA); WiFi.begin(); }

  if (reconnectingWiFi) {
    if (now - reconnectStart > 12000) { reconnectingWiFi = false; lastSyncWiFi = now; WiFi.disconnect(false); WiFi.mode(WIFI_OFF); }
    if (WiFi.status() == WL_CONNECTED) { trySyncTime(); reconnectingWiFi = false; }
  }

  struct tm timeinfo; char hStr[3] = "--", mStr[3] = "--", sStr[3] = "--"; char dayMonthStr[6] = "--/--", yearStr[5] = "----";
  if (hasFirstSyncEver && getLocalTime(&timeinfo)) { 
    gioHienTaiHeThong = timeinfo.tm_hour;
    snprintf(hStr, sizeof(hStr), "%02d", timeinfo.tm_hour); snprintf(mStr, sizeof(mStr), "%02d", timeinfo.tm_min); snprintf(sStr, sizeof(sStr), "%02d", timeinfo.tm_sec);
    snprintf(dayMonthStr, sizeof(dayMonthStr), "%02d/%02d", timeinfo.tm_mday, timeinfo.tm_mon + 1); snprintf(yearStr, sizeof(yearStr), "%04d", timeinfo.tm_year + 1900);
  }
  
  if (hasFirstSyncEver) {
    if (gioHienTaiHeThong >= 6 && gioHienTaiHeThong < 19) { laBanDem = false; ledBrightness = LED_DAY_BRIGHTNESS; } 
    else { laBanDem = true; ledBrightness = LED_NIGHT_BRIGHTNESS; }
  }

  bool sensorReadDone = false;
  if (now - lastSensorRead > 2000) {
    rawTemp = sht31.readTemperature(); rawHum = sht31.readHumidity(); lastSensorRead = now; sensorReadDone = true;
    if (!isnan(rawTemp) && !isnan(rawHum)) { 
      isSensorOnline = true;
      if (isnan(filteredTemp)) filteredTemp = rawTemp; if (isnan(filteredHum)) filteredHum = rawHum;
      filteredTemp = (filteredTemp * (1.0 - alpha)) + (rawTemp * alpha); filteredHum = (filteredHum * (1.0 - alpha)) + (rawHum * alpha);
    } else { isSensorOnline = false; }
  }
  if (sensorReadDone && !isSensorOnline && (now - lastSensorRetry > 10000)) { recoverI2CBus(); if (sht31.begin(0x44)) isSensorOnline = true; lastSensorRetry = now; }

  bool dangChayCheDoOffline = (!hasFirstSyncEver || (now - thoiGianCapNhatMangCuoi > timeoutMatMang));
  if (dangChayCheDoOffline) {
    if (isSensorOnline && !isnan(rawTemp) && !isnan(rawHum)) {
      if (rawHum > 85.0 || (rawHum > 80.0 && rawHum <= 85.0 && rawTemp < 26.0)) trangThaiThoiTiet = "MUA";
      else if (rawHum >= 60.0 && rawHum <= 85.0) trangThaiThoiTiet = "MAY"; else trangThaiThoiTiet = "NANG";
    } else { trangThaiThoiTiet = "NANG"; }
  }

  unsigned long pulseModulo = now % 15000; bool blinkPhase = (now % 600) < 300; bool isWithinTenSec = (reconnectingWiFi && (now - reconnectStart <= 12000));

  static unsigned long lastDisplayRender = 0;
  if (now - lastDisplayRender >= 33) {
    lastDisplayRender = now; display.clearDisplay(); display.setTextSize(1); display.setTextWrap(false); 

    static bool oledDimmed = false; bool shouldDim = hasFirstSyncEver && (gioHienTaiHeThong >= 23 || gioHienTaiHeThong <= 5);
    if (shouldDim != oledDimmed) { display.dim(shouldDim); oledDimmed = shouldDim; }

    if (now - lastShiftTime > 30000) { 
      burnShiftX = random(-1, 2); burnShiftY = random(-1, 2); textShiftX = random(0, 2); 
      iconShiftX = random(-1, 2); iconShiftY = random(-1, 2); lastShiftTime = now;
    }

    drawCyberLaserLines(19, 95); 

    // --- TẦNG 1: HEADER ---
    if (!hasFirstSyncEver || isWithinTenSec) {
      if (blinkPhase) { display.setCursor(4 + textShiftX, 5); display.print(hasFirstSyncEver ? F("SYNC") : F("WAIT")); }
    } else {
      if (hasFirstSyncEver) {
        if (pulseModulo < 6000) { display.setCursor(1 + textShiftX, 0); display.print(dayMonthStr); display.setCursor(4 + textShiftX, 10); display.print(yearStr); } 
        else if (pulseModulo >= 6000 && pulseModulo < 11000) { char bufferShortDay[4]; strcpy_P(bufferShortDay, (char*)pgm_read_ptr(&(daysOfWeekShort[timeinfo.tm_wday]))); display.setCursor(7 + textShiftX, 5); display.print(bufferShortDay); } 
        else if (pulseModulo >= 11000 && pulseModulo < 15000) { char bufferShortMonth[4]; strcpy_P(bufferShortMonth, (char*)pgm_read_ptr(&(monthsShort[timeinfo.tm_mon]))); display.setCursor(7 + textShiftX, 5); display.print(bufferShortMonth); }
      }
    }

    // --- TẦNG 2: CORE CLOCK & CHỮ CHẠY TĨNH IN HOA ---
    if (hasFirstSyncEver) {
      display.setTextSize(2); display.setCursor(4 + burnShiftX, 28 + burnShiftY); display.print(hStr); display.setCursor(4 + burnShiftX, 49 + burnShiftY); display.print(mStr);
      display.setTextSize(1); display.setCursor(4 + burnShiftX, 71 + burnShiftY); display.print(sStr); display.print(F("s")); 

      if (dangChayCheDoOffline) { chuoiChayTongHop = F("SENSOR SHT3X"); } 
      else { chuoiChayTongHop = chuoiChayOwmDungSan; } 

      int doRongChuoi = chuoiChayTongHop.length() * 6;
      if (dangChoNghi20s) { if (now - waitStartTime >= 20000) { dangChoNghi20s = false; X_chuChay = 32; } } 
      else { if (now - lastScrollTime >= 80) { lastScrollTime = now; X_chuChay--; if (X_chuChay <= -doRongChuoi) { dangChoNghi20s = true; waitStartTime = now; } } }
      if (!dangChoNghi20s) { display.setCursor(X_chuChay + burnShiftX, 85 + burnShiftY); display.print(chuoiChayTongHop); }
    } else {
      display.setTextSize(1);
      if (blinkPhase) { display.setCursor(4 + burnShiftX, 35 + burnShiftY); display.print(F("WAIT")); display.setCursor(1 + burnShiftX, 50 + burnShiftY); display.print(F("-ING")); display.setCursor(-2 + burnShiftX, 69 + burnShiftY); display.print(F("EARTH")); }
    }

    // --- TẦNG 3: METEO FOOTER ---
    veChuyenTrangIconFooter(16, 108, laBanDem, now);

    display.display();
  }

  // --- LED HARDWARE PROCESS ---
  unsigned long ledCycle = now % 6000;
  if (ledCycle < 50) { ledcWrite(LED_PIN_5, ledBrightness); ledcWrite(LED_PIN_4, 0); ledcWrite(LED_PIN_6, 0); } 
  else if (ledCycle >= 120 && ledCycle < 170) { ledcWrite(LED_PIN_5, ledBrightness); ledcWrite(LED_PIN_4, 0); ledcWrite(LED_PIN_6, 0); } 
  else if (ledCycle >= 300 && ledCycle < 350) { ledcWrite(LED_PIN_5, 0); ledcWrite(LED_PIN_4, ledBrightness); ledcWrite(LED_PIN_6, 0); } 
  else if (ledCycle >= 420 && ledCycle < 470) { ledcWrite(LED_PIN_5, 0); ledcWrite(LED_PIN_4, ledBrightness); ledcWrite(LED_PIN_6, 0); } 
  else if (ledCycle >= 600 && ledCycle < 650) { ledcWrite(LED_PIN_5, 0); ledcWrite(LED_PIN_4, 0); ledcWrite(LED_PIN_6, ledBrightness); } 
  else if (ledCycle >= 720 && ledCycle < 770) { ledcWrite(LED_PIN_5, 0); ledcWrite(LED_PIN_4, 0); ledcWrite(LED_PIN_6, ledBrightness); } 
  else { ledcWrite(LED_PIN_5, 0); ledcWrite(LED_PIN_4, 0); ledcWrite(LED_PIN_6, 0); }
  delay(10);
}
