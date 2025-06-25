// =========================================================================
//         PROJETO RELÓGIO DE PONTO - SKETCH ARDUINO UNO (ONLINE-ONLY)
// =========================================================================

// --- Bibliotecas ---
#include <Wire.h>
#include <SPI.h>
#include <MFRC522.h>
#include <Keypad.h>
#include <LiquidCrystal_I2C.h>
#include <SoftwareSerial.h>

// --- Definições de Pinos ---
#define SS_PIN   10 // SS do RFID RC522
#define RST_PIN  A2 // RST do RFID RC522
#define ESP_RX_PIN 2 // Arduino RX <- TX do ESP
#define ESP_TX_PIN 3 // Arduino TX -> RX do ESP
const byte ROWS = 4;
const byte COLS = 4;
byte rowPins[ROWS] = {9, 8, 7, 6};
byte colPins[COLS] = {5, A0, A1, A3};

// --- Objetos e Variáveis Globais ---
MFRC522 mfrc522(SS_PIN, RST_PIN);
SoftwareSerial espSerial(ESP_RX_PIN, ESP_TX_PIN);
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
Keypad customKeypad = Keypad( makeKeymap(keys), rowPins, colPins, ROWS, COLS );
LiquidCrystal_I2C lcd(0x27, 16, 2); // Verifique o endereço com I2C Scanner

enum State { STATE_IDLE, STATE_GETTING_MATRICULA, STATE_GETTING_SENHA, STATE_MESSAGE };
State currentState = STATE_IDLE;
String inputMatricula = "";
String inputSenha = "";
unsigned long messageEndTime = 0;

// --- SETUP ---
void setup() {
  Serial.begin(9600);
  while (!Serial);
  Serial.println(F("====================================="));
  Serial.println(F("Inicializando Relogio (Online-Only)"));
  Serial.println(F("====================================="));

  pinMode(SS_PIN, OUTPUT);
  digitalWrite(SS_PIN, HIGH);

  Wire.begin();
  Serial.println(F("I2C Iniciado."));

  Serial.print(F("Iniciando LCD... "));
  lcd.init();
  lcd.backlight();
  lcd.clear();
  Wire.beginTransmission(0x27);
  if (Wire.endTransmission() == 0) { Serial.println(F("OK.")); }
  else { Serial.println(F("FALHA!")); }

  espSerial.begin(9600);
  Serial.println(F("SoftwareSerial OK."));
  
  SPI.begin();
  Serial.println(F("SPI Iniciado."));

  Serial.print(F("Iniciando RFID... "));
  mfrc522.PCD_Init();
  if (mfrc522.PCD_PerformSelfTest()) { Serial.println(F("OK.")); }
  else { Serial.println(F("FALHOU!")); }

  Serial.println(F("====================================="));
  Serial.println(F("Sistema Pronto."));
  lcd_print(F("Aproxime Cracha"), F("ou digite #"));
}

// --- LOOP ---
void loop() {
  switch (currentState) {
    case STATE_IDLE:      handleIdleState();      break;
    case STATE_GETTING_MATRICULA: handleMatriculaState(); break;
    case STATE_GETTING_SENHA:   handleSenhaState();   break;
    case STATE_MESSAGE:   handleMessageState();   break;
  }
  if (espSerial.available()) {
     String fromEsp = espSerial.readStringUntil('\n');
     fromEsp.trim();
     Serial.print(F("Msg do ESP: ")); Serial.println(fromEsp);
     processEspMessage(fromEsp);
  }
}

// --- Funções de Estado ---
void handleIdleState() {
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    String uid = getUidString();
    lcd_print_temp(F("Cartao Lido"), uid, 2000);
    String dataToSend = "TYPE=RFID;UID=" + uid;
    sendDataToEsp(dataToSend);
    mfrc522.PICC_HaltA();
  }
  char key = customKeypad.getKey();
  if (key == '#') {
    inputMatricula = "";
    inputSenha = "";
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print(F("Matricula:"));
    currentState = STATE_GETTING_MATRICULA;
  }
}

void handleMatriculaState() {
  char key = customKeypad.getKey();
  if (key) {
    if (isdigit(key) && inputMatricula.length() < 10) {
      inputMatricula += key;
      lcd.setCursor(0, 1); lcd.print(inputMatricula);
    } else if (key == '*' ) {
      lcd_print_temp(F("Cancelado"), F(""), 1000);
      currentState = STATE_IDLE;
      lcd_print(F("Aproxime Cracha"), F("ou digite #"));
    } else if (key == '#' && inputMatricula.length() > 0) {
       lcd.clear();
       lcd.setCursor(0, 0); lcd.print(F("Senha:"));
       currentState = STATE_GETTING_SENHA;
    }
  }
}

void handleSenhaState() {
  char key = customKeypad.getKey();
  if (key) {
    if (isdigit(key) && inputSenha.length() < 8) {
      inputSenha += key;
      lcd.setCursor(inputSenha.length() - 1, 1); lcd.print("*");
    } else if (key == '*' ) {
      lcd_print_temp(F("Cancelado"), F(""), 1000);
      currentState = STATE_IDLE;
      lcd_print(F("Aproxime Cracha"), F("ou digite #"));
    } else if (key == '#' && inputSenha.length() > 0) {
      lcd_print_temp(F("Aguarde..."), F("Validando..."), 2000);
      String dataToSend = "TYPE=KEYPAD;MAT=" + inputMatricula + ";SEN=" + inputSenha;
      sendDataToEsp(dataToSend);
      inputMatricula = ""; inputSenha = "";
    }
  }
}

void handleMessageState(){
  if (millis() > messageEndTime) {
    currentState = STATE_IDLE;
    lcd_print(F("Aproxime Cracha"), F("ou digite #"));
  }
}

// --- Funções Auxiliares ---
String getUidString() {
  String uid = "";
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    if (mfrc522.uid.uidByte[i] < 0x10) uid += "0";
    uid += String(mfrc522.uid.uidByte[i], HEX);
  }
  uid.toUpperCase();
  return uid;
}

void sendDataToEsp(String data) {
  Serial.print(F("Arduino -> ESP: ")); Serial.println(data);
  espSerial.println(data);
}

void processEspMessage(String msg) {
  if (msg.startsWith("STATUS:")) {
    String statusMsg = msg.substring(7);
    if (statusMsg == "OK") {
       lcd_print_temp(F("Ponto"), F("Registrado!"), 2000);
    } else {
       lcd_print_temp(F("Erro:"), statusMsg, 3000);
    }
  }
}

// --- Funções do LCD ---
void lcd_print(const __FlashStringHelper* line1, const String& line2) {
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print(line1);
    lcd.setCursor(0, 1); lcd.print(line2);
}
void lcd_print(const String& line1, const String& line2) { /*...mesma lógica...*/ } // Implementação completa omitida por brevidade

void lcd_print_temp(const __FlashStringHelper* line1, const String& line2, unsigned long duration) {
    lcd_print(line1, line2);
    currentState = STATE_MESSAGE;
    messageEndTime = millis() + duration;
}
void lcd_print_temp(const String& line1, const String& line2, unsigned long duration) { /*...mesma lógica...*/ } // Implementação completa omitida
