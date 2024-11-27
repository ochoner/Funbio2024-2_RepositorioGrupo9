ESTE ES EL CÓDIGO DEL FUNCIONAMIENTO COMPLETO

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Encoder.h>
#include <Servo.h>

// OLED
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Pines
#define ENCODER_PIN_A 2
#define ENCODER_PIN_B 3
#define ENCODER_BUTTON_PIN 4
#define ESC_PIN 9
#define LIMIT_SWITCH_PIN 5
#define BUZZER_PIN 8
#define SENSOR_HALL_PIN 6

// Variables del menú y motor
Encoder encoder(ENCODER_PIN_A, ENCODER_PIN_B);
Servo ESC;
long oldPosition = -999;
int currentMenu = 0;
bool inMenu = true;
long rpmValue = 0;        // RPM objetivo
int timeValue = 0;        // Tiempo en minutos
int maxRPM = 6000;
int maxTime = 60;
bool motorOn = false;
unsigned long operationTime = 0; // Tiempo de operación en ms
unsigned long lastTimeUpdate = 0; // Tiempo en el que se inició el motor

/*aaaaaaaaaaaaaaa*/
// Variables para el sensor Hall
volatile unsigned long lastPulseTime = 0; // Tiempo del último pulso
volatile int pulseCount = 0;              // Contador de pulsos
float actualRPM = 0;                      // RPM medidas en tiempo real
const int pulsesPerRevolution = 1;        // Pulsos por revolución del motor (ajustar según el sensor/motor)
unsigned long previousMillis = 0;         // Temporizador para el cálculo de RPM
const unsigned long interval = 1000;      // Intervalo de cálculo de RPM en milisegundos

// Control de velocidad
int escSignal = 1000; // Señal inicial para el ESC (en µs)
/*aaaaaaaaaaaaaaa*/

// Configuración inicial
void setup() {  
  Serial.begin(9600);

  pinMode(ENCODER_BUTTON_PIN, INPUT_PULLUP);
  pinMode(LIMIT_SWITCH_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(SENSOR_HALL_PIN, INPUT_PULLUP);

  attachInterrupt(digitalPinToInterrupt(SENSOR_HALL_PIN), countPulse, FALLING); // Interrupción en flanco de bajada

  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("Error al iniciar la pantalla OLED"));
    while (1);
  }
  display.clearDisplay();

  ESC.attach(ESC_PIN);
  ESC.writeMicroseconds(1000); // Inicializa el ESC
  delay(2000);

  tone(BUZZER_PIN, 523, 500); // Tono de inicio
  delay(250);
  tone(BUZZER_PIN, 659, 500);
  delay(250);

  showWelcomeScreen();
}

void loop() {
  handleEncoderInput(); // Manejar la interacción con el menú
  if (motorOn) {
    controlMotor(); // Control del motor en tiempo real
  }
}

void handleEncoderInput() {
  long newPosition = encoder.read() / 4;

  if (newPosition != oldPosition) {
    oldPosition = newPosition;
    if (inMenu) {
      currentMenu = constrain(newPosition, 0, 3);
      showMenu();
    } else {
      if (currentMenu == 0) {
        rpmValue = updateEncoderValue(rpmValue, maxRPM, 100);
        showRPMConfig();
      } else if (currentMenu == 1) {
        timeValue = updateEncoderValue(timeValue, maxTime, 1);
        showTimeConfig();
      }
    }
  }

  if (digitalRead(ENCODER_BUTTON_PIN) == LOW) {
    delay(500); // Antirrebote
    if (inMenu) {
      if (currentMenu == 3) { // Iniciar motor
        if (digitalRead(LIMIT_SWITCH_PIN) == LOW) {
          startMotor();
        } else {
          displayError("Error: Tapa abierta");
        }
      } else {
        enterMenuOption(currentMenu);
      }
    } else {
      inMenu = true;
      showMenu();
    }
  }
}

void showWelcomeScreen() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(20, 25);
  display.println(F("Bienvenido"));
  display.display();
  delay(2000);
  showMenu();
}

void showMenu() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println(F("Menu principal"));
  display.setCursor(0, 20);
  display.println(currentMenu == 0 ? "> RPM" : "  RPM");
  display.setCursor(0, 30);
  display.println(currentMenu == 1 ? "> TIEMPO" : "  TIEMPO");
  display.setCursor(0, 50);
  display.println(currentMenu == 3 ? "> INICIAR" : "  INICIAR");
  display.display();
}

void showRPMConfig() {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.print(F("Configurar RPM: "));
  display.println(rpmValue);
  display.display();
}

void showTimeConfig() {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.print(F("Configurar Tiempo: "));
  display.print(timeValue);
  display.println(F(" min"));
  display.display();
}

long updateEncoderValue(long currentValue, long maxValue, long step) {
  return constrain(currentValue + step * ((encoder.read() / 4) - oldPosition), 0, maxValue);
}

void enterMenuOption(int menu) {
  if (menu == 0) {
    inMenu = false;
    showRPMConfig();
  } else if (menu == 1) {
    inMenu = false;
    showTimeConfig();
  }
}

void startMotor() {
  if (!checkBalance()) {  // Verificar balance antes de iniciar el motor
    displayError("Error: Desbalance detectado");
    return;
  }

  motorOn = true;
  operationTime = timeValue * 60000;
  lastTimeUpdate = millis(); // Guarda el momento de inicio
  previousMillis = millis();
  lastPulseTime = millis();
  pulseCount = 0;
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println(F("Motor Iniciado"));
  display.display();
}

void controlMotor() {
  unsigned long currentMillis = millis();

  // Calcular RPM cada intervalo
  if (currentMillis - previousMillis >= interval) {
    noInterrupts(); // Desactivar interrupciones temporalmente
    float elapsedTime = currentMillis - lastPulseTime;
    actualRPM = (pulseCount * 60000.0) / (pulsesPerRevolution * elapsedTime); // Calcular RPM
    pulseCount = 0; // Reiniciar contador de pulsos
    interrupts(); // Reactivar interrupciones

    // Control proporcional para ajustar la velocidad
    int error = rpmValue - actualRPM;             // Diferencia entre deseadas y actuales
    escSignal = constrain(escSignal + error * 0.05, 1000, 2000); // Ajuste proporcional suave
    ESC.writeMicroseconds(escSignal);            // Aplicar señal ajustada al ESC

    // Actualizar pantalla OLED
    display.clearDisplay();
    display.setCursor(0, 0);
    display.print(F("RPM Actual: "));
    display.println(actualRPM);
    display.print(F("RPM Objetivo: "));
    display.println(rpmValue);
    display.display();
  }

  // Verificar si el tiempo de operación ha expirado
  if (currentMillis - lastTimeUpdate >= operationTime) {
    stopMotor("Ciclo Finalizado");
  }
}

void stopMotor(const char* message) {
  motorOn = false;
  ESC.writeMicroseconds(1000); // Señal mínima para detener el motor
  displayError(message);
}

bool checkBalance() {
  int sensorValue = analogRead(PHOTO_SENSOR_PIN); // Leer el valor del fotodiodo
  Serial.print("Valor del sensor: ");
  Serial.println(sensorValue);  // Imprimir el valor del sensor para depuración
  
  // Si el valor del sensor es suficientemente bajo, indica que el fotodiodo está recibiendo luz infrarroja
  if (sensorValue < 1000) {  // Ajusta este umbral según el comportamiento de tu sensor
    return true;  // Balance correcto, luz infrarroja detectada
  } else {
    return false;  // Desbalance detectado, no se detecta luz infrarroja
  }
}

void displayError(const char* errorMessage) {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.println(errorMessage);
  display.display();
  
  // Sonido de error
  tone(BUZZER_PIN, 400, 300);
  delay(300);
  tone(BUZZER_PIN, 300, 300);
  
  delay(3000);
  showMenu();
}

void countPulse() {
  pulseCount++;
  lastPulseTime = millis();
}
