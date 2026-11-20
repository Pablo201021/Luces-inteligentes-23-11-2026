# Luces-inteligentes-23-11-2026
/*
  PARQUEADERO INTELIGENTE - ESP32 + Bluetooth + App Inventor
  Prof. Galo Hurtado - Interfaces de Control

  Funciones:
   1. Sensor ultrasónico HC-SR04: si un auto llega a menos del umbral,
      se ABRE la barrera (servo). A los 20 segundos se BAJA sola.
   2. Sensor de sonido (KY-037 / KY-038): con un APLAUSO se enciende
      o apaga un foco de 110 V conectado a un relé.
   3. Envía el estado a la app por Bluetooth y recibe órdenes.

  Placa: ESP32 Dev Module (ESP32-WROOM-32). C3/C6/S3 no tienen Bluetooth clásico.
  Librería necesaria: "ESP32Servo" (Kevin Harrington) desde el Gestor de librerías.

  Conexiones:
    HC-SR04   VCC -> VIN (5V)   GND -> GND
              TRIG -> GPIO 5
              ECHO -> GPIO 18  (con divisor de voltaje 1k / 2k: el ECHO da 5 V)
    Servo     Rojo -> VIN (5V)  Café -> GND   Naranja (señal) -> GPIO 13
    Sonido    VCC -> 3V3        GND -> GND    DO -> GPIO 34
    Relé      VCC -> VIN (5V)   GND -> GND    IN -> GPIO 26
              COM y NO del relé interrumpen la FASE del foco de 110 V.

  Protocolo Bluetooth
    ESP32 -> App (cada 300 ms):  distancia,puerta,segundos,foco\n
        ejemplo "12,1,17,0" = 12 cm, barrera abierta, faltan 17 s, foco apagado
    App -> ESP32:
        "U25\n"  cambia el umbral a 25 cm
        "F\n"    cambia el foco (igual que un aplauso)
*/

#include "BluetoothSerial.h"
#include <ESP32Servo.h>

#if !defined(CONFIG_BT_ENABLED) || !defined(CONFIG_BLUEDROID_ENABLED)
#error "Bluetooth no habilitado. Usa una placa ESP32 clásica (ESP32 Dev Module)."
#endif

// ---------- Pines ----------
const int PIN_TRIG   = 5;
const int PIN_ECHO   = 18;
const int PIN_SERVO  = 13;
const int PIN_SONIDO = 34;
const int PIN_RELE   = 26;

// ---------- Ajustes ----------
const bool RELE_ACTIVO_EN_ALTO = true;      // jumper del módulo en H
const int  FLANCO_SONIDO = RISING;          // si tu módulo marca al revés, usa FALLING
const int  ANGULO_CERRADA = 0;
const int  ANGULO_ABIERTA = 90;
const unsigned long TIEMPO_ABIERTA = 20000; // 20 segundos
const unsigned long PAUSA_APLAUSO  = 500;   // ignora rebotes del mismo aplauso
const unsigned long SILENCIO_SERVO = 1000;  // ignora ruido del servo al moverse

BluetoothSerial SerialBT;
Servo barrera;

int  umbral = 20;               // cm (lo cambia la app)
int  distancia = 400;
bool puertaAbierta = false;
bool esperandoSalida = false;   // no reabrir hasta que el auto se retire
bool foco = false;
int  lecturasCerca = 0;

unsigned long tApertura = 0;
unsigned long tMovServo = 0;
unsigned long tMedicion = 0;
unsigned long tEnvio = 0;

volatile bool hayAplauso = false;
volatile unsigned long tUltimoAplauso = 0;

void IRAM_ATTR detectarSonido() {
  unsigned long t = millis();
  if (t - tUltimoAplauso > PAUSA_APLAUSO) {
    tUltimoAplauso = t;
    hayAplauso = true;
  }
}

int medirDistancia() {
  digitalWrite(PIN_TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(PIN_TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(PIN_TRIG, LOW);
  long duracion = pulseIn(PIN_ECHO, HIGH, 25000);   // máx. ~4 m
  if (duracion == 0) return 400;                    // sin eco
  int cm = duracion * 0.0343 / 2;
  return constrain(cm, 2, 400);
}

void ponerFoco(bool encendido) {
  foco = encendido;
  digitalWrite(PIN_RELE, (foco == RELE_ACTIVO_EN_ALTO) ? HIGH : LOW);
  Serial.println(foco ? "Foco ENCENDIDO" : "Foco APAGADO");
}

void abrirBarrera() {
  barrera.write(ANGULO_ABIERTA);
  puertaAbierta = true;
  tApertura = millis();
  tMovServo = millis();
  Serial.println("Barrera ABIERTA");
}

void cerrarBarrera() {
  barrera.write(ANGULO_CERRADA);
  puertaAbierta = false;
  esperandoSalida = true;
  tMovServo = millis();
  Serial.println("Barrera CERRADA (20 s cumplidos)");
}

void leerBluetooth() {
  while (SerialBT.available()) {
    String cmd = SerialBT.readStringUntil('\n');
    cmd.trim();
    if (cmd.length() == 0) continue;
    if (cmd.charAt(0) == 'U') {
      int u = cmd.substring(1).toInt();
      if (u >= 3 && u <= 100) {
        umbral = u;
        Serial.println("Nuevo umbral: " + String(umbral) + " cm");
      }
    } else if (cmd.charAt(0) == 'F') {
      ponerFoco(!foco);
    }
  }
}

void enviarEstado() {
  int segundos = 0;
  if (puertaAbierta) {
    unsigned long pasado = millis() - tApertura;
    segundos = (pasado >= TIEMPO_ABIERTA) ? 0 : (TIEMPO_ABIERTA - pasado + 999) / 1000;
  }
  String linea = String(distancia) + "," + String(puertaAbierta ? 1 : 0) + "," +
                 String(segundos) + "," + String(foco ? 1 : 0);
  SerialBT.print(linea);
  SerialBT.print('\n');               // DelimiterByte = 10 en App Inventor
}

void setup() {
  Serial.begin(115200);
  pinMode(PIN_TRIG, OUTPUT);
  pinMode(PIN_ECHO, INPUT);
  pinMode(PIN_SONIDO, INPUT);
  pinMode(PIN_RELE, OUTPUT);
  ponerFoco(false);

  barrera.setPeriodHertz(50);
  barrera.attach(PIN_SERVO, 500, 2400);
  barrera.write(ANGULO_CERRADA);

  attachInterrupt(digitalPinToInterrupt(PIN_SONIDO), detectarSonido, FLANCO_SONIDO);

  SerialBT.setTimeout(50);
  SerialBT.begin("ESP32_PARQUEO");
  Serial.println("Listo. Empareja el celular con ESP32_PARQUEO");
}

void loop() {
  leerBluetooth();

  // 1) Medir distancia cada 100 ms
  if (millis() - tMedicion >= 100) {
    tMedicion = millis();
    distancia = medirDistancia();

    // 3 lecturas seguidas cerca = hay auto (evita falsos disparos)
    lecturasCerca = (distancia <= umbral) ? lecturasCerca + 1 : 0;

    if (!puertaAbierta && !esperandoSalida && lecturasCerca >= 3) {
      abrirBarrera();
    }
    // el auto ya se retiró del sensor: se puede volver a abrir
    if (esperandoSalida && distancia > umbral + 5) {
      esperandoSalida = false;
    }
  }

  // 2) Cerrar barrera a los 20 s
  if (puertaAbierta && millis() - tApertura >= TIEMPO_ABIERTA) {
    cerrarBarrera();
  }

  // 3) Aplauso -> cambia el foco (ignorando el ruido del servo)
  if (hayAplauso) {
    hayAplauso = false;
    if (millis() - tMovServo > SILENCIO_SERVO) {
      ponerFoco(!foco);
    }
  }

  // 4) Enviar estado a la app cada 300 ms
  if (millis() - tEnvio >= 300) {
    tEnvio = millis();
    enviarEstado();
  }
}
