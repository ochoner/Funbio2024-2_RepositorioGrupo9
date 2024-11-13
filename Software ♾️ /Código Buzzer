La función de este código será avisar mediante el sonido la inicialización, finalización del proceso de centrifugación. Además, notificará si la tapa está abierta.

#define BUZZER_PIN 8
pinMode(BUZZER_PIN, OUTPUT);

// Activar buzzer
void updateTimer() {
    if (remainingTime == 0) {
        stopMotor("Terminó la Centrifugación");
        tone(BUZZER_PIN, 1000, 2000);  // Sonido del buzzer al finalizar
    }
}

