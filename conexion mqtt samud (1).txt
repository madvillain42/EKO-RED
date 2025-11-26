#include <DHT.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

// ================================
//        CONFIG HARDWARE
// ================================
#define ANCHO 128
#define ALTO 64
Adafruit_SSD1306 pantalla(ANCHO, ALTO, &Wire, -1);

#define PIN_DHT D4
#define PIN_MQ2 A0
#define PIN_BUZZER D5
#define PIN_BOTON D7
#define DHTTYPE DHT11

DHT dht(PIN_DHT, DHTTYPE);

// ================================
//        CREDENCIALES WiFi / MQTT
// ================================
const char* ssid = "WiFi_Mesh-238936";
const char* password = "SsyNFQzy";
const char* mqtt_server = "20.171.27.160";

WiFiClient espClient;
PubSubClient client(espClient);

// ================================
//          VARIABLES
// ================================
int pantallaActual = 0;

float t_global = 0;
float h_global = 0;
int gas_global = 0;

unsigned long lastMsg = 0;
unsigned long lastDHTRead = 0;
unsigned long lastReconnectAttempt = 0;
const long INTERVALO_ENVIO = 30000; // 30 segundos

// Botón / debounce / long press
bool botonPrevio = HIGH;      // estado "estable" previo del botón
int botonRawPrev = HIGH;      // lectura cruda previa
unsigned long lastChange = 0; // último cambio de lectura cruda
const unsigned long DEBOUNCE_MS = 50;
const unsigned long LONG_PRESS_MS = 5000;
bool isPressing = false;
unsigned long pressStart = 0;

// ================================
//     FUNCIONES VISUALES
// ================================
void pantallaInicio() {
  pantalla.clearDisplay();
  for (int c = 0; c <= 255; c += 20) {
    pantalla.ssd1306_command(SSD1306_SETCONTRAST);
    pantalla.ssd1306_command(c);
    delay(8);
  }

  pantalla.setTextColor(SSD1306_WHITE);
  pantalla.setTextSize(2);
  pantalla.setCursor(20, 10);
  pantalla.println("Eko-Red");

  pantalla.setTextSize(1);
  pantalla.setCursor(5, 40);
  pantalla.println("Presiona el boton");

  pantalla.setCursor(15, 52);
  pantalla.println("para empezar");

  pantalla.display();
}

void pantallaDHT(float t, float h) {
  pantalla.clearDisplay();
  pantalla.setTextSize(2);
  pantalla.setCursor(20, 0);
  pantalla.println("DHT11");

  pantalla.setTextSize(1);
  pantalla.setCursor(0, 30);
  pantalla.print("Temp: ");
  pantalla.print(t, 1);
  pantalla.println(" C");

  pantalla.setCursor(0, 45);
  pantalla.print("Humedad: ");
  pantalla.print(h, 1);
  pantalla.println(" %");

  pantalla.display();
}

void pantallaMQ2(int gas) {
  pantalla.clearDisplay();
  pantalla.setTextSize(2);
  pantalla.setCursor(35, 0);
  pantalla.println("MQ-2");

  pantalla.setTextSize(1);
  pantalla.setCursor(0, 40);
  pantalla.print("Gas (0-1023): ");
  pantalla.println(gas);

  pantalla.display();
}

void pantallaTiempoReal(float t, float h, int gas) {
  pantalla.clearDisplay();
  pantalla.setTextSize(1);
  pantalla.setCursor(25, 0);
  pantalla.println("Tiempo Real");

  pantalla.setCursor(0, 20);
  pantalla.print("Temp: "); pantalla.print(t, 1); pantalla.println(" C");

  pantalla.setCursor(0, 33);
  pantalla.print("Hum : "); pantalla.print(h, 1); pantalla.println(" %");

  pantalla.setCursor(0, 46);
  pantalla.print("Gas : "); pantalla.println(gas);

  if (client.connected()) pantalla.fillCircle(124, 60, 2, SSD1306_WHITE);

  pantalla.display();
}

// ================================
//           BUZZER
// ================================
void beep() {
  digitalWrite(PIN_BUZZER, HIGH);
  delay(80);
  digitalWrite(PIN_BUZZER, LOW);
}

void beepLargo() {
  digitalWrite(PIN_BUZZER, HIGH);
  delay(500);
  digitalWrite(PIN_BUZZER, LOW);
}

// ================================
//          WIFI / MQTT
// ================================
void iniciarWiFi() {
  // arrancamos la conexión (no bloqueante; comprobamos en loop)
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
}

boolean intentarReconnectMQTT() {
  if (client.connected()) return true;
  String clientId = "EkoRed-" + String(random(0xffff), HEX);
  bool ok = client.connect(clientId.c_str());
  if (ok) {
    // si necesitas suscripciones, agregalas aquí
  }
  return ok;
}

// ================================
//     LECTURA DE SENSORES
// ================================
void leerSensores() {
  // gas: lectura rápida y continua
  gas_global = analogRead(PIN_MQ2);

  // DHT: cada 2 s para no bloquear
  if (millis() - lastDHTRead > 2000) {
    lastDHTRead = millis();
    float t_temp = dht.readTemperature();
    float h_temp = dht.readHumidity();
    if (!isnan(t_temp)) t_global = t_temp;
    if (!isnan(h_temp)) h_global = h_temp;
  }
}

// ================================
//               SETUP
// ================================
void setup() {
  Serial.begin(115200);

  pinMode(PIN_BUZZER, OUTPUT);
  pinMode(PIN_BOTON, INPUT_PULLUP);

  if (!pantalla.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("Pantalla OLED no detectada");
    while (true);
  }

  dht.begin();

  // iniciar wifi de forma no bloqueante
  iniciarWiFi();

  client.setServer(mqtt_server, 1883);

  pantallaInicio();
}

// Manejo no bloqueante del botón (debounce + short/long press)
void procesarBotonNonBlocking() {
  int lectura = digitalRead(PIN_BOTON); // HIGH cuando no presionado (INPUT_PULLUP)
  unsigned long ahora = millis();

  // detectar cambios en la lectura cruda
  if (lectura != botonRawPrev) {
    lastChange = ahora;
    botonRawPrev = lectura;
  }

  // si está estable por más de DEBOUNCE_MS, consideramos estado "real"
  if (ahora - lastChange > DEBOUNCE_MS) {
    if (lectura != botonPrevio) {
      // cambio de estado estable
      botonPrevio = lectura;
      if (botonPrevio == LOW) {
        // comienzo de presión
        isPressing = true;
        pressStart = ahora;
      } else {
        // se soltó
        if (isPressing) {
          unsigned long dur = ahora - pressStart;
          if (dur >= LONG_PRESS_MS) {
            // LONG PRESS (>=5s)
            beepLargo();
            pantallaActual = 0;
            pantallaInicio();
          } else {
            // SHORT PRESS
            beep();
            pantallaActual++;
            if (pantallaActual > 3) pantallaActual = 1;
            // redibujar pantallas inmediatamente en next loop
          }
          isPressing = false;
        }
      }
    } else {
      // lectura estable y sin cambio: si está presionado y supera LONG_PRESS_MS,
      // ejecutamos la acción de long press una sola vez (si no la hemos hecho todavía).
      if (botonPrevio == LOW && isPressing) {
        if ((ahora - pressStart) >= LONG_PRESS_MS) {
          // hacemos la acción de long press y simulamos "soltado" para no repetir
          beepLargo();
          pantallaActual = 0;
          pantallaInicio();
          isPressing = false;

          // Forzamos que se considere como liberado (evitamos doble activación)
          // No tocamos el pin, solo evitamos repetir la acción por software.
        }
      }
    }
  }
}

// ================================
//              LOOP
// ================================
void loop() {
  unsigned long ahora = millis();

  // 1) Si no está conectado al WiFi, reintentar conexión cada 5s (no bloqueante).
  if (WiFi.status() != WL_CONNECTED) {
    static unsigned long lastWiFiCheck = 0;
    if (ahora - lastWiFiCheck > 5000) {
      lastWiFiCheck = ahora;
      // Reiniciamos intento de conexión si hace falta
      if (WiFi.status() != WL_CONNECTED) {
        WiFi.begin(ssid, password);
      }
    }
  }

  // 2) Mantener MQTT: si wifi ok y mqtt no conectado, intentar reconnect cada 5s (no bloqueante)
  if (WiFi.status() == WL_CONNECTED) {
    if (!client.connected()) {
      if (ahora - lastReconnectAttempt > 5000) {
        lastReconnectAttempt = ahora;
        intentarReconnectMQTT();
      }
    } else {
      // mantener vivo MQTT
      client.loop();
    }
  }

  // 3) Lectura sensores (no bloqueante internamente)
  leerSensores();

  // 4) Publicar cada INTERVALO_ENVIO si mqtt conectado
  if (client.connected() && (ahora - lastMsg > INTERVALO_ENVIO)) {
    lastMsg = ahora;
    // publicamos como strings (temperatura con 1 decimal)
    char buf[32];
    snprintf(buf, sizeof(buf), "%.1f", t_global);
    client.publish("medicion/temperatura", buf);
    snprintf(buf, sizeof(buf), "%.1f", h_global);
    client.publish("medicion/humedad", buf);
    snprintf(buf, sizeof(buf), "%d", gas_global);
    client.publish("medicion/gas_ppm", buf);
  }

  // 5) Procesar botón (no bloqueante)
  procesarBotonNonBlocking();

  // 6) Actualizar pantalla según pantallaActual
  switch (pantallaActual) {
    case 0:
      // nada (pantalla estática de inicio ya dibujada)
      break;
    case 1:
      pantallaDHT(t_global, h_global);
      break;
    case 2:
      pantallaMQ2(gas_global);
      break;
    case 3:
      pantallaTiempoReal(t_global, h_global, gas_global);
      break;
  }

  // pequeña pausa cooperativa para evitar 100% CPU, sin bloquear input
  delay(20);
}
