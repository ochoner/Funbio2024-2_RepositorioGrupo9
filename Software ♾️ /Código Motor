A continuación se mostrará el código que se usó para el motor (con el ESC):

#include <Servo.h>
#define ESC_PIN 9
Servo ESC;
bool motorOn = false;
long rpmValue = 0;
int maxRPM = 6000;

// Control del motor
void startMotor() {
    ESC.writeMicroseconds(map(rpmValue, 0, maxRPM, 1000, 2000));
    motorOn = true;
}

void stopMotor(const char* message) {
    motorOn = false;
    ESC.writeMicroseconds(1000);  // Detener motor
    displayError(message);         // Mostrar error
}
