A continuación se mostrará el código que se usó para el Encoder Rotativo.
Esta sección configura y gestiona el encoder rotativo, permitiendo al usuario navegar por el menú y configurar valores de RPM y tiempo.

#include <Encoder.h>
#define ENCODER_PIN_A 2
#define ENCODER_PIN_B 3
#define ENCODER_BUTTON_PIN 4
Encoder encoder(ENCODER_PIN_A, ENCODER_PIN_B);
long oldPosition = -999;

// Lógica de control del encoder
void loop() {
    long newPosition = encoder.read() / 4;
    if (newPosition != oldPosition) {
        // Código para cambiar de menú o ajustar configuraciones
    }
    // Leer botón del encoder para seleccionar opciones
    if (digitalRead(ENCODER_BUTTON_PIN) == LOW) { /* Código para seleccionar opción */ }
}

long updateEncoderValue(long currentValue, long maxValue, long step) {
    return constrain(encoder.read() / 4 * step, 0, maxValue);
}

