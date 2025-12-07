#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ====== WiFi Credentials ======
const char* ssid = "Prabakaran";
const char* password = "        ";

// ====== Telegram Bot ======
const char* BOT_TOKEN = "8402164632:AAFqMrzp2-mZklqBgC4GHuMHoz-iSrrQ4lg";
String chat_id = "2078130403";

// ====== Pin Configuration ======
#define IR1_PIN 13
#define IR2_PIN 12
#define MQ135_PIN 34

// ====== Thresholds ======
int people_threshold = 10;
int gas_threshold = 5000;

// ====== Variables ======
int people_count = 0;
bool ir1_triggered = false;
bool alert_sent = false;

// ====== LCD Setup ======
LiquidCrystal_I2C lcd(0x27, 16, 2); // change 0x27 to your I2C address

// ====== WiFi & Telegram Setup ======
WiFiClientSecure client;
UniversalTelegramBot bot(BOT_TOKEN, client);

// ====== Connect WiFi ======
void connectWiFi() {
  Serial.print("Connecting to WiFi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected!");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());
  client.setInsecure();
}

// ====== Setup ======
void setup() {
  Serial.begin(115200);
  Serial.println("Starting Smart Hygiene Monitor...");

  pinMode(IR1_PIN, INPUT_PULLUP);
  pinMode(IR2_PIN, INPUT_PULLUP);
  pinMode(MQ135_PIN, INPUT);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Hygiene Sys");

  connectWiFi();
  bot.sendMessage(chat_id, "Hygiene Monitor Online", "");
  delay(1500);
}

// ====== Main Loop ======
void loop() {
  // Read sensors
  int ir1 = digitalRead(IR1_PIN); // LOW = triggered
  int ir2 = digitalRead(IR2_PIN); // LOW = triggered
  int gas_level = analogRead(MQ135_PIN);

  // Debug serial
  Serial.print("IR1: "); Serial.print(ir1);
  Serial.print(" IR2: "); Serial.print(ir2);
  Serial.print(" Gas: "); Serial.println(gas_level);

  // ---- Entry Detection Logic ----
  if (ir1 == LOW && !ir1_triggered) {
    ir1_triggered = true;  // IR1 detected first
    Serial.println("IR1 triggered");
  }

  if (ir1_triggered && ir2 == LOW) {
    people_count++;
    ir1_triggered = false;
    Serial.println("Person Entered, Count: " + String(people_count));
    delay(200); // small debounce
  }

  if (ir2 == LOW && !ir1_triggered) {
    // Invalid sequence, reset
    ir1_triggered = false;
  }

  // ---- LCD Display ----
  static int last_count = -1;
  static int last_gas = -1;
  if (people_count != last_count || gas_level != last_gas) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Count: "); lcd.print(people_count);
    lcd.setCursor(0, 1);
    lcd.print("Gas: "); lcd.print(gas_level);
    last_count = people_count;
    last_gas = gas_level;
  }

  // ---- Alert Condition ----
  if ((people_count > people_threshold || gas_level > gas_threshold) && !alert_sent) {
    String alert = "ALERT!\nPeople: " + String(people_count) +
                   "\nGas: " + String(gas_level) +
                   "\nHygiene Alert!";
    bot.sendMessage(chat_id, alert, "");
    Serial.println("ALERT SENT");

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("ALERT! Check");
    lcd.setCursor(0, 1);
    lcd.print("Hygiene!");
    alert_sent = true;
    delay(2000);
  }

  // Reset alert if conditions normalize
  if (people_count <= people_threshold && gas_level <= gas_threshold) {
    alert_sent = false;
  }

  delay(200); // loop delay
}
