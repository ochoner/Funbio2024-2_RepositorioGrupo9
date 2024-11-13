Con este código y usando el switch se verificará cuando la tapa de la centrífuga esté cerrada:

#define LIMIT_SWITCH_PIN 5
pinMode(LIMIT_SWITCH_PIN, INPUT_PULLUP);

// Verificar si la tapa está cerrada
void loop() {
    if (motorOn && digitalRead(LIMIT_SWITCH_PIN) == HIGH) {
        stopMotor("Error: Tapa abierta");
    }
}
