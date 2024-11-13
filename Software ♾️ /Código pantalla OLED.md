A continuación se mostrará el código que se usó para la pantalla OLED:

#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Inicialización y mensajes en la pantalla OLED
void showWelcomeScreen() { /* Código para pantalla de bienvenida */ }
void showMenu() { /* Código para mostrar menú principal */ }
void showRPMConfig() { /* Configuración de RPM en pantalla */ }
void showTimeConfig() { /* Configuración de tiempo en pantalla */ }
void displayError(const char* errorMessage) { /* Mostrar mensajes de error */ }
