Análisis de los archivos fuente
1. startup_stm32f103rbtx.s (Código de inicio / Ensamblador)
Este archivo contiene el código de bajo nivel que el microcontrolador ejecuta inmediatamente al recibir energía o ser reiniciado.

Tabla de vectores de interrupción (g_pfnVectors): Define dónde están ubicados en memoria los punteros a todas las rutinas de servicio de interrupción (ISRs) y fallos del sistema. El primer valor es el puntero a la pila (_estack) y el segundo es el Reset_Handler.

Reset_Handler: Es el punto de entrada real del microcontrolador. Sus tareas principales son llamar a la función SystemInit para la configuración inicial del reloj, copiar las variables inicializadas (sección .data) desde la memoria Flash a la SRAM, inicializar en cero las variables no inicializadas (sección .bss), y finalmente saltar a la función main() del código C.

2. main.c (Programa principal)
Este archivo contiene la lógica principal de la aplicación y la inicialización de hardware.

Inicialización Base: Comienza llamando a HAL_Init() para restablecer los periféricos y configurar el temporizador del sistema (SysTick).

Configuración del Reloj (SystemClock_Config): Configura el reloj principal del sistema utilizando el oscilador interno de alta velocidad (HSI) y el bucle de bloqueo de fase (PLL) para multiplicar la frecuencia.

Inicialización de Periféricos: Llama a MX_GPIO_Init() para configurar los pines de entrada/salida (como el botón en B1_Pin y el LED en LD2_Pin) y a MX_USART2_UART_Init() para configurar la comunicación serial a 115200 baudios.

Bucle de la Aplicación: Tras inicializar componentes de usuario con app_init(), entra en un bucle infinito while (1) donde ejecuta continuamente app_update().

3. stm32f1xx_it.c (Manejadores de Interrupciones)
Este archivo aísla las funciones que responden a los eventos de interrupción (ISRs).

Excepciones del Sistema: Contiene manejadores vacíos o con bucles infinitos para fallos críticos como HardFault_Handler, MemManage_Handler, etc., que detienen el sistema si ocurre un error grave.

SysTick_Handler: Se ejecuta periódicamente (generalmente cada 1 ms) y llama a HAL_IncTick(), lo que incrementa el contador de tiempo global usado por la librería HAL para funciones de retraso y tiempos de espera (timeouts).

EXTI15_10_IRQHandler: Maneja las interrupciones externas provenientes de los pines 10 al 15. En este caso, se encarga de capturar el evento del botón conectado a B1_Pin e invoca a HAL_GPIO_EXTI_IRQHandler(B1_Pin) para procesarlo.

Evolución de SysTick y SystemCoreClock
A continuación se detalla cómo evolucionan el temporizador del sistema (SysTick) y la variable global que almacena la frecuencia del reloj (SystemCoreClock) a lo largo del arranque:

Reinicio (Reset_Handler en startup_stm32f103rbtx.s):

SystemCoreClock: Al ejecutarse la función SystemInit (llamada desde el ensamblador), el reloj se configura a su valor por defecto de hardware. Para la familia STM32F1, este suele ser el oscilador HSI corriendo a 8 MHz.

SysTick: En esta etapa, el temporizador SysTick está deshabilitado y apagado.

Inicio del programa (HAL_Init() en main.c):

SystemCoreClock: Permanece en su valor por defecto inicial (e.g., 8 MHz).

SysTick: HAL_Init() activa el SysTick. Se configuran sus registros de recarga (RELOAD) basados en la frecuencia actual del sistema para que genere una interrupción exactamente cada 1 milisegundo. A partir de este momento, el SysTick_Handler comienza a dispararse periódicamente.

Configuración avanzada (SystemClock_Config() en main.c):

SystemCoreClock: En esta función, el reloj cambia drásticamente. El código configura el oscilador para usar la fuente HSI dividida por 2 y multiplicada por 16 en el PLL (RCC_PLLSOURCE_HSI_DIV2 y RCC_PLL_MUL16). Asumiendo un HSI de 8 MHz, el cálculo es: (8 MHz / 2) * 16 = 64 MHz. Al llamar a HAL_RCC_ClockConfig(), la HAL actualiza internamente la variable global SystemCoreClock al nuevo valor de 64,000,000 Hz.

SysTick: Debido a que la frecuencia de la CPU acaba de saltar de 8 MHz a 64 MHz, si el SysTick no se ajustara, empezaría a contar mucho más rápido. La función HAL_RCC_ClockConfig() reconfigura automáticamente los registros del SysTick para adaptarse al nuevo SystemCoreClock de 64 MHz, garantizando que la interrupción siga ocurriendo de manera precisa cada 1 ms.

Bucle principal (while (1) en main.c):

SystemCoreClock: Permanece constante en 64 MHz a menos que se invoque una reconfiguración explícita del hardware.

SysTick: Funciona ininterrumpidamente en segundo plano, ejecutando HAL_IncTick() cada 1 ms exacto, proporcionando una base de tiempo estable para las lógicas de la aplicación dentro de app_update().
