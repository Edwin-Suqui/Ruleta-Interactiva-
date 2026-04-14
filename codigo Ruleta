#include <SPI.h>
#include <MFRC522v2.h>
#include <MFRC522DriverSPI.h>
#include <MFRC522DriverPinSimple.h>
#include <HardwareSerial.h>
#include <DFRobotDFPlayerMini.h>
#include <Arduino.h>

// -------- PINES --------
#define SS1 5
#define RST1 22
#define SS2 4
#define RST2 21
#define RELE_VERDE 25
#define RELE_ROJO 26
#define MP3_BUSY_PIN 15  
#define POT_PIN 34       
#define BTN_RESET 32      

SPIClass spi1(VSPI);
SPIClass spi2(HSPI);
HardwareSerial mp3(2);
DFRobotDFPlayerMini player;

MFRC522DriverPinSimple ss1(SS1);
MFRC522DriverPinSimple ss2(SS2);
MFRC522DriverSPI driver1{ss1, spi1};
MFRC522DriverSPI driver2{ss2, spi2};
MFRC522 lectorRuleta{driver1};
MFRC522 lectorLlaveros{driver2};

// -------- CONTROL --------
String uidAnterior = "";
unsigned long tiempoCambio = 0;
unsigned long timerRespuesta = 0;    
unsigned long timerRecordatorio = 0; 
unsigned long timerVolumen = 0;   
unsigned long tiempoPresionado = 0;

int ultimoVolumen = -1;

bool reproducido = true;      
int animalActual = 0;
bool esperandoRespuesta = false;
bool modoRecordatorio = false;       
bool bloqueoInicial = true;   
bool enModo16 = false;

// ---------- FUNCIONES ----------

void esperarAudio() {
  delay(600); 
  unsigned long s = millis();
  while (digitalRead(MP3_BUSY_PIN) == HIGH && (millis() - s < 1500)) delay(10);
  while (digitalRead(MP3_BUSY_PIN) == LOW) delay(10);
  delay(300);
}

// 🔥 NUEVO VOLUMEN PRO SUAVE
void actualizarVolumen() {

  if (enModo16) return;

  static int volumenSuave = 20;
  static int volumenObjetivo = 20;

  if (millis() - timerVolumen > 100) {

    int lectura = 0;
    for (int i = 0; i < 5; i++) {
      lectura += analogRead(POT_PIN);
      delay(2);
    }
    lectura /= 5;

    volumenObjetivo = map(lectura, 0, 4095, 5, 30);

    if (volumenSuave < volumenObjetivo) volumenSuave++;
    else if (volumenSuave > volumenObjetivo) volumenSuave--;

    if (volumenSuave != ultimoVolumen) {
      player.volume(volumenSuave);
      ultimoVolumen = volumenSuave;
    }

    timerVolumen = millis();
  }
}

// 🔥 BOTÓN MAESTRO
void revisarBoton() {

  static bool estadoAnterior = HIGH;
  static unsigned long ultimoCambio = 0;
  const int debounceTime = 50; // 🔥 antirrebote (50ms)

  bool lectura = digitalRead(BTN_RESET);

  // Detectar cambio con antirrebote
  if (lectura != estadoAnterior && millis() - ultimoCambio > debounceTime) {
    ultimoCambio = millis();
    estadoAnterior = lectura;

    // -------- BOTON PRESIONADO --------
    if (lectura == LOW) {
      tiempoPresionado = millis();
    }

    // -------- BOTON SOLTADO --------
    else {

      if (tiempoPresionado != 0) {

        unsigned long duracion = millis() - tiempoPresionado;

        if (duracion < 6000 || enModo16) {
          Serial.println("Reiniciando sistema...");
          player.volume(30);
          player.play(15);
          esperarAudio();
          ESP.restart();
        }

        tiempoPresionado = 0;
      }
    }
  }

  // -------- PRESION LARGA (SIN SOLTAR) --------
  if (estadoAnterior == LOW && tiempoPresionado != 0) {

    if (millis() - tiempoPresionado > 6000 && !enModo16) {
      enModo16 = true;
      player.volume(30);
      player.play(16);
      Serial.println("Modo 16 Activado");
    }
  }
}

int animalTarjeta(String uid) {
  if (uid == "C30AC335") return 1;
  if (uid == "F3BB4D36") return 2;
  if (uid == "73FBD335") return 3;
  if (uid == "03BDC335") return 4;
  if (uid == "035FDD35") return 5;
  if (uid == "53C5501C") return 6;
  if (uid == "A3064F36") return 7;
  if (uid == "23B5D935") return 8;
  return 0;
}

int animalLlavero(String uid) {
  if (uid == "CDE90C04") return 1;
  if (uid == "5113443D") return 2;
  if (uid == "70D50D04") return 3;
  if (uid == "673B202D") return 4;
  if (uid == "03D5F70C") return 5;
  if (uid == "53B0F90C") return 6;
  if (uid == "A9AB433D") return 7;
  if (uid == "20D70D04") return 8;
  return 0;
}

String getUID(MFRC522 &reader) {
  String uid = "";
  if (!reader.PICC_ReadCardSerial()) return "";
  for (byte i = 0; i < reader.uid.size; i++) {
    if (reader.uid.uidByte[i] < 0x10) uid += "0";
    uid += String(reader.uid.uidByte[i], HEX);
  }
  uid.toUpperCase();
  return uid;
}

// ---------- SETUP ----------

void setup() {
  Serial.begin(115200);

  pinMode(RST1, OUTPUT); pinMode(RST2, OUTPUT);
  digitalWrite(RST1, HIGH); digitalWrite(RST2, HIGH);

  pinMode(RELE_VERDE, OUTPUT);
  pinMode(RELE_ROJO, OUTPUT);

  pinMode(MP3_BUSY_PIN, INPUT); 
  pinMode(POT_PIN, INPUT);
  pinMode(BTN_RESET, INPUT_PULLUP);

  spi1.begin(18, 19, 23, SS1);
  spi2.begin(14, 27, 13, SS2);

  lectorRuleta.PCD_Init();
  lectorLlaveros.PCD_Init();

  lectorRuleta.PCD_AntennaOn();
  lectorLlaveros.PCD_AntennaOff();

  mp3.begin(9600, SERIAL_8N1, 16, 17);

  if (!player.begin(mp3)) {
    while (true);
  }

  actualizarVolumen();

  player.play(9);
  esperarAudio();

  if (lectorRuleta.PICC_IsNewCardPresent()) {
    uidAnterior = getUID(lectorRuleta);
  }

  bloqueoInicial = true;
  modoRecordatorio = true;
  reproducido = true;
  timerRecordatorio = millis();
}

// ---------- LOOP ----------

void loop() {

  revisarBoton();

  if (enModo16) return;

  actualizarVolumen();

  // ================= RULETA =================
  if (!esperandoRespuesta) {

    if (digitalRead(MP3_BUSY_PIN) == HIGH) {

      if (lectorRuleta.PICC_IsNewCardPresent()) {

        String uidActual = getUID(lectorRuleta);

        if (uidActual != "" && uidActual != uidAnterior) {
          uidAnterior = uidActual;
          tiempoCambio = millis();
          reproducido = false;
          modoRecordatorio = false;
          bloqueoInicial = false;
        }
      }
    }

    if (!bloqueoInicial && !reproducido && uidAnterior != "" && (millis() - tiempoCambio > 2000)) {

      if (digitalRead(MP3_BUSY_PIN) == HIGH) {

        animalActual = animalTarjeta(uidAnterior);

        if (animalActual > 0) {

          player.play(animalActual);
          esperarAudio();

          player.play(12);

          timerRespuesta = millis();

          lectorRuleta.PCD_AntennaOff();
          lectorLlaveros.PCD_AntennaOn();

          esperandoRespuesta = true;
        }

        reproducido = true;
      }
    }

    if (modoRecordatorio && (millis() - timerRecordatorio > 5000)) {
      if (digitalRead(MP3_BUSY_PIN) == HIGH) {
        player.play(14);
        timerRecordatorio = millis();
      }
    }
  }

  // ================= LLAVEROS =================
  else {

    if (millis() - timerRespuesta > 13000) {

      player.stop();
      delay(300);

      player.play(13);
      esperarAudio();

      player.play(14);

      esperandoRespuesta = false;
      modoRecordatorio = true;
      timerRecordatorio = millis();
      bloqueoInicial = true;

      lectorLlaveros.PCD_AntennaOff();
      lectorRuleta.PCD_AntennaOn();

      return;
    }

    if (digitalRead(MP3_BUSY_PIN) == HIGH) {
      player.play(12);
      delay(200);
    }

    if (lectorLlaveros.PICC_IsNewCardPresent()) {

      String uidLlavero = getUID(lectorLlaveros);

      if (uidLlavero != "") {

        player.stop();
        delay(300);

        int respuesta = animalLlavero(uidLlavero);

        if (respuesta == animalActual) {

          player.play(10);

         digitalWrite(RELE_VERDE, HIGH);
         delay(1500);
          digitalWrite(RELE_VERDE, LOW);
          }

         else {

          player.play(11);

          digitalWrite(RELE_ROJO, HIGH);
          delay(1500);
          digitalWrite(RELE_ROJO, LOW);
        }

        esperarAudio();

        lectorLlaveros.PCD_AntennaOff();
        lectorRuleta.PCD_AntennaOn();

        esperandoRespuesta = false;
        modoRecordatorio = true;
        bloqueoInicial = true;
        timerRecordatorio = millis();
      }
    }
  }
}
