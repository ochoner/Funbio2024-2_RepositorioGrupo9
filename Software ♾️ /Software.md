#include <Encoder.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_GFX.h>
#include <TimerOne.h>

// Pines de conexión
#define ENCODER_CLK 2
#define ENCODER_DT 3
#define BUTTON_PIN 4
#define HALL_SENSOR_PIN 5
#define BUZZER_PIN 6
#define LID_SENSOR_PIN 7
#define ESC_PIN 9

// Configuración del encoder y la pantalla OLED
Encoder encoder(ENCODER_CLK, ENCODER_DT);
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// Variables del menú
int menuOption = 0;
bool buttonPressed = false;
int rpmSetting = 0; // RPM configurados
int timeSetting = 0; // Tiempo configurado en minutos
int currentRPM = 0;
int elapsedTime = 0;
bool isRunning = false;
 TimerOne myTimer;

void setup() {
    pinMode(BUTTON_PIN, INPUT_PULLUP);
    pinMode(HALL_SENSOR_PIN, INPUT);
    pinMode(BUZZER_PIN, OUTPUT);
    pinMode(LID_SENSOR_PIN, INPUT_PULLUP);
    pinMode(ESC_PIN, OUTPUT);

    // Inicializar pantalla OLED
    display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
    display.clearDisplay();
    display.display();

    // Mensaje de bienvenida
    display.setTextSize(1);
    display.setTextColor(WHITE);
    display.setCursor(0, 0);
    display.print("Bienvenido!");
    display.display();
    delay(5000); // Pausa de 5 segundos en la pantalla de bienvenida

    // Configurar el encoder
    encoder.write(0);
     // Crea una instancia de TimerOne


    myTimer.initialize(1000000); // 1 segundo
    myTimer.attachInterrupt(timerISR); // Temporizador para conteo


}

void loop() {
    // Leer el valor del encoder
    int16_t encoderPosition = encoder.read() / 4;

    if (!isRunning) {
        if (encoderPosition != menuOption) {
            menuOption = encoderPosition;
            if (menuOption < 0) menuOption = 0;
            if (menuOption > 3) menuOption = 3;
            displayMenu(menuOption);
        }

        if (digitalRead(BUTTON_PIN) == LOW && !buttonPressed) {
            buttonPressed = true;
            handleMenuSelection(menuOption);
            delay(300);
        }

        if (digitalRead(BUTTON_PIN) == HIGH) {
            buttonPressed = false;
        }
    }
    // Mostrar RPM y tiempo restante durante centrifugación
    else {
        monitorCentrifugation();
    }
}

// Función de interrupción del temporizador
void timerISR() {
    if (isRunning && currentRPM >= rpmSetting) {
        elapsedTime++;
        if (elapsedTime >= timeSetting * 60) {
            isRunning = false;
            display.clearDisplay();
            display.setCursor(0, 0);
            display.print("Terminó la Centrifugación");
            display.display();
            tone(BUZZER_PIN, 1000, 2000); // Sonido del buzzer al finalizar
        }
    }
}

// Función para mostrar el menú
void displayMenu(int option) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.setTextColor(WHITE);

    if (option == 0) display.print("> RPM");
    else display.print("  RPM");

    display.setCursor(0, 16);
    if (option == 1) display.print("> Tiempo");
    else display.print("  Tiempo");

    display.setCursor(0, 32);
    if (option == 2) display.print("> Errores");
    else display.print("  Errores");

    display.setCursor(0, 48);
    if (option == 3) display.print("> Iniciar");
    else display.print("  Iniciar");

    display.display();
}

// Función para manejar la selección del menú
void handleMenuSelection(int option) {
    if (option == 0) {
        rpmSetting = adjustRPM();
    } else if (option == 1) {
        timeSetting = adjustTime();
    } else if (option == 2) {
        displayErrors();
    } else if (option == 3) {
        startCentrifugation();
    }
}

// Ajuste de RPM usando el encoder
int adjustRPM() {
    int rpm = 0;
    while (digitalRead(BUTTON_PIN) == HIGH) {
        rpm = (encoder.read() / 4) * 100;
        if (rpm < 0) rpm = 0;
        if (rpm > 6000) rpm = 6000;
        
        display.clearDisplay();
        display.setCursor(0, 0);
        display.print("RPM: ");
        display.print(rpm);
        display.display();
        delay(100);
    }
    encoder.write(rpm / 100 * 4);
    return rpm;
}

// Ajuste de tiempo usando el encoder
int adjustTime() {
    int time = 0;
    while (digitalRead(BUTTON_PIN) == HIGH) {
        time = encoder.read() / 4;
        if (time < 0) time = 0;
        if (time > 60) time = 60;
        
        display.clearDisplay();
        display.setCursor(0, 0);
        display.print("Tiempo: ");
        display.print(time);
        display.print(" min");
        display.display();
        delay(100);
    }
    encoder.write(time * 4);
    return time;
}

// Mostrar errores
void displayErrors() {
    display.clearDisplay();
    display.setCursor(0, 0);
    display.print("Verifica:");
    display.setCursor(0, 16);
    if (digitalRead(LID_SENSOR_PIN) == HIGH) {
        display.print("- Tapa cerrada");
    } else {
        display.print("- ERROR: Tapa abierta");
    }
    display.setCursor(0, 32);
    if (digitalRead(HALL_SENSOR_PIN) == LOW) {
        display.print("- Balanceo correcto");
    } else {
        display.print("- ERROR: Desequilibrado");
    }
    display.display();
    delay(3000);
}

// Iniciar centrifugación
void startCentrifugation() {
    if (digitalRead(LID_SENSOR_PIN) == LOW) {
        display.clearDisplay();
        display.setCursor(0, 0);
        display.print("ERROR: Tapa abierta");
        display.display();
        delay(2000);
        return;
    }
    if (digitalRead(HALL_SENSOR_PIN) == HIGH) {
        display.clearDisplay();
        display.setCursor(0, 0);
        display.print("ERROR: Desequilibrio");
        display.display();
        delay(2000);
        return;
    }

    elapsedTime = 0;
    isRunning = true;
    analogWrite(ESC_PIN, map(rpmSetting, 0, 6000, 0, 255));
}

// Monitorear centrifugación
void monitorCentrifugation() {
    display.clearDisplay();
    display.setCursor(0, 0);
    display.print("RPM actual: ");
    display.print(currentRPM);

    display.setCursor(0, 16);
    display.print("Tiempo: ");
    display.print((timeSetting * 60 - elapsedTime) / 60);
    display.print(" min");
    display.display();
}
