
 * SMART SATELLITE CLOCK v7.9.4 – OPEN-FRAME IoT SYSTEM (ESP32 / OLED / SHT31)
 * =============================================================================================
 *
 * 👤 QUẢN LÝ DỰ ÁN      : Trần Nam
 * 🔩 SOURCE CORE         : HuyVector (YouTube – nền tảng phần cứng & cảm hứng thiết kế)
 * 🤖 PHÁT TRIỂN MỞ RỘNG  : AI Collaborators (Gemini + ChatGPT – hỗ trợ tối ưu & hậu kiểm code)
 *
 * =============================================================================================
 * 🧠 TỔNG QUAN HỆ THỐNG
 * - Đồng hồ IoT khung thau hở mô phỏng vệ tinh mini
 * - ESP32C3 SUPER MINI + OLED SSD1306 (I2C)
 * - Cảm biến SHT31 đo nhiệt độ / độ ẩm
 * - Kiến trúc open-frame: ưu tiên cơ khí, tản nhiệt và dễ bảo trì
 *
 * =============================================================================================
 * ⚙️ 1. WIFI CORE (WiFiManager SYSTEM)
 * - Auto AP: "Smart_Satellite" khi mất kết nối
 * - Captive portal cấu hình bằng điện thoại
 * - Tự reconnect định kỳ
 * - Tự tắt WiFi sau khi sync để tiết kiệm năng lượng
 *
 * =============================================================================================
 * ⏰ 2. TIME CORE (NTP HYBRID SYSTEM)
 * - Đồng bộ thời gian qua NTP (pool.ntp.org)
 * - Duy trì thời gian offline bằng millis()
 * - Chu kỳ sync định kỳ để tránh lệch giờ
 *
 * =============================================================================================
 * 🧩 3. DISPLAY ENGINE (OLED UI SYSTEM)
 * - OLED SSD1306 xoay dọc 90°
 * - 3 trang hiển thị luân phiên 15 giây:
 *   + Page 1 (0–7s)  : Ngày / Tháng / Năm
 *   + Page 2 (7–11s) : Thứ (MON, TUE...)
 *   + Page 3 (11–15s): Tháng (JAN, FEB...)
 * - Line Flip 100ms khi chuyển trang (giả lập cơ khí)
 * - Anti burn-in: dịch UI ±1px theo chu kỳ
 *
 * =============================================================================================
 * 🌡 4. SENSOR CORE (SHT31 SYSTEM)
 * - Đọc nhiệt độ / độ ẩm mỗi 2 giây
 * - Lọc nhiễu bằng exponential smoothing
 * - Có cơ chế recover I2C khi bus treo
 * - Tự đánh dấu sensor online/offline
 *
 * =============================================================================================
 * 🌦 5. WEATHER ENGINE (PIXEL ART ICON SYSTEM)
 * - Rain  : humidity > 75% OR (humidity > 65% AND temp < 27°C)
 * - Cloud : 60% → 80%
 * - Sun   : < 60%
 *
 * - Hiển thị icon pixel art ở footer Page 3
 * - Khi lỗi sensor: hiển thị “?”
 *
 * =============================================================================================
 * 💡 6. LED PULSE ENGINE (DUAL PWM SYSTEM)
 * - LED nháy kép luân phiên 2 chân
 * - Chu kỳ cơ học: 4.5 giây
 * - Day mode   : 06:00 → 19:00 → brightness = 10
 * - Night mode : 19:00 → 06:00 → brightness = 5 (eco mode)
 *
 * =============================================================================================
 * 🔧 7. SYSTEM PRINCIPLE
 * - Ưu tiên ổn định hơn hiệu ứng
 * - Tự phục hồi khi lỗi WiFi / I2C
 * - Giữ kiến trúc open-frame hardware nguyên bản
 * - Mở rộng module nhưng không phá core logic
 *
 * =============================================================================================
 */

#include <Adafruit_GFX.h>       
#include <Adafruit_SSD1306.h>   
#include <Adafruit_SHT31.h>     
#include <WebServer.h>          
#include <DNSServer.h>          
#include <WiFiManager.h>        
#include <time.h>               

// --- CẤU HÌNH PHẦN CỨNG MÀN HÌNH & CẢM BIẾN ---
#define SCREEN_WIDTH 128        
#define SCREEN_HEIGHT 32        
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
Adafruit_SHT31 sht31 = Adafruit_SHT31();

// --- KHAI BÁO BIẾN HỆ THỐNG CẢM BIẾN ---
float filteredTemp = NAN, filteredHum = NAN;        
const float alpha = 0.1;        
unsigned long lastSensorRetry = 0, lastSensorRead = 0;   
bool isSensorOnline = false;    

// --- ĐỊNH NGHĨA CHÂN NGOẠI VI VÀ ĐỘ SÁNG LED ---
#define I2C_SDA 8               
#define I2C_SCL 9               
#define LED_PIN_5 5             
#define LED_PIN_4 4             
#define LED_DAY_BRIGHTNESS   10  // Ban Ngày 6h-19h
#define LED_NIGHT_BRIGHTNESS 5 // Ban Đêm 19h-6h hôm sau
uint8_t ledBrightness = 10;      // Giữ nguyên độ sáng dịu hiu hiu dọc khung thau

// --- BIẾN QUẢN LÝ ĐỒNG BỘ THỜI GIAN & WIFI ---
unsigned long lastSyncWiFi = 0;            
const unsigned long syncInterval = 3600000; // [ĐÃ ĐỔI] Đúng 1 TIẾNG đồng bộ lại 1 lần
bool hasFirstSyncEver = false;             
bool wifiConnectError = false;             
bool reconnectingWiFi = false;             
unsigned long reconnectStart = 0;          

WiFiManager wm;

// --- MẢNG LƯU CHUỖI KÝ TỰ TRÊN BỘ NHỚ FLASH (PROGMEM) ---
const char* const daysOfWeekShort[] PROGMEM = {"SUN", "MON", "TUE", "WED", "THU", "FRI", "SAT"};
const char* const monthsShort[] PROGMEM = {"JAN", "FEB", "MAR", "APR", "MAY", "JUN", "JUL", "AUG", "SEP", "OCT", "NOV", "DEC"};

// --- BIẾN CHỐNG CHÁY HÌNH (ANTI-BURN-IN) & ĐỔI TRANG ---
static int8_t burnShiftX = 0, burnShiftY = 0; 
static int8_t textShiftX = 0;                  
static unsigned long lastShiftTime = 0;
static int lastActivePage = 0;

// --- ENGINE ĐỒ HỌA ICON PIXEL ART SẮC NÉT ---
// Mặt trời
void drawIconSunny(int x, int y) {
  display.drawCircle(x + 8, y + 6, 4, SSD1306_WHITE);

  // Tia dọc
  display.drawFastVLine(x + 8, y + 0, 2, SSD1306_WHITE);
  display.drawFastVLine(x + 8, y + 10, 2, SSD1306_WHITE);

  // Tia ngang
  display.drawFastHLine(x + 2, y + 6, 2, SSD1306_WHITE);
  display.drawFastHLine(x + 12, y + 6, 2, SSD1306_WHITE);

  // Tia chéo
  display.drawPixel(x + 4,  y + 2, SSD1306_WHITE);
  display.drawPixel(x + 12, y + 2, SSD1306_WHITE);
  display.drawPixel(x + 4,  y + 10, SSD1306_WHITE);
  display.drawPixel(x + 12, y + 10, SSD1306_WHITE);
}
 // Mây
void drawIconCloudy(int x, int y) {
  display.fillCircle(x + 15, y + 3, 3, SSD1306_WHITE);
  display.fillCircle(x + 19, y + 4, 2, SSD1306_WHITE);
  display.fillCircle(x + 11, y + 5, 2, SSD1306_WHITE);
  display.fillRect(x + 11, y + 4, 9, 3, SSD1306_WHITE);
  display.drawCircle(x + 5, y + 7, 3, SSD1306_WHITE);
  display.drawCircle(x + 9, y + 6, 3, SSD1306_WHITE);
  display.drawCircle(x + 12, y + 8, 2, SSD1306_WHITE);
  display.drawCircle(x + 2, y + 8, 2, SSD1306_WHITE);
  display.fillRect(x + 3, y + 7, 9, 3, SSD1306_BLACK);     
  display.drawFastHLine(x + 2, y + 10, 11, SSD1306_WHITE);  
}
// Mưa
void drawIconRainy(int x, int y) {
  drawIconCloudy(x, y);

  display.drawLine(x + 2,  y + 12, x + 1,  y + 15, SSD1306_WHITE);
  display.drawLine(x + 6,  y + 12, x + 5,  y + 15, SSD1306_WHITE);
  display.drawLine(x + 10, y + 12, x + 9,  y + 15, SSD1306_WHITE);
  display.drawLine(x + 14, y + 12, x + 13, y + 15, SSD1306_WHITE);
}

void drawWeatherIconFooter(int x, int y, float temp, float humidity) {
  if (isnan(humidity) || isnan(temp) || !isSensorOnline) { 
    display.setTextSize(1); display.setCursor(x + 5, y + 4); display.print(F("?")); return;
  }
  if (humidity > 80.0 || (humidity > 75.0 && temp < 27.0)) drawIconRainy(x, y); 
  else if (humidity >= 60.0 && humidity <= 80.0) drawIconCloudy(x, y);        
  else drawIconSunny(x, y);                                                   
}

// --- HÀM CỨU HỘ KHÔI PHỤC BUS I2C ---
void recoverI2CBus() {
  Wire.end(); delay(5);
  pinMode(I2C_SCL, OUTPUT); pinMode(I2C_SDA, INPUT_PULLUP); delayMicroseconds(2);
  for (int i = 0; i < 9; i++) { 
    digitalWrite(I2C_SCL, HIGH); delayMicroseconds(5);
    digitalWrite(I2C_SCL, LOW);  delayMicroseconds(5);
  }
  Wire.begin(I2C_SDA, I2C_SCL); Wire.setClock(400000); delay(5);
}

// --- HÀM ĐỒNG BỘ GIỜ NTP TỐC ĐỘ CAO ---
void trySyncTime() {
  if (WiFi.status() == WL_CONNECTED) {
    configTime(7 * 3600, 0, "asia.pool.ntp.org", "time.google.com");
    struct tm timeinfo;
    if (getLocalTime(&timeinfo, 1500)) { 
      hasFirstSyncEver = true; 
      wifiConnectError = false; 
      lastSyncWiFi = millis();      
      
      WiFi.disconnect(false);
      WiFi.mode(WIFI_OFF);
    }
  }
}

// =============================================================================================
//                                  CẤU HÌNH PHẦN CỨNG BAN ĐẦU (SETUP)
// =============================================================================================
void setup() {
  Serial.begin(115200); 
  Wire.begin(I2C_SDA, I2C_SCL); Wire.setClock(400000); Wire.setTimeOut(50);                 
  
  randomSeed(analogRead(0)); 

  // Cấu hình băm xung PWM phần cứng cho cặp LED tín hiệu
  ledcAttach(LED_PIN_5, 5000, 8); 
  ledcAttach(LED_PIN_4, 5000, 8); 
  ledcWrite(LED_PIN_5, 0); 
  ledcWrite(LED_PIN_4, 0); 

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { for (;;); } 
  display.clearDisplay(); display.setRotation(1); display.setTextColor(SSD1306_WHITE); 

  // Màn hình khởi động chỉn chu tinh gọn theo layout đứng
  display.setCursor(1, 10);  display.print(F("SMART"));
  display.setCursor(1, 25);  display.print(F("SAT"));       
  display.setCursor(1, 40);  display.print(F("SYS"));       
  display.setCursor(1, 55);  display.print(F("v7.9.8"));    
  display.drawFastHLine(0, 75, 32, SSD1306_WHITE);
  display.setCursor(1, 85);  display.print(F("READY"));     
  display.display(); delay(1500); 

  if (sht31.begin(0x44)) isSensorOnline = true; else isSensorOnline = false;

  wm.setConfigPortalBlocking(false); 
  wm.setConfigPortalTimeout(60);     
  wm.setConnectTimeout(150);         // [ĐÃ ĐỔI] Chốt hạ 150 giây cực kỳ chắc ăn cho bác Nam
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(); 
  lastSyncWiFi = millis();
}

// =============================================================================================
//                                VÒNG LẶP HỆ THỐNG TUẦN HOÀN (LOOP)
// =============================================================================================
void loop() {
  if (WiFi.getMode() != WIFI_OFF) {
    wm.process(); 
  }
  
  unsigned long now = millis(); 

  if (!hasFirstSyncEver && (WiFi.status() != WL_CONNECTED) && (now > 5000) && (WiFi.getMode() != WIFI_OFF) && (!wm.getConfigPortalActive())) {
    wm.startConfigPortal("Smart_Satellite");
  }

  if (!hasFirstSyncEver && (WiFi.status() == WL_CONNECTED)) {
    trySyncTime();
  }

  if (hasFirstSyncEver && !reconnectingWiFi && (now - lastSyncWiFi > syncInterval)) {
    reconnectingWiFi = true; 
    reconnectStart = now; 
    WiFi.mode(WIFI_STA); 
    WiFi.begin();
  }

  if (reconnectingWiFi) {
    if (now - reconnectStart > 12000) { 
      reconnectingWiFi = false; 
      lastSyncWiFi = now; 
      WiFi.disconnect(false); 
      WiFi.mode(WIFI_OFF); 
    }
    if (WiFi.status() == WL_CONNECTED) {
      trySyncTime(); 
      reconnectingWiFi = false;
    }
  }

  struct tm timeinfo;
  char hStr[3] = "--", mStr[3] = "--", sStr[3] = "--";
  char dayMonthStr[6] = "--/--", yearStr[5] = "----";
  int currentHour = 0;

  if (hasFirstSyncEver && getLocalTime(&timeinfo, 10)) { 
    currentHour = timeinfo.tm_hour;
    snprintf(hStr, sizeof(hStr), "%02d", timeinfo.tm_hour);
    snprintf(mStr, sizeof(mStr), "%02d", timeinfo.tm_min);
    snprintf(sStr, sizeof(sStr), "%02d", timeinfo.tm_sec);
    snprintf(dayMonthStr, sizeof(dayMonthStr), "%02d/%02d", timeinfo.tm_mday, timeinfo.tm_mon + 1); 
    snprintf(yearStr, sizeof(yearStr), "%04d", timeinfo.tm_year + 1900);
  }
if (hasFirstSyncEver) {
  if (currentHour >= 6 && currentHour < 19) {
    ledBrightness = LED_DAY_BRIGHTNESS;
  } else {
    ledBrightness = LED_NIGHT_BRIGHTNESS;
  }
}
  float rawTemp = NAN, rawHum  = NAN; bool sensorReadDone = false; 
  if (now - lastSensorRead > 2000) {
    rawTemp = sht31.readTemperature(); rawHum = sht31.readHumidity(); lastSensorRead = now; sensorReadDone = true; 
    if (!isnan(rawTemp) && !isnan(rawHum)) {
      isSensorOnline = true;
      if (isnan(filteredTemp)) filteredTemp = rawTemp; 
      if (isnan(filteredHum))  filteredHum  = rawHum;  
      filteredTemp = (filteredTemp * (1.0 - alpha)) + (rawTemp * alpha);
      filteredHum  = (filteredHum * (1.0 - alpha)) + (rawHum * alpha);
    } else { isSensorOnline = false; }
  }
  
  if (sensorReadDone && !isSensorOnline) {
    if (now - lastSensorRetry > 10000) { 
      recoverI2CBus(); if (sht31.begin(0x44)) isSensorOnline = true; lastSensorRetry = now; 
    }
  }

  unsigned long pulseModulo = now % 15000; 
  int activePage = 0; 
  if (pulseModulo < 7000) activePage = 0;                             
  else if (pulseModulo >= 7000 && pulseModulo < 11000) activePage = 1; 
  else activePage = 2;                                                

  bool isPageFlipping = false; static unsigned long flipStartTime = 0;
  
  if (activePage != lastActivePage) { 
    isPageFlipping = true; 
    flipStartTime = now; 
    lastActivePage = activePage; 
  }
  
  if (now - flipStartTime < 100) isPageFlipping = true; 

  bool blinkPhase = (now % 600) < 300; 
  bool isWithinTenSec = (reconnectingWiFi && (now - reconnectStart <= 12000)); 

  static unsigned long lastDisplayRender = 0;
  if (now - lastDisplayRender >= 33) {
    lastDisplayRender = now; display.clearDisplay(); display.setTextSize(1); 

    static bool oledDimmed = false;
    bool shouldDim = hasFirstSyncEver && (currentHour >= 23 || currentHour <= 5);
    if (shouldDim != oledDimmed) { display.dim(shouldDim); oledDimmed = shouldDim; }

    if (now - lastShiftTime > 30000) { 
      burnShiftX = random(-1, 2); burnShiftY = random(-1, 2); textShiftX = random(0, 2); lastShiftTime = now;
    }

    if (wifiConnectError) {
      display.setCursor(4 + textShiftX, 0); display.print(F("CONN")); display.setCursor(7 + textShiftX, 10); display.print(F("ERR"));
    } 
    else if (!hasFirstSyncEver || isWithinTenSec) {
      if (blinkPhase) { display.setCursor(4 + textShiftX, 5); display.print(hasFirstSyncEver ? F("SYNC") : F("WAIT")); }
    } 
    else {
      if (hasFirstSyncEver) {
        if (activePage == 0) {
          display.setCursor(1 + textShiftX, 0); display.print(dayMonthStr); display.setCursor(4 + textShiftX, 10); display.print(yearStr); 
        } else if (activePage == 1) {
          char bufferShortDay[4]; strcpy_P(bufferShortDay, (char*)pgm_read_ptr(&(daysOfWeekShort[timeinfo.tm_wday])));
          display.setCursor(7 + textShiftX, 5); display.print(bufferShortDay); 
        } else if (activePage == 2) {
          char bufferShortMonth[4]; strcpy_P(bufferShortMonth, (char*)pgm_read_ptr(&(monthsShort[timeinfo.tm_mon])));
          display.setCursor(7 + textShiftX, 5); display.print(bufferShortMonth);
        }
      }
    }

    if (!isPageFlipping) display.drawFastHLine(0, 19, 32, SSD1306_WHITE); 

    if (hasFirstSyncEver) {
      // [ĐÃ TINH CHỈNH] Hạ thấp tọa độ Y của khối Giờ, Phút, Giây để tạo khoảng thoáng so với gạch trên
      display.setTextSize(2); 
      display.setCursor(4 + burnShiftX, 28 + burnShiftY); display.print(hStr);
      display.setCursor(4 + burnShiftX, 49 + burnShiftY); display.print(mStr);
      display.setTextSize(1); 
      display.setCursor(4 + burnShiftX, 71 + burnShiftY); display.print(sStr); display.print(F("s")); // Thêm chữ 's' tĩnh
    } else {
      display.setTextSize(1); 
      if (blinkPhase) { 
        display.setCursor(4 + burnShiftX, 35 + burnShiftY); display.print(F("WAIT")); 
        display.setCursor(1 + burnShiftX, 50 + burnShiftY); display.print(F("-ING")); 
        display.setCursor(-2 + burnShiftX, 69 + burnShiftY); display.print(F("EARTH")); 
      }
    }

    if (!isPageFlipping) display.drawFastHLine(0, 95, 32, SSD1306_WHITE); 

    display.setTextSize(1);
    if (activePage == 0 || activePage == 1) {
      if (isSensorOnline && !isnan(filteredTemp) && !isnan(filteredHum)) {
        display.setCursor(1 + textShiftX, 102); display.print((int)filteredTemp); display.print((char)247); display.print(F("C")); 
        display.setCursor(0 + textShiftX, 116); display.print(F("H:")); display.print((int)filteredHum);  display.print(F("%")); 
      } else {
        display.setCursor(4 + textShiftX, 102); display.print(F("--")); display.print((char)247); display.print(F("C")); 
        display.setCursor(0 + textShiftX, 116); display.print(F("H:--%")); 
      }
    } else if (activePage == 2) {
      drawWeatherIconFooter(4 + textShiftX, 102, filteredTemp, filteredHum); 
    }

    display.display(); 
  }

  // Cụm điều khiển nhấp nháy đuổi kép LED cơ học
  unsigned long ledCycle = now % 4500; 
 if (ledCycle < 45) {
  ledcWrite(LED_PIN_5, ledBrightness);
  ledcWrite(LED_PIN_4, 0);
}
else if (ledCycle >= 100 && ledCycle < 145) {
  ledcWrite(LED_PIN_5, ledBrightness);
  ledcWrite(LED_PIN_4, 0);
}
else if (ledCycle >= 265 && ledCycle < 310) {
  ledcWrite(LED_PIN_5, 0);
  ledcWrite(LED_PIN_4, ledBrightness);
}
else if (ledCycle >= 365 && ledCycle < 410) {
  ledcWrite(LED_PIN_5, 0);
  ledcWrite(LED_PIN_4, ledBrightness);
}
else {
  ledcWrite(LED_PIN_5, 0);
  ledcWrite(LED_PIN_4, 0);
}
  delay(10); // Nhịp thở FreeRTOS nuôi luồng WiFi ngầm ổn định
}
