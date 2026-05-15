#define BLYNK_TEMPLATE_ID "TMPL2JOTuEfSe"
#define BLYNK_TEMPLATE_NAME "iot"
#define BLYNK_AUTH_TOKEN "OfHUt0h_hhhWPYnt8xrplPuhZczX3On2"

#include <WiFi.h>
#include <PubSubClient.h>
#include <ESP32Servo.h>
#include <DHT.h>

// ================= WIFI =================
char ssid[] = "Yasminaa";
char pass[] = "12341234";

// ================= BLYNK MQTT =================
const char* mqtt_server = "ny3.blynk.cloud";
const int   mqtt_port   = 1883;
const char* mqtt_user   = "device";
const char* mqtt_pass   = BLYNK_AUTH_TOKEN;

WiFiClient   espClient; //Creates internet communication object
PubSubClient mqtt(espClient);// creat mqtt object  to publish,subscribe..

// ================= SYSTEM =================
bool systemOn = false;

// ================= DHT + FAN =================
#define DHTPIN   5
#define DHTTYPE  DHT11
DHT dht(DHTPIN, DHTTYPE);

#define FAN_PIN 18
float temperature = 0;
bool  manualFan   = false;

// ================= ULTRASONIC =================
#define TRIG_IN  13
#define ECHO_IN  12
#define TRIG_OUT 27
#define ECHO_OUT 33

#define DIST_THRESHOLD 7
#define COOLDOWN_MS    2000 //after counting wait 2 sec

unsigned long lastTriggerIn  = 0; //to store last detection num
unsigned long lastTriggerOut = 0;
bool inZone  = false;
bool outZone = false;//prevent repeted trigrring if object remains infront sensor

// ================= SOUND =================
#define SOUND_PIN 32
int baseline   = 0; //normal enviroment sound level

int soundLevel = 0;
int soundValue = 0;

// ================= LEDS =================
#define GREEN_LED  16
#define YELLOW_LED 17
#define RED_LED    21

unsigned long ledTimer = 0;//hold time
const int holdTime = 2000;
int activeLED = 0;

// ================= SERVO =================
#define SERVO_PIN 26
Servo myServo;
bool servoActive = false;
unsigned long servoUpTime = 0;//used for delayed reset.

// ================= MQ135 =================
#define MQ135_PIN 34
#define BUZZER    25
int gasThreshold = 600;
int gasValue     = 0;

// ================= LDR =================
#define LDR_PIN 35
#define LED1_PIN    19
#define LED2_PIN    4
#define LED1_PWM_CH 2
#define LED2_PWM_CH 3
#define LDR_BRIGHT_THRESHOLD 1000
int lightValue = 0;
int brightness = 0;

// ================= PEOPLE =================
int peopleInside = 0;
int peopleOut    = 0;
int totalEntries = 0;
int totalProfit  = 0;

// ================= CAPACITY =================
#define MAX_CAPACITY 5

// ================= BUZZER =================
unsigned long lastBeepToggle = 0;
bool buzzerState = false;
#define BEEP_INTERVAL 400

// ================= TIMERS =================
unsigned long lastUpdate      = 0;
unsigned long lastReconnectAt = 0;
#define UPDATE_INTERVAL    2000
#define RECONNECT_INTERVAL 5000 //reconnect every 5 secs

// ================= CONNECTION =================
bool wifiConnected = false;
bool mqttConnected = false;

// ======================================================
// PUBLISH HELPER
// Uses exact datastream NAME as topic: ds/People In
// This is the correct Blynk MQTT API format
// ======================================================
void pub(const char* datastreamName, String value)//creat function for mqtt topic
{
  if (!mqtt.connected()) return;//if mqtt disconnected ,stop function
  String topic = "ds/" + String(datastreamName);//creat mqtt topic
  mqtt.publish(topic.c_str(), value.c_str());//send topic and payload to blynk broker
}

// ======================================================
// MQTT CALLBACK — receives commands from Blynk dashboard
// ======================================================
void mqttCallback(char* topic, byte* payload, unsigned int length)//esp32 receives mqtt from broker
{
  String message = "";//to store icomming massages
  for (unsigned int i = 0; i < length; i++)
    message += (char)payload[i];//convert bytes into redable text

  String t = String(topic);

  Serial.print("MQTT IN: "); Serial.print(t);
  Serial.print(" = ");       Serial.println(message);

  // Ignore server redirect
  if (t == "downlink/redirect") return;

  // System ON/OFF — datastream name is "System ON"
  if (t == "downlink/ds/System ON")//check if dahboard button changed?
  {
    systemOn = (message.toInt() == 1);//covert paylod into boolean 0--off
    Serial.println(systemOn ? "System ON" : "System OFF");
  }

  // Fan control — datastream name is "Manual Fan Control"
  if (t == "downlink/ds/Manual Fan Control")
  {
    manualFan = (message.toInt() == 1);
    Serial.println(manualFan ? "Fan ON" : "Fan OFF");
  }
}

// ======================================================
// MQTT RECONNECT
// ======================================================
void reconnectMQTT()//reconnects esp32 with mqtt broker if dissconnected 
{
  if (WiFi.status() != WL_CONNECTED) return;//NO Wifi stop reconnecting

  unsigned long now = millis();
  if (now - lastReconnectAt < RECONNECT_INTERVAL) return;
  lastReconnectAt = now;

  Serial.println("Connecting to Blynk MQTT...");

  String clientId = "ESP32_" + String((uint32_t)ESP.getEfuseMac(), HEX);

  if (mqtt.connect(clientId.c_str(), mqtt_user, mqtt_pass))//mqtt login 
  {
    mqttConnected = true;
    Serial.println("MQTT Connected!");

    // Subscribe to all dashboard commands fan /system
    mqtt.subscribe("downlink/#");

    // Tell Blynk device is online
    String info = "{\"tmpl\":\"" + String(BLYNK_TEMPLATE_ID) + "\","
                  "\"ver\":\"1.0.0\","
                  "\"build\":\"" + String(__DATE__) + "\","
                  "\"rxbuff\":1024}";
    mqtt.publish("info/mcu", info.c_str());//Sends device information to Blynk cloud.
    Serial.println("Device online");
  }
  else
  {
    mqttConnected = false;
    Serial.print("MQTT failed rc=");
    Serial.println(mqtt.state());
  }
}

// ======================================================
// SEND ALL DATA TO BLYNK
// Uses exact datastream names from your Blynk console
// ======================================================
void sendData()
{
  if (!mqtt.connected()) return;

  totalProfit = totalEntries * 35;

  // Exact names from your Blynk datastream console:
  pub("People In",          String(peopleInside));
  pub("People out",         String(peopleOut));
  pub("Total Profit",       String(totalProfit));
  pub("Sound",              String(soundValue));
  pub("Servo",              String((int)servoActive));
  pub("Gas Value",          String(gasValue));
  pub("Gas Alert",          String(gasValue > gasThreshold ? 1 : 0));
  pub("Light Value",        String(lightValue));
  pub("Temperature",        String((int)temperature));   // Integer datastream
  pub("available",          String(peopleInside < MAX_CAPACITY ? 1 : 0));
  pub("full capasity",      String(peopleInside >= MAX_CAPACITY ? 1 : 0));

  Serial.println("Data sent to Blynk via MQTT");
}

// ======================================================
float readDistance(int trig, int echo)
{
  digitalWrite(trig, LOW);
  delayMicroseconds(4);
  digitalWrite(trig, HIGH);
  delayMicroseconds(10);
  digitalWrite(trig, LOW);
  long d = pulseIn(echo, HIGH, 15000);
  delay(30);
  return d == 0 ? 999 : d * 0.034 / 2.0;
}

// ======================================================
void setup()
{
  Serial.begin(115200);
  delay(500);

  pinMode(TRIG_IN,  OUTPUT);
  pinMode(ECHO_IN,  INPUT);
  pinMode(TRIG_OUT, OUTPUT);
  pinMode(ECHO_OUT, INPUT);

  pinMode(GREEN_LED,  OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(RED_LED,    OUTPUT);

  pinMode(BUZZER,  OUTPUT);
  pinMode(FAN_PIN, OUTPUT);

  ledcSetup(LED1_PWM_CH, 5000, 8);//configure pwm channel 
  ledcAttachPin(LED1_PIN, LED1_PWM_CH);//cconnect pwm channel to led pin
  ledcSetup(LED2_PWM_CH, 5000, 8);
  ledcAttachPin(LED2_PIN, LED2_PWM_CH);

  myServo.attach(SERVO_PIN);//connect servo to gpio pin
  myServo.write(0);

  dht.begin();

  // Sound baseline
  long sum = 0;
  for (int i = 0; i < 100; i++) {
    sum += analogRead(SOUND_PIN);
    delay(5);
  }
  baseline = sum / 100;
  Serial.print("Sound baseline: ");
  Serial.println(baseline);

  // WiFi — 15s timeout then local mode
  Serial.print("Connecting to WiFi");
  WiFi.mode(WIFI_STA); //set esp as client
  WiFi.begin(ssid, pass);

  unsigned long wifiStart = millis();//time where wifi is connected
  while (WiFi.status() != WL_CONNECTED && millis() - wifiStart < 15000)
  {
    delay(500);
    Serial.print(".");
  }

  if (WiFi.status() == WL_CONNECTED)//wifi connected suscc?
  {
    wifiConnected = true;
    Serial.println("\nWiFi Connected: " + WiFi.localIP().toString());

    mqtt.setServer(mqtt_server, mqtt_port);// sets ny3.blynk.cloud
    mqtt.setBufferSize(1024);//1024 bytes for json size
    mqtt.setKeepAlive(60);
    mqtt.setSocketTimeout(10);
    mqtt.setCallback(mqttCallback);//dashboard commands

    delay(500);
    reconnectMQTT();
  }
  else
  {
    wifiConnected = false;
    mqttConnected = false;
    Serial.println("\nWiFi FAILED — local mode");
    systemOn = true;
  }
}

// ======================================================
void loop()
{
  // ===== CONNECTIVITY =====
  if (WiFi.status() == WL_CONNECTED)
  {
    wifiConnected = true;
    if (!mqtt.connected())
    {
      mqttConnected = false;
      reconnectMQTT();
    }
    else
    {
      mqttConnected = true;
      mqtt.loop();
    }
  }
  else
  {
    if (wifiConnected)
    {
      wifiConnected = false;
      mqttConnected = false;
      Serial.println("WiFi lost — local mode");
      systemOn = true;
    }
  }

  // ===== SYSTEM OFF =====
  if (!systemOn)
  {
    digitalWrite(FAN_PIN,    LOW);
    digitalWrite(BUZZER,     LOW);
    digitalWrite(GREEN_LED,  LOW);
    digitalWrite(YELLOW_LED, LOW);
    digitalWrite(RED_LED,    LOW);
    ledcWrite(LED1_PWM_CH, 0);
    ledcWrite(LED2_PWM_CH, 0);
    myServo.write(0);
    buzzerState = false;
    delay(200);
    return;
  }

  unsigned long now = millis();

  // ===== SOUND + LED =====
  int diff = abs(analogRead(SOUND_PIN) - baseline);
  soundValue = diff;

  if (diff <= 20)       soundLevel = 0;
  else if (diff <= 100)  soundLevel = 1;
  else                  soundLevel = 2;

  if (soundLevel != 0) {
    activeLED = soundLevel;
    ledTimer  = now;
  }

  if ((now - ledTimer) < (unsigned long)holdTime) {
    digitalWrite(GREEN_LED,  LOW);
    digitalWrite(YELLOW_LED, activeLED == 1 ? HIGH : LOW);
    digitalWrite(RED_LED,    activeLED == 2 ? HIGH : LOW);
  } else {
    digitalWrite(GREEN_LED,  HIGH);
    digitalWrite(YELLOW_LED, LOW);
    digitalWrite(RED_LED,    LOW);
  }

  // ===== SERVO =====
  if (soundLevel == 2 && !servoActive) {
    myServo.write(70);
    servoActive = true;
    servoUpTime = now;
  }
  if (servoActive && (now - servoUpTime >= 2000)) {
    myServo.write(0);
    servoActive = false;
  }

  // ===== ULTRASONIC =====
  float dIn  = readDistance(TRIG_IN,  ECHO_IN);
  float dOut = readDistance(TRIG_OUT, ECHO_OUT);

  if (dIn < DIST_THRESHOLD && !inZone) {
    inZone = true;
    if (now - lastTriggerIn > COOLDOWN_MS) {
      lastTriggerIn = now;
      peopleInside++;
      totalEntries++;
      Serial.print("Entered. Inside: "); Serial.println(peopleInside);
    }
  }
  if (dIn >= DIST_THRESHOLD) inZone = false;

  if (dOut < DIST_THRESHOLD && !outZone) {
    outZone = true;
    if (now - lastTriggerOut > COOLDOWN_MS) {
      if (peopleInside > 0) {
        lastTriggerOut = now;
        peopleInside--;
        peopleOut++;
        Serial.print("Left. Inside: "); Serial.println(peopleInside);
      }
    }
  }
  if (dOut >= DIST_THRESHOLD) outZone = false;

  // ===== TEMPERATURE + FAN =====
  float t = dht.readTemperature();
  if (!isnan(t)) temperature = t;

  if (mqttConnected)
    digitalWrite(FAN_PIN, (manualFan && temperature >= 24) ? HIGH : LOW);
  else
    digitalWrite(FAN_PIN, temperature >= 24 ? HIGH : LOW);

  // ===== LDR =====
  lightValue = analogRead(LDR_PIN);
  brightness = (lightValue < LDR_BRIGHT_THRESHOLD)
    ? 0
    : constrain(map(lightValue, LDR_BRIGHT_THRESHOLD, 4095, 0, 255), 0, 255);

  ledcWrite(LED1_PWM_CH, brightness);
  ledcWrite(LED2_PWM_CH, brightness);

  // ===== GAS + BUZZER =====
  gasValue = analogRead(MQ135_PIN);

  if (gasValue > gasThreshold)
  {
    digitalWrite(BUZZER, HIGH);
    buzzerState    = true;
    lastBeepToggle = now;
  }
  else if (peopleInside >= MAX_CAPACITY)
  {
    if (now - lastBeepToggle >= BEEP_INTERVAL)
    {
      lastBeepToggle = now;
      buzzerState    = !buzzerState;
      digitalWrite(BUZZER, buzzerState ? HIGH : LOW);
    }
  }
  else
  {
    digitalWrite(BUZZER, LOW);
    buzzerState    = false;
    lastBeepToggle = now;
  }

  // ===== SEND DATA every 2s =====
  if (now - lastUpdate >= UPDATE_INTERVAL)
  {
    lastUpdate = now;
    sendData();
  }
}
