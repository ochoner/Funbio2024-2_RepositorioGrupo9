 El código a continuación es el código completo de la funcionalidad de la centrífuga.
 
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Encoder.h>
#include <Servo.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

#define ENCODER_PIN_A 2
#define ENCODER_PIN_B 3
#define ENCODER_BUTTON_PIN 4
#define ESC_PIN 9
#define LIMIT_SWITCH_PIN 5
#define IR_LED_PIN 6
#define IR_RECEIVER_PIN 7
#define BUZZER_PIN 8

Encoder encoder(ENCODER_PIN_A, ENCODER_PIN_B);
Servo ESC;

long oldPosition = -999;
int currentMenu = 0;
bool inMenu = true;
long rpmValue = 0;
int timeValue = 0; 
int maxRPM = 6000;
int maxTime = 60; 
bool motorOn = false;

int remainingTime = 0;
unsigned long lastTimeUpdate = 0;

void setup() {
  Serial.begin(9600);
  
  pinMode(ENCODER_BUTTON_PIN, INPUT_PULLUP);
  pinMode(LIMIT_SWITCH_PIN, INPUT_PULLUP);
  pinMode(IR_LED_PIN, OUTPUT);
  pinMode(IR_RECEIVER_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("Error al iniciar la pantalla OLED"));
    while (1);
  }
  display.clearDisplay();

  ESC.attach(ESC_PIN);
  ESC.writeMicroseconds(2000);
  delay(2000);
  ESC.writeMicroseconds(1000);
  delay(2000);

  // Sonido inicial
  tone(BUZZER_PIN, 800, 500);
  delay(500);
  tone(BUZZER_PIN, 1000, 500);
  delay(500);
  tone(BUZZER_PIN, 1200, 500);
  
  showWelcomeScreen();
}

void loop() {
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
    if (inMenu) {
      if (currentMenu == 3) {
        if (digitalRead(LIMIT_SWITCH_PIN) == LOW) {
          if (checkBalance()) {
            startMotor();
          } else {
            displayError("Desbalance detectado");
          }
        } else {
          displayError("Error: Tapa abierta");
        }
      } else {
        enterMenuOption(currentMenu);
      }
    } else {
      inMenu = true;
      if (currentMenu == 0 || currentMenu == 1) {
        encoder.write(0);
      }
      showMenu();
    }
    delay(500);
  }

  if (motorOn) {
    // Verificar tapa y desbalance en cada ciclo de loop
    if (digitalRead(LIMIT_SWITCH_PIN) == HIGH) {
      stopMotor("Error: Tapa abierta");
    } else if (!checkBalance()) {
      stopMotor("Desbalance detectado");
    } else if (millis() - lastTimeUpdate >= 1000) {
      lastTimeUpdate = millis();
      updateTimer();
    }
  }
}

void showWelcomeScreen() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(20, 25);
  display.println(F("Bienvenido"));
  display.display();
  delay(2000);
  showMenu();
}

void showMenu() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println(F("Menu principal"));
  display.setCursor(0, 20);
  display.println(currentMenu == 0 ? "> RPM" : "  RPM");
  display.setCursor(0, 30);
  display.println(currentMenu == 1 ? "> TIEMPO" : "  TIEMPO");
  display.setCursor(0, 40);
  display.println(currentMenu == 2 ? "> ERRORES" : "  ERRORES");
  display.setCursor(0, 50);
  display.println(currentMenu == 3 ? "> INICIAR" : "  INICIAR");
  display.display();
}

void enterMenuOption(int option) {
  inMenu = false;
  if (option == 0) {
    showRPMConfig();
  } else if (option == 1) {
    showTimeConfig();
  } else {
    displayError("Opción seleccionada");
  }
}

void showRPMConfig() {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.print(F("Configurar RPM: "));
  display.println(rpmValue);
  display.setCursor(0, 50);
  display.println(F("Presiona para salir"));
  display.display();
}

void showTimeConfig() {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.print(F("Configurar Tiempo: "));
  display.print(timeValue);
  display.println(F(" min"));
  display.setCursor(0, 50);
  display.println(F("Presiona para salir"));
  display.display();
}

long updateEncoderValue(long currentValue, long maxValue, long step) {
  long newValue = constrain(encoder.read() / 4 * step, 0, maxValue);
  return newValue;
}

void startMotor() {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.println(F("Iniciando centrifugado"));
  display.display();
  
  ESC.writeMicroseconds(map(rpmValue, 0, maxRPM, 1000, 2000));
  remainingTime = timeValue * 60;
  motorOn = true;
  lastTimeUpdate = millis();
}

void updateTimer() {
  if (remainingTime > 0) {
    remainingTime--;
    display.clearDisplay();
    display.setCursor(0, 0);
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.print(F("RPM: "));
    display.println(rpmValue);
    display.setCursor(0, 20);
    display.print(F("Tiempo restante: "));
    display.print(remainingTime / 60);
    display.print(F(":"));
    display.println(remainingTime % 60);
    display.display();
  } else {
    stopMotor("Terminó la Centrifugación");
    tone(BUZZER_PIN, 1000, 2000);  // Sonido de finalización
  }
}

void stopMotor(const char* message) {
  motorOn = false;
  ESC.writeMicroseconds(1000);
  displayError(message);
}

bool checkBalance() {
  digitalWrite(IR_LED_PIN, HIGH);
  delay(10);
  bool inBalance = digitalRead(IR_RECEIVER_PIN) == HIGH;
  digitalWrite(IR_LED_PIN, LOW);
  return inBalance;
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
