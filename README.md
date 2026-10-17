# sleep_tactil — App 3 (Clase 3)

## Objetivo

Modificar el programa de la ESP32-S2-Kaluga-1 para eliminar el
temporizador de la fase activa y mostrar por el monitor serie una
tabla de memoria RAM al tocar `NETWORK`, conservando los modos activo,
light sleep y deep sleep.

La fase activa permanece esperando sin límite de tiempo. Cada toque en
`NETWORK` muestra memoria total, usada y libre en bytes, sin cambiar de
modo. La tabla corresponde al heap con capacidad `MALLOC_CAP_8BIT`;
la memoria usada se calcula como total menos libre. `PLAY` termina la
fase activa e inicia las rondas de light sleep.

El ciclo implementado es:

```text
ACTIVO → PLAY → LIGHT SLEEP (3 rondas) → DEEP SLEEP → nuevo arranque
```

Cada ronda de light sleep termina por un toque o después de 5 segundos.
El deep sleep termina al tocar RECORD o después de 30 segundos.
Al salir de light sleep continúa la ejecución y se conserva la RAM;
al salir de deep sleep se ejecuta nuevamente `app_main()` y los contadores
se conservan mediante `RTC_DATA_ATTR`.

## Conexiones

- Conectar la **ESP-LyraP-TouchA** al **Touch FPC Connector** de la Kaluga-1
  mediante el cable plano de 20 pines.
- Dejar los **microinterruptores T1–T14 en OFF**.
- Colocar el **jumper RGB** para utilizar el LED direccionable en **GPIO45**.
- Conectar el USB al puerto **UART** de la placa para flashear y monitorear.

| Botón | Canal | GPIO | Función |
|---|---|---|---|
| RECORD | T5 | 5 | Despertar del deep sleep |
| PLAY | T2 | 2 | Terminar la fase activa |
| NETWORK | T11 | 11 | Mostrar RAM total, usada y libre durante la fase activa |

Cualquiera de estos tres botones puede despertar del light sleep.
No tocar la placa durante el arranque, mientras se calibran los sensores.

## Tabla de consumo

El firmware estima el consumo a partir de los tiempos registrados y estas
corrientes de referencia del **ESP32-S2 Series Datasheet v1.9**, tablas 5-8
y 5-9, a 3,3 V. **No son mediciones de corriente de la placa.**

| Modo | Corriente de referencia | Duración configurada |
|---|---:|---|
| Activo | 23 000 µA | Hasta pulsar PLAY |
| Light sleep | 750 µA | Hasta 5 s por ronda; 3 rondas |
| Deep sleep con táctil | 22 µA | Hasta 30 s o un toque en RECORD |

El balance serie incluye tiempo por modo, carga estimada en µAh, porcentaje
del tiempo total, corriente media y autonomía estimada para una batería de
**1 000 mAh**. Las constantes están al principio de
`main/sleep_tactil_main.c`.

La medición de la placa completa incluye también el consumo del regulador,
los LEDs y los periféricos; por eso puede diferir de los valores del chip.

## Cómo compilar y flashear

Con ESP-IDF disponible en la terminal, ejecutar desde este directorio:

```bash
idf.py set-target esp32s2
idf.py build
idf.py -p /dev/cu.usbserial-XXXX flash monitor
```

Sustituir `/dev/cu.usbserial-XXXX` por el puerto de la placa; en macOS se
puede consultar con `ls /dev/cu.*`. Para salir del monitor, usar `Ctrl + ]`.

## Resultado esperado y comprobación

El comportamiento implementado se comprueba con esta secuencia en el
monitor serie. No se incluye aquí un registro de una prueba en la placa.

1. Al arrancar, aparece `[ACTIVO]`. Esperar más de los 10 segundos que
   duraba originalmente esta fase: debe permanecer activa hasta tocar PLAY.
2. Tocar **NETWORK**: aparece `--- Memoria RAM ---` y una tabla con las
   columnas `total`, `usada` y `libre`. Se puede repetir sin salir de la fase.
3. Tocar **PLAY**: aparecen los mensajes `[LIGHT SLEEP]` de las tres rondas,
   con la causa de despertar por toque o por plazo.
4. Después aparece `[DEEP SLEEP]`. Al tocar **RECORD** o cumplirse los
   30 segundos, aparece un nuevo `Arranque #`, la causa del despertar y
   el balance de consumo acumulado.

Durante el funcionamiento, el LED indica calibración en ámbar, fase activa
en verde y salida de light sleep con un destello azul. Se apaga antes de
dormir.
