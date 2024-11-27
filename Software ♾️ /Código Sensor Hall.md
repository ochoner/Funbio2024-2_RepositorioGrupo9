#include <Servo.h> // Librería para controlar el ESC

// Pines
const int sensorHallPin = 2; // Pin digital para el sensor Hall
const int escPin = 9;        // Pin PWM para el ESC

// Variables
volatile unsigned long lastPulseTime = 0; // Tiempo del último pulso del sensor Hall
volatile int pulseCount = 0;              // Cuenta de pulsos
float rpm = 0;                            // RPM calculados
unsigned long previousMillis = 0;         // Para temporización
unsigned long startMillis = 0;            // Marca de tiempo al iniciar
const unsigned long interval = 1000;      // Intervalo de cálculo de RPM en ms
int targetRPM = 100;                     // RPM deseados
int escSignal = 1200;                     // Señal inicial para ESC (en µs, mínimo 1000)

// Constantes
const int pulsesPerRevolution = 1; // Pulsos por vuelta (ajustar según tu motor/sensor)

// Objeto Servo para manejar el ESC
Servo esc;

// Tiempo de operación (ingresado por el usuario)
unsigned long operationTime = 0; // Tiempo en milisegundos

// Configuración inicial
void setup() {
  pinMode(sensorHallPin, INPUT_PULLUP); // Configura el sensor Hall con resistencia pull-up
  attachInterrupt(digitalPinToInterrupt(sensorHallPin), countPulse, FALLING); // Interrupción en flanco de bajada
  esc.attach(escPin); // Configura el pin del ESC para señales PWM
  Serial.begin(9600); // Inicia comunicación serie
  Serial.println("Introduce el tiempo de operación en segundos:");

  // Esperar a que el usuario introduzca el tiempo de operación
  while (!Serial.available()) {
    // Espera entrada
  }
  operationTime = Serial.parseInt() * 1000; // Leer y convertir a milisegundos
  Serial.print("Tiempo de operación configurado a: ");
  Serial.print(operationTime / 1000);
  Serial.println(" segundos.");

  esc.writeMicroseconds(1000); // Inicializa el ESC con el mínimo (calibración inicial)
  delay(2000); // Espera para asegurar que el ESC esté listo
}

// Bucle principal
void loop() {
  unsigned long currentMillis = millis();

  // Verifica si el tiempo de operación ha expirado
  if (startMillis == 0) {
    startMillis = currentMillis; // Registra el inicio del tiempo de operación
  } else if (currentMillis - startMillis >= operationTime) {
    Serial.println("Tiempo de operación finalizado. Deteniendo motor.");
    esc.writeMicroseconds(1000); // Detener el motor
    while (true) {
      // Detener el programa aquí
    }
  }

  // Calcula los RPM cada 'interval' ms
  if (currentMillis - previousMillis >= interval) {
    noInterrupts(); // Desactiva interrupciones temporalmente
    unsigned long elapsedTime = currentMillis - lastPulseTime;
    float timePerPulse = elapsedTime / (float)pulseCount;
    rpm = (pulseCount / (float)pulsesPerRevolution) * (60000.0 / timePerPulse); // Convierte pulsos en RPM
    pulseCount = 0; // Reinicia el contador de pulsos
    interrupts(); // Reactiva interrupciones

    Serial.print("RPM: ");
    Serial.println(rpm);

    // Control de velocidad con transición gradual
    int error = targetRPM - rpm;                // Calcula el error
    escSignal += error * 0.05;                  // Reduce la ganancia proporcional
    escSignal = constrain(escSignal, 1000, 2000); // Limita el rango entre 1000 y 2000 µs
    int currentSignal = esc.readMicroseconds(); // Lee la señal actual aplicada al ESC
    int smoothSignal = (currentSignal + escSignal) / 2; // Suaviza la transición
    esc.writeMicroseconds(smoothSignal);        // Aplica la señal suavizada al ESC

    previousMillis = currentMillis; // Actualiza el tiempo de referencia
  }
}

// Función de interrupción para contar pulsos
void countPulse() {
  pulseCount++;
  lastPulseTime = millis();
}
