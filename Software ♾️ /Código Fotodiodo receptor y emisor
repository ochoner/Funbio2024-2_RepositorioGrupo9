
#define IR_LED_PIN 6
#define IR_RECEIVER_PIN 7
pinMode(IR_LED_PIN, OUTPUT);
pinMode(IR_RECEIVER_PIN, INPUT_PULLUP);

bool checkBalance() {
    digitalWrite(IR_LED_PIN, HIGH);
    delay(10);
    bool inBalance = digitalRead(IR_RECEIVER_PIN) == HIGH;
    digitalWrite(IR_LED_PIN, LOW);
    return inBalance;
}

// Usado en el menú de inicio para verificar balance antes de comenzar
void loop() {
    if (motorOn && !checkBalance()) {
        stopMotor("Desbalance detectado");
    }
}
