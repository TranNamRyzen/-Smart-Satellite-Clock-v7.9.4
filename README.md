# 🛰️ Smart Satellite Clock v7.9.4

Open-frame IoT Weather Clock using ESP32 / OLED SSD1306 / SHT31

---

## 👤 Project Info

- **Project Owner:** Trần Nam  
- **Hardware Source Core:** HuyVector (YouTube)  
- **Firmware Expansion:** AI Collaborators (Gemini + ChatGPT)

## 🧠 Overview

Smart Satellite Clock is an **open-frame IoT weather clock system** built with ESP32.

It combines:
- Real-time clock (NTP + offline fallback)
- Environmental sensing (temperature + humidity)
- Pixel-art weather rendering
- Dual LED mechanical pulse system
- OLED animated UI (flip-page style)

Designed in an **open hardware aesthetic (khung thau hở)**.

---

## ⚙️ Hardware

- ESP32 / ESP8266
- OLED SSD1306 (I2C 128x32)
- SHT31 Temperature & Humidity Sensor
- Dual LED output (PWM control)
- Open-frame copper/brass structure

---

## 🌐 1. WiFi Core (WiFiManager)

- Auto AP mode: `Smart_Satellite`
- Captive portal setup via phone
- Auto reconnect system
- Auto WiFi shutdown after sync (power saving)

---

## ⏰ 2. Time Core (NTP Hybrid)

- Sync via `pool.ntp.org`
- Offline time tracking using `millis()`
- Periodic resync for accuracy

---

## 🧩 3. Display Engine (OLED UI System)

- OLED SSD1306 rotated 90°
- 3-page cyclic interface (15s loop):

| Page | Time | Content |
|------|------|--------|
| 1 | 0–7s | Date (DD/MM/YYYY) |
| 2 | 7–11s | Weekday (MON–SUN) |
| 3 | 11–15s | Month + Weather |

- Line flip animation (100ms)
- Anti burn-in: ±1px random UI shift

---

## 🌡 4. Sensor Core (SHT31)

- Temperature + Humidity reading (2s interval)
- Exponential smoothing filter
- I2C bus recovery system
- Sensor online/offline detection

---

## 🌦 5. Weather Engine (Pixel Art)

Logic:

- 🌧 Rain:
  - humidity > 75%
  - OR (humidity > 65% AND temp < 27°C)

- ☁ Cloud:
  - humidity 60% → 80%

- ☀ Sun:
  - humidity < 60%

📍 Display:
- Shown in Page 3 footer
- Fallback: "?" if sensor error

---

## 💡 6. LED Pulse Engine

- Dual LED alternating PWM
- Mechanical blink effect (4.5s cycle)

Modes:
- 🌞 Day: 06:00 – 19:00 → brightness = 10  
- 🌙 Night: 19:00 – 06:00 → brightness = 5 (eco mode)

---

## 🔧 7. System Philosophy

- Stability > Visual effects
- Self-healing (WiFi / I2C recovery)
- Minimal hardware dependency
- Open-frame mechanical design preserved
- Modular firmware architecture

---
## 📌 Notes

This project is continuously evolving.
Core hardware design belongs to **HuyVector inspiration**, firmware extended by community collaboration.

---
