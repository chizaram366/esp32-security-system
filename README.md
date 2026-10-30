# ESP32 RFID & Laser Security System

A compact IoT security system built with an ESP32 microcontroller that integrates RFID authentication, laser tripwires, a local LCD status display, and real-time Telegram bot notifications[span_1](start_span)[span_1](end_span).

## Features
* **RFID Access Control:** Arm and disarm the system using authorized MFRC522 RFID key cards[span_2](start_span)[span_2](end_span).
* **Dual Laser Tripwires:** Dual-beam breach detection monitoring subtle changes in light intensity using analog photo-resistor panels[span_3](start_span)[span_3](end_span).
* **Telegram Bot Integration:** Remotely check status, arm (`/arm`), or disarm (`/disarm`) the system over Wi-Fi, with real-time intrusion alerts sent directly to your phone[span_4](start_span)[span_4](end_span).
* **Local Feedback:** Visual feedback using an I2C LCD display, buzzer alarm patterns, and status indicator LEDs[span_5](start_span)[span_5](end_span).
* **Dual-Core Architecture:** Runs network tasks and Telegram polling independently on Core 0 while core security routines execute smoothly on Core 1[span_6](start_span)[span_6](end_span).

## Hardware Requirements
* ESP32 Development Board
* MFRC522 RFID Reader module + compatible UID cards[span_7](start_span)[span_7](end_span)
* 16x2 I2C LCD Display (`0x27` or `0x3F`)[span_8](start_span)[span_8](end_span)
* 2x Laser modules and 2x light sensor panels[span_9](start_span)[span_9](end_span)
* Active Buzzer and status LEDs[span_10](start_span)[span_10](end_span)

## Required Libraries
Make sure you have the following libraries installed in your Arduino IDE:
* `SPI`[span_11](start_span)[span_11](end_span)
* `MFRC522`[span_12](start_span)[span_12](end_span)
* `Wire`[span_13](start_span)[span_13](end_span)
* `LiquidCrystal_I2C`[span_14](start_span)[span_14](end_span)
* `WiFi`[span_15](start_span)[span_15](end_span)
* `WiFiClientSecure`[span_16](start_span)[span_16](end_span)
* `UniversalTelegramBot`[span_17](start_span)[span_17](end_span)
* `ArduinoJson` (v6.x)[span_18](start_span)[span_18](end_span)

## Configuration
Before uploading the code to your ESP32, make sure to update your credentials in the code[span_19](start_span)[span_19](end_span):
1. Set your Wi-Fi SSID and Password (`WIFI_SSID`, `WIFI_PASS`)[span_20](start_span)[span_20](end_span).
2. Input your Telegram Bot Token (`BOT_TOKEN`) and authorized Chat ID (`CHAT_ID`)[span_21](start_span)[span_21](end_span).
3. Register your authorized RFID key card UIDs and corresponding user names in the `AUTH_UIDS` and `AUTH_NAMES` arrays[span_22](start_span)[span_22](end_span).
