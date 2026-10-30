// ESP32 RFID Laser Security (compact): RFID + LCD + 2 lasers + Telegram
// Card: Welcome -> countdown -> ARMED. Phone /arm arms straight away.
// Disarm: card, /disarm, or 'd' in the Serial Monitor.
// Index 0 = beam 1 (low, 3 cm), index 1 = beam 2 (top, 7 cm).
// Libraries: MFRC522, LiquidCrystal_I2C, UniversalTelegramBot, ArduinoJson (6.x)
 
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <ArduinoJson.h>
 
// ---- Edit these ----
const char* WIFI_SSID = "Galaxy A328624";
const char* WIFI_PASS = "SecTech.5";
const String BOT_TOKEN = "9053377978:AAHllQbrSgp2B70dmXvU1YmOk9-BuD86W1Q";
const String CHAT_ID = "5251885642";          // only this chat can control the system
 
// Card UIDs: capital letters, colon between pairs. Keep both lists in the same order.
String AUTH_UIDS[] = {
 "41:BD:47:6A",   // slot 1
 "71:19:69:E7",   // slot 2
 "5D:72:8B:8E"    // slot 3
};
String AUTH_NAMES[] = {
 "Sec_Tech.5",        // slot 1 (max 16 characters)
 "Professor",        // slot 2
 "Gospel"         // slot 3
};
const int NUM_AUTH = sizeof(AUTH_UIDS) / sizeof(AUTH_UIDS[0]);
 
// ---- Pins ----
const int RFID_SS = 5, RFID_RST = 27, BUZZER_PIN = 4;
const int PANEL[2] = { 34, 35 };
const int LASER[2] = { 16, 17 };
const int LED[2]   = { 13, 15 };
 
// ---- Tuning ----
const float TRIP_FRACTION = 0.94;               // trip below 90% of normal
const int MIN_NORMAL = 25, CONFIRM_READS = 3, SAMPLES = 20;
const float ALPHA = 0.014;                      // normal follows room light in about 7 s
const unsigned long UPDATE_MS = 100, WELCOME_MS = 2000, COUNTDOWN_MS = 3000, SETTLE_MS = 2000;
const unsigned long DISARMED_MS = 3000, DENIED_MS = 2000, SCAN_COOLDOWN = 3000;
const unsigned long LED_FLASH_MS = 150, BLINK_MS = 400, POLL_MS = 1000;
const int SEND_TRIES = 5;
 
MFRC522 rfid(RFID_SS, RFID_RST);
LiquidCrystal_I2C lcd(0x27, 16, 2);             // try 0x3F if the LCD stays blank
WiFiClientSecure secureClient;
UniversalTelegramBot bot(BOT_TOKEN, secureClient);   // used only by the Telegram task
 
// LCD icons: 0 = closed padlock, 1 = open padlock, 2 = warning
byte ICONS[3][8] = {
 { 0b01110, 0b10001, 0b10001, 0b11111, 0b11011, 0b11011, 0b11111, 0b00000 },
 { 0b01110, 0b10001, 0b00001, 0b11111, 0b11011, 0b11011, 0b11111, 0b00000 },
 { 0b00100, 0b01110, 0b01110, 0b00100, 0b00100, 0b00000, 0b00100, 0b00000 }
};
 
enum State { DISARMED, COUNTDOWN, ARMED, TRIPPED };
State state = DISARMED;
 
float norm[2];
int lowCnt[2];
bool tripped[2];
unsigned long lastUpdate, lastFlash, lastPrint, lastScan, lastTimerDraw, lastBlink;
unsigned long deniedUntil, lcdOffAt, countdownStart, armedSince;
int lastShown = -1;                             // -1 = redraw, 99 = welcome shown, else the digit
bool flashOn = false, blinkOn = true;
String currentName = "";
uint8_t armSource = 0;                          // 0 = card or Serial, 1 = phone
 
// ---- Messages between the two cores ----
// The Telegram task never touches the LCD, lasers or buzzer. It sets pendingCmd
// for loop() to carry out, and sends the alerts that loop() leaves in the queue.
enum Cmd { CMD_NONE, CMD_ARM, CMD_DISARM };
volatile int pendingCmd = CMD_NONE;
 
enum AlertCode { A_INTRUSION = 1, A_ARMED, A_DISARMED, A_BEAMERR, A_BUSY,
                A_ALREADY_ARMED, A_ALARM_ACTIVE, A_ALREADY_DISARMED };
struct Alert { uint8_t code; uint8_t info; uint32_t secs; };
const int QSIZE = 8;
Alert alertQ[QSIZE];
volatile int qHead = 0, qTail = 0;
 
const char* const MSG[] = {
 "", "", "System ARMED", "System DISARMED",
 "BEAM ERROR: a beam is not reaching its panel. Check the aim, then try /arm again.",
 "The system is already arming.", "Already ARMED",
 "The alarm is active. Send /disarm first.", "Already DISARMED."
};
 
void pushAlert(uint8_t code, uint8_t info, uint32_t secs) {
 int next = (qHead + 1) % QSIZE;
 if (next == qTail) return;                    // queue full: drop
 alertQ[qHead] = { code, info, secs };
 qHead = next; 
}
 
// ---- LCD ----
void lcdShow(const String& a, const String& b) {
 lcd.backlight();
 lcd.clear();
 lcd.setCursor(0, 0);
 lcd.print(a);
 lcd.setCursor(0, 1);
 lcd.print(b);
}
 
void lcdIcon(int n) { lcd.setCursor(15, 0); lcd.write((byte)n); }
 
void lcdLine(int row, const String& t) {        // overwrite one row, no flicker
 lcd.setCursor(0, row);
 lcd.print(t);
 for (int i = t.length(); i < 16; i++) lcd.print(' ');
}
 
void lcdOff() { lcd.clear(); lcd.noBacklight(); }
 
void drawTimer() {
 unsigned long s = (millis() - armedSince) / 1000;
 char buf[20];
 snprintf(buf, sizeof(buf), "Guard %02lu:%02lu:%02lu", s / 3600, (s / 60) % 60, s % 60);
 lcdLine(1, String(buf));
}
 
void drawArmed() {
 lcdShow("ARMED", "");
 lcdIcon(0);
 drawTimer();
 lastTimerDraw = millis();
}
 
void drawTrippedLine1(bool on) { lcdLine(0, on ? " \x02 INTRUSION \x02" : ""); }
 
void showTripped() {
 lcdShow("", tripped[0] && tripped[1] ? "Beams 1 + 2" : tripped[0] ? "Beam 1 (low)" : "Beam 2 (top)");
 blinkOn = true;
 lastBlink = millis();
 drawTrippedLine1(true);
}
 
void restoreScreen() {
 if (state == DISARMED) lcdOff();
 else if (state == COUNTDOWN) lastShown = -1;
 else if (state == ARMED) drawArmed();
 else showTripped();
}
 
// ---- Panels, lasers, alarm ----
int readPanel(int i) {
 long total = 0;
 for (int k = 0; k < SAMPLES; k++) {
   total += analogRead(PANEL[i]);
   delayMicroseconds(500);
 }
 return total / SAMPLES;
}
 
void setLasers(bool on) { for (int i = 0; i < 2; i++) digitalWrite(LASER[i], on); }
 
void setAlarm(bool on) {
 digitalWrite(BUZZER_PIN, on);
 for (int i = 0; i < 2; i++) digitalWrite(LED[i], on);
}
 
// Beam 1 cut = two beeps, beam 2 cut = three beeps, both = fast continuous
bool buzzerPattern() {
 unsigned long t = millis();
 if (tripped[0] && tripped[1]) return (t / 150) % 2 == 0;
 if (tripped[0]) { unsigned long p = t % 800; return p < 150 || (p >= 300 && p < 450); }
 if (tripped[1]) { unsigned long p = t % 1000; return p < 100 || (p >= 200 && p < 300) || (p >= 400 && p < 500); }
 return false;
}
 
// ---- RFID ----
String readCard() {
 if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return "";
 String uid = "";
 for (byte i = 0; i < rfid.uid.size; i++) {
   if (i > 0) uid += ":";
   if (rfid.uid.uidByte[i] < 0x10) uid += "0";
   uid += String(rfid.uid.uidByte[i], HEX);
 }
 uid.toUpperCase();
 rfid.PICC_HaltA();
 rfid.PCD_StopCrypto1();
 return uid;
}
 
int findUser(const String& uid) {
 for (int i = 0; i < NUM_AUTH; i++) if (uid == AUTH_UIDS[i]) return i;
 return -1;
}
 
// ---- State changes ----
void disarm(uint8_t source, bool notify) {
 setLasers(false);
 setAlarm(false);
 state = DISARMED;
 deniedUntil = 0;
 lcdShow("DISARMED", "");
 lcdIcon(1);
 lcdOffAt = millis() + DISARMED_MS;
 Serial.println("DISARMED");
 if (notify) pushAlert(A_DISARMED, source, 0);
}
 
void startCountdown(const String& name) {
 currentName = name;
 armSource = 0;
 state = COUNTDOWN;
 countdownStart = millis();
 lastShown = -1;
 lcdOffAt = 0;
 Serial.println("Card accepted: " + name);
}
 
void finishArming() {
 setLasers(true);                              // both lasers on together
 lcdShow("Lasers on", "Please wait");
 delay(SETTLE_MS);
 
 bool ok = true;
 for (int i = 0; i < 2; i++) {
   norm[i] = readPanel(i);
   if (norm[i] < MIN_NORMAL) ok = false;
 }
 Serial.printf("Starting normal: panel1 = %.0f, panel2 = %.0f\n", norm[0], norm[1]);
 
 if (!ok) {
   Serial.println("BEAM ERROR: a panel reads too weak. Check aim or shading.");
   setLasers(false);
   lcdShow("BEAM ERROR", "Check aim");
   pushAlert(A_BEAMERR, 0, 0);
   delay(2000);
   disarm(armSource, false);                   // no second Telegram message
   return;
 }
 
 for (int i = 0; i < 2; i++) { lowCnt[i] = 0; tripped[i] = false; }
 lastUpdate = armedSince = millis();
 state = ARMED;
 drawArmed();
 Serial.println("ARMED");
 pushAlert(A_ARMED, armSource, 0);
}
 
// beamBits: 1 = beam 1, 2 = beam 2, 3 = both
void breach(uint8_t beamBits) {
 state = TRIPPED;
 showTripped();
 Serial.printf("INTRUSION! Beam cut: %s%s\n", (beamBits & 1) ? "low (3 cm) " : "", (beamBits & 2) ? "top (7 cm)" : "");
 pushAlert(A_INTRUSION, beamBits, (millis() - armedSince) / 1000);
}
 
// ---- Telegram (runs on core 0) ----
String hms(uint32_t s) {
 char buf[16];
 snprintf(buf, sizeof(buf), "%02lu:%02lu:%02lu", (unsigned long)(s / 3600), (unsigned long)((s / 60) % 60), (unsigned long)(s % 60));
 return String(buf);
}
 
String buildMessage(const Alert& a) {
 String m = MSG[a.code];
 if (a.code == A_INTRUSION) {
   m = "INTRUSION! Beam cut:";
   if (a.info & 1) m += " low (3 cm)";
   if (a.info & 2) m += " top (7 cm)";
   m += "\nArmed for " + hms(a.secs) + ".\nSend /disarm to silence the alarm.";
 } else if (a.code == A_ARMED || a.code == A_DISARMED) {
   m += (a.info == 1) ? " (phone)." : " (card).";
 } else if (a.code == A_ALREADY_ARMED) {
   m += " (for " + hms(a.secs) + ").";
 }
 return m;
}
 
bool sendWithRetry(const String& msg) {
 for (int i = 0; i < SEND_TRIES; i++) {
   if (WiFi.status() == WL_CONNECTED && bot.sendMessage(CHAT_ID, msg, "")) return true;
   vTaskDelay(2000 / portTICK_PERIOD_MS);
 }
 Serial.println("Telegram: message could not be sent");
 return false;
}
 
void handleMessages(int count) {
 for (int i = 0; i < count; i++) {
   if (String(bot.messages[i].chat_id) != CHAT_ID) continue;   // ignore other chats
   String text = bot.messages[i].text;
   text.trim();
   text.toLowerCase();
   int at = text.indexOf('@');
   if (at > 0) text = text.substring(0, at);
   if (text == "/arm") pendingCmd = CMD_ARM;
   else if (text == "/disarm") pendingCmd = CMD_DISARM;
   else if (text == "/start" || text == "/help")
     sendWithRetry("Commands:\n/arm  arm the system\n/disarm  disarm the system");
 }
}
 
void telegramTask(void* param) {
 secureClient.setInsecure();                   // skips the certificate check (fine for a hobby project)
 bool firstRun = true;
 unsigned long lastPoll = 0;
 
 for (;;) {
   if (WiFi.status() != WL_CONNECTED) {
     WiFi.begin(WIFI_SSID, WIFI_PASS);
     unsigned long t = millis();
     while (WiFi.status() != WL_CONNECTED && millis() - t < 10000) vTaskDelay(500 / portTICK_PERIOD_MS);
     if (WiFi.status() != WL_CONNECTED) continue;
     Serial.println("WiFi connected");
   }
 
   if (firstRun) {
     firstRun = false;
     for (int i = 0; i < 10; i++)              // flush old messages so a stale /arm can't arm at boot
       if (bot.getUpdates(bot.last_message_received + 1) <= 0) break;
     sendWithRetry("System online. Send /arm or /disarm.");
   }
 
   while (qTail != qHead) {                    // send queued alerts first
     sendWithRetry(buildMessage(alertQ[qTail]));
     qTail = (qTail + 1) % QSIZE;
   }
 
   if (millis() - lastPoll >= POLL_MS) {
     lastPoll = millis();
     int n = bot.getUpdates(bot.last_message_received + 1);
     while (n > 0) {
       handleMessages(n);
       n = bot.getUpdates(bot.last_message_received + 1);
     }
   }
   vTaskDelay(100 / portTICK_PERIOD_MS);
 }
}
 
// ---- Setup ----
void setup() {
 Serial.begin(115200);
 pinMode(BUZZER_PIN, OUTPUT);
 analogReadResolution(12);
 for (int i = 0; i < 2; i++) {
   pinMode(LASER[i], OUTPUT);
   pinMode(LED[i], OUTPUT);
   analogSetPinAttenuation(PANEL[i], ADC_11db);
 }
 setLasers(false);
 setAlarm(false);
 
 Wire.begin(21, 22);
 lcd.init();
 for (int i = 0; i < 3; i++) lcd.createChar(i, ICONS[i]);
 lcdOff();
 
 SPI.begin(18, 19, 23, RFID_SS);
 rfid.PCD_Init();
 
 WiFi.mode(WIFI_STA);
 xTaskCreatePinnedToCore(telegramTask, "telegram", 12288, NULL, 1, NULL, 0);   // core 0
 Serial.println("Ready. Scan a card, or type 'a' to arm and 'd' to disarm.");
}
 
// ---- Main loop ----
void loop() {
 while (Serial.available()) {
   char c = Serial.read();
   if (c == 'a' && state == DISARMED) startCountdown("Admin");
   if (c == 'd') disarm(0, true);
 }
 
 // Commands left by the Telegram task
 int cmd = pendingCmd;
 if (cmd != CMD_NONE) {
   pendingCmd = CMD_NONE;
   if (cmd == CMD_ARM) {
     if (state == DISARMED) {                  // phone arming: no welcome, no countdown
       currentName = "Phone";
       armSource = 1;
       lcdOffAt = deniedUntil = 0;
       Serial.println("Telegram: /arm");
       finishArming();
     } else {
       pushAlert(state == COUNTDOWN ? A_BUSY : state == ARMED ? A_ALREADY_ARMED : A_ALARM_ACTIVE,
                 0, (millis() - armedSince) / 1000);
     }
   } else {
     Serial.println("Telegram: /disarm");
     if (state == DISARMED) pushAlert(A_ALREADY_DISARMED, 0, 0);
     else disarm(1, true);
   }
 }
 
 // Card scan (an unknown card never pauses beam detection)
 String uid = readCard();
 if (uid != "" && millis() - lastScan >= SCAN_COOLDOWN) {
   lastScan = millis();
   Serial.println("Card UID: " + uid);
   int user = findUser(uid);
   if (user >= 0) {
     if (state == DISARMED) startCountdown(AUTH_NAMES[user]);
     else disarm(0, true);                     // a valid card disarms from any state
   } else {
     lcdShow("ACCESS DENIED", "");
     deniedUntil = millis() + DENIED_MS;
     lcdOffAt = 0;
   }
 }
 
 if (deniedUntil != 0 && millis() > deniedUntil) {   // restore the screen after ACCESS DENIED
   deniedUntil = 0;
   restoreScreen();
 }
 
 if (state == DISARMED && lcdOffAt != 0 && millis() > lcdOffAt) {
   lcdOffAt = 0;
   lcdOff();
 }
 
 // Welcome, then countdown 3-2-1, then arm
 if (state == COUNTDOWN) {
   unsigned long elapsed = millis() - countdownStart;
   if (elapsed < WELCOME_MS) {
     if (lastShown != 99 && deniedUntil == 0) {
       lastShown = 99;
       lcdShow("Welcome,", currentName);
     }
   } else if (elapsed - WELCOME_MS >= COUNTDOWN_MS) {
     finishArming();
   } else {
     int secs = (COUNTDOWN_MS - (elapsed - WELCOME_MS) + 999) / 1000;
     if (secs != lastShown && deniedUntil == 0) {
       lastShown = secs;
       lcdShow("Arming in", String(secs));
     }
   }
 }
 
 // Watching the beams: each beam works on its own, even during an alarm
 if (state == ARMED || state == TRIPPED) {
   bool doUpdate = millis() - lastUpdate >= UPDATE_MS;
   if (doUpdate) lastUpdate = millis();
   uint8_t newBits = 0;
   int r[2] = { 0, 0 };
 
   for (int i = 0; i < 2; i++) {
     if (tripped[i]) continue;
     r[i] = readPanel(i);
     bool below = r[i] < norm[i] * TRIP_FRACTION;
     lowCnt[i] = below ? lowCnt[i] + 1 : 0;
     if (doUpdate && !below) norm[i] += (r[i] - norm[i]) * ALPHA;   // follow room light
     if (lowCnt[i] >= CONFIRM_READS) {
       tripped[i] = true;
       digitalWrite(LASER[i], LOW);            // this beam's laser goes off
       newBits |= 1 << i;
     }
   }
 
   if (state == ARMED && deniedUntil == 0 && millis() - lastTimerDraw >= 1000) {
     lastTimerDraw = millis();
     drawTimer();
   }
 
   if (millis() - lastPrint >= 250) {
     lastPrint = millis();
     for (int i = 0; i < 2; i++) {
       Serial.print(i ? "    P2 " : "P1 ");
       if (tripped[i]) Serial.print("CUT");
       else Serial.printf("%d / normal %d", r[i], (int)norm[i]);
     }
     Serial.println();
   }
 
   if (newBits) breach(newBits);               // one alert, even if both beams are cut together
 }
 
 // Alarm: buzzer pattern, and only the LED of each cut beam blinks
 if (state == TRIPPED) {
   digitalWrite(BUZZER_PIN, buzzerPattern());
   if (millis() - lastFlash >= LED_FLASH_MS) {
     lastFlash = millis();
     flashOn = !flashOn;
     for (int i = 0; i < 2; i++) digitalWrite(LED[i], tripped[i] && flashOn);
   }
   if (deniedUntil == 0 && millis() - lastBlink >= BLINK_MS) {
     lastBlink = millis();
     blinkOn = !blinkOn;
     drawTrippedLine1(blinkOn);
   }
 }
}
 
 
