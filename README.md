#include <SPI.h>
#include <MFRC522.h>
#include <Servo_ESP32.h>
#include <WiFi.h>
#include <ESPAsyncWebServer.h>

#define SS_PIN    5    // ESP32 pin GPIO5
#define RST_PIN   27   // ESP32 pin GPIO27
#define BUZZER_PIN 22   // Buzzer pin
#define BLUE_LED  12   // Blue LED GPIO pin
#define RED_LED   16   // Red LED GPIO pin
#define GREEN_LED 21   // Green LED GPIO pin
#define SERVO_PIN  15   // Servo motor GPIO pin

MFRC522 rfid(SS_PIN, RST_PIN);
Servo_ESP32 gateServo;

const char* ssid = "Khushi";
const char* password = "47273ECECD9E";

AsyncWebServer server(80);

bool blueLEDOn = true;
bool greenLEDOn = false;
bool redLEDOn = false;

unsigned long accessStartTime = 0;
unsigned long gateCloseTime = 0;
bool gateOpen = false;

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  gateServo.attach(SERVO_PIN);
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BLUE_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  digitalWrite(BLUE_LED, HIGH);
  blueLEDOn = true;

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");

  // Print the ESP32's IP address to the Serial Monitor
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
  String html = "<html><body>";
  html += "<h1>RFID Gate Control</h1>";
  html += "<button style=\"width: 100px; height: 100px; font-size: 20px;\" onclick=\"openGate()\">Open</button>";
  html += "<button style=\"width: 100px; height: 100px; font-size: 20px;\" onclick=\"closeGate()\">Close</button>";
  html += "<script>";
  html += "function openGate() {";
  html += "  fetch('/open')";
  html += "}";
  html += "function closeGate() {";
  html += "  fetch('/close')";
  html += "}";
  html += "</script>";
  html += "</body></html>";
  request->send(200, "text/html", html);
});


  server.on("/open", HTTP_GET, [](AsyncWebServerRequest *request){
    openGate();
    request->send(200, "text/plain", "Gate opened.");
  });

  server.on("/close", HTTP_GET, [](AsyncWebServerRequest *request){
    closeGate();
    request->send(200, "text/plain", "Gate closed.");
  });

  server.begin();

  Serial.println("Tap an RFID/NFC tag on the RFID-RC522 reader");
}

void loop() {
  unsigned long currentTime = millis();
  unsigned long elapsedTime = currentTime - accessStartTime;
  
  // Check if the gate was opened and it's time to close it
  if (gateOpen && (currentTime - gateCloseTime >= 8000)) {
    closeGate();
    gateOpen = false;
    digitalWrite(BLUE_LED, HIGH);
    blueLEDOn = true;
  }

  if (greenLEDOn && elapsedTime >= 8000) {
    digitalWrite(GREEN_LED, LOW);
    greenLEDOn = false;
    digitalWrite(BLUE_LED, HIGH);
    blueLEDOn = true;
  }

  if (redLEDOn && elapsedTime >= 8000) {
    digitalWrite(RED_LED, LOW);
    redLEDOn = false;
    digitalWrite(BLUE_LED, HIGH);
    blueLEDOn = true;
  }

  if (rfid.PICC_IsNewCardPresent()) {
    if (rfid.PICC_ReadCardSerial()) {
      MFRC522::PICC_Type piccType = rfid.PICC_GetType(rfid.uid.sak);
      Serial.print("RFID/NFC Tag Type: ");
      Serial.println(rfid.PICC_GetTypeName(piccType));

      Serial.print("UID:");
      for (int i = 0; i < rfid.uid.size; i++) {
        Serial.print(rfid.uid.uidByte[i] < 0x10 ? " 0" : " ");
        Serial.print(rfid.uid.uidByte[i], HEX);
      }
      Serial.println();

      if (isAuthorized(rfid.uid)) {
        beepHigh(375);
        digitalWrite(GREEN_LED, HIGH);
        greenLEDOn = true;
        if (redLEDOn) {
          digitalWrite(RED_LED, LOW);
          redLEDOn = false;
        }
        if (blueLEDOn) {
          digitalWrite(BLUE_LED, LOW);
          blueLEDOn = false;
        }
        accessStartTime = currentTime;
        openGate();
        gateCloseTime = currentTime; // Set the time for gate closing
        gateOpen = true; // Indicate that the gate is open
      } else {
        beepHigh(1000);
        digitalWrite(RED_LED, HIGH);
        redLEDOn = true;
        if (greenLEDOn) {
          digitalWrite(GREEN_LED, LOW);
          greenLEDOn = false;
        }
        if (blueLEDOn) {
          digitalWrite(BLUE_LED, LOW);
          blueLEDOn = false;
        }
        accessStartTime = currentTime;
        // You can close the gate here if needed
        closeGate();
      }

      rfid.PICC_HaltA();
      rfid.PCD_StopCrypto1();
    }
  }
}

bool isAuthorized(MFRC522::Uid uid) {
  // Modify this condition to match your authorized UID
  return (uid.uidByte[0] == 0x03 &&
          uid.uidByte[1] == 0x59 &&
          uid.uidByte[2] == 0xD8 &&
          uid.uidByte[3] == 0x1B);
}

void beepHigh(int duration) {
  tone(BUZZER_PIN, 2000, duration);
  delay(1000);
  noTone(BUZZER_PIN);
}

void openGate() {
  gateServo.write(90);
}

void closeGate() {
  gateServo.write(0);
}
