| Supported Targets | ESP32-S2 |
| ----------------- | -------- |

# sleep_tactil — App 3 (Clase 3)

Firmware base de la **App 3** de la Clase 3. La **ESP32-S2-Kaluga-1** recorre
los tres modos de energía —**activo**, ***light sleep*** y ***deep sleep***—
despertando por los **botones táctiles** de la **ESP-LyraP-TouchA**, y publica
el **balance de consumo** de cada ciclo por el monitor serie.

```
   ACTIVO 10 s ──▶ LIGHT SLEEP ──(toque o plazo)──▶ ACTIVO ──▶ …
       │                                                        │
       └────── tras 3 rondas ──▶ DEEP SLEEP ──(toque RECORD)────┘
                                     │
                              reinicia desde app_main()
```

La diferencia que hay que ver con los propios ojos:

- de ***light sleep*** se vuelve **a la instrucción siguiente**, con la RAM y
  las tareas intactas;
- de ***deep sleep*** se vuelve **a `app_main()`**, y lo único que sobrevive es
  lo marcado `RTC_DATA_ATTR`.

## Hardware

- **ESP-LyraP-TouchA** conectada al **Touch FPC Connector** de la Kaluga-1
  con el cable plano de 20 pines.
- **Microinterruptores T1–T14** de la cara inferior en **OFF**.
- **LED RGB direccionable** en **GPIO45**.
  ⚠️ **Coloca el jumper RGB** en la placa: sin él, el LED no recibe señal.
- Target ESP-IDF: **`esp32s2`** (touch sensor **V2**).

## Mapa de botones

| Serigrafía | Canal | GPIO | Acción |
|---|:-:|:-:|---|
| **RECORD** | T5 | 5 | **Despierta del *deep sleep*** (el "botón de encendido") |
| **PLAY** | T2 | 2 | Fuerza el paso al modo siguiente |
| **VOL_UP** | T1 | 1 | Prolonga la fase activa otros 10 s |
| Cualquiera de los tres | — | — | Despierta del *light sleep* |

> En *deep sleep* solo el **canal designado** puede despertar el chip: es el
> primero de la tabla `BOTONES[]` en el código. En *light sleep* sirve
> cualquier canal habilitado.

## Colores del LED

| Color | Significa |
|---|---|
| **Ámbar** | Arrancando y calibrando |
| **Verde** | Fase activa |
| **Azul** (destello) | Acaba de despertar de *light sleep* |
| **Apagado** | Durmiendo — a propósito: 20 mA de LED arruinan cualquier medida |

## Estructura

```
sleep_tactil/
├── CMakeLists.txt
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml      Declara la dependencia espressif/led_strip
│   └── sleep_tactil_main.c
└── README.md   (este archivo)
```

## Cómo compilar, flashear y monitorear

```bash
idf.py set-target esp32s2
idf.py build
idf.py -p /dev/cu.usbserial-XXXX flash monitor
```

- Sustituye `/dev/cu.usbserial-XXXX` por tu puerto (ver `ls /dev/cu.*` en macOS).
- Conecta el cable al puerto **UART** de la placa.
- Salir del monitor: `Ctrl + ]`.

> El puerto serie **no desaparece** al dormir: lo sirve el FT2232 de la placa,
> que se alimenta del USB y sigue vivo aunque el ESP32-S2 esté en *deep sleep*.

## Qué hace, paso a paso

1. **Incrementa `s_arranques`** (en `RTC_DATA_ATTR`) e **informa de la causa
   del despertar** con `esp_sleep_get_wakeup_causes()`. Si la causa es
   `TOUCHPAD`, imprime el canal con `esp_sleep_get_touchpad_wakeup_status()`.
2. **Mide cuánto durmió** comparando `gettimeofday()` con la marca guardada en
   RTC antes de dormirse: el reloj del sistema se recupera del temporizador
   RTC, que sigue contando con el chip apagado.
3. **Publica el balance acumulado** de los ciclos anteriores.
4. **Calibra el táctil** midiendo el *benchmark* real de cada canal
   (`active_thresh = benchmark × 2 %`), igual que en la Clase 2.
5. **Configura el táctil como fuente de *wake-up*** con
   `touch_sensor_config_sleep_wakeup()`, designando el canal de *deep sleep*.
6. **Recorre el ciclo**: fase activa → tres rondas de *light sleep* →
   *deep sleep*.

> ⚠️ **No toques la placa durante el arranque.** Si el `benchmark` inicial se
> mide con el dedo apoyado, ese botón no se activará hasta el siguiente reset.

## Salida esperada

```
I (312) sleep_tactil: === Arranque #4 ===
I (312) sleep_tactil: Dormido en deep sleep: 18342 ms
I (313) sleep_tactil: Causa del despertar: TACTIL (canal T5, RECORD)
I (314) sleep_tactil: --- Balance de energia acumulado (4 arranques) ---
I (315) sleep_tactil:   activo           41,2 s  x  23000 uA  =     263,222 uAh  ( 29,0 %)
I (316) sleep_tactil:   light sleep      28,7 s  x    750 uA  =       5,979 uAh  ( 20,2 %)
I (317) sleep_tactil:   deep sleep       72,1 s  x     22 uA  =       0,440 uAh  ( 50,7 %)
I (318) sleep_tactil:   TOTAL           142,0 s                =     269,642 uAh
I (319) sleep_tactil:   corriente media : 6835 uA  (6,835 mA)
I (320) sleep_tactil:   autonomia con 1000000 uAh: 146 h  (6,0 dias)
I (321) sleep_tactil:   despertares: 3 por tactil, 5 por timer
I (322) sleep_tactil:   (corrientes del datasheet, no medidas: ver el README)
W (330) touch: Keep the RTC_PERIPH power on in case the false trigger
I (335) sleep_tactil: Calibrando... NO toques la placa durante el arranque
I (352) sleep_tactil:   RECORD   T5   benchmark=41230  umbral=824
I (353) sleep_tactil:   PLAY     T2   benchmark=40887  umbral=817
I (354) sleep_tactil:   VOL_UP   T1   benchmark=39954  umbral=799
I (356) sleep_tactil: Canal de deep sleep: RECORD (T5, GPIO5)
I (357) sleep_tactil: [ACTIVO] 10000 ms  ·  PLAY = avanzar ya  ·  VOL_UP = prolongar
I (2104) sleep_tactil:   toque en PLAY (T2), latencia ISR->tarea 37 us
I (2105) sleep_tactil: [LIGHT SLEEP] ronda 1/3, hasta 5000 ms o hasta que toques
I (7108) sleep_tactil:   vencio el PLAZO tras 5001 ms
```

El aviso `W (330) touch: Keep the RTC_PERIPH power on…` es **normal y
correcto**: en el ESP32-S2 el driver mantiene alimentado el dominio RTC
aunque se le pida apagarlo, para evitar disparos falsos del táctil.

## Los parámetros que vas a tocar

Están todos juntos al principio del `.c`:

| Constante | Qué es | Valor |
|---|---|---|
| `ACTIVO_MS` | Duración de la fase activa | 10 000 ms |
| `LIGHT_SLEEP_MS` | Plazo máximo de cada *light sleep* | 5 000 ms |
| `RONDAS_LIGHT` | Cuántos *light sleep* antes del *deep sleep* | 3 |
| `DEEP_SLEEP_S` | Plazo de respaldo del *deep sleep* | 30 s |
| `I_ACTIVO_UA` | Corriente en activo, del datasheet | 23 000 µA |
| `I_LIGHT_SLEEP_UA` | Corriente en *light sleep* | 750 µA |
| `I_DEEP_SLEEP_UA` | Corriente en *deep sleep* con táctil | 22 µA |
| `BATERIA_UAH` | Capacidad para estimar autonomía | 1 000 000 µAh |

> Las corrientes salen del **ESP32-S2 Series Datasheet v1.9**, tablas 5-8
> (Modem-sleep) y 5-9 (modos de bajo consumo), a 3,3 V. Están en **enteros**
> a propósito: el Xtensa LX7 del ESP32-S2 **no tiene FPU**, así que cada
> operación con `float` la emula el compilador en software.

## Sobre medir el consumo de verdad

Un multímetro en serie con el conector de batería mide **la placa entera**:
el FT2232, el regulador, el LED de encendido y las placas de extensión **no
duermen**. En *deep sleep* verás **miliamperios**, no los 22 µA del chip.

Eso no invalida la medida: **es el resultado**. La App 3 pide una estimación
a partir del datasheet y de los tiempos medidos, y la medida directa —si la
haces— sirve para **explicar la diferencia**, que es donde está el aprendizaje.

| Si quieres acercarte al dato del chip | Cómo |
|---|---|
| Quitar el consumo del puente USB | Alimentar por el conector de batería, sin el USB de datos |
| Quitar el LED RGB | Retirar el *jumper* RGB |
| Quitar las extensiones | Desconectar el cable FPC (pierdes el táctil) |

## Cosas para probar

| Experimento | Qué observar |
|---|---|
| Subir `ACTIVO_MS` a 30 000 | La corriente media se dispara: manda el ciclo de trabajo |
| Bajar `ACTIVO_MS` a 2 000 | La media baja mucho más de lo que parece |
| `RONDAS_LIGHT = 0` | Se va directo a *deep sleep*: media mínima, respuesta peor |
| Poner un `ESP_LOGI` después de `esp_deep_sleep_start()` | No se ejecuta jamás: la función es `noreturn` |
| Quitar `preparar_sueno()` | El último mensaje sale cortado y el LED queda encendido |
| Declarar `s_arranques` sin `RTC_DATA_ATTR` | Vuelve a 1 en cada despertar: se perdió la RAM |

## Solución de problemas

| Síntoma | Solución |
|---|---|
| No despierta al tocar en *deep sleep* | Usa **RECORD (T5)**, que es el canal designado |
| Despierta solo, sin tocar | Subir `UMBRAL_RATIO` a `0.03f`–`0.05f` |
| No despierta nunca por táctil | Bajar `UMBRAL_RATIO` a `0.01f`; recalibrar sin tocar |
| Un canal siempre activo | Se calibró con el dedo encima: reiniciar sin tocar |
| Ningún botón responde | Microinterruptores **T1–T14 en OFF**; cable FPC bien insertado |
| El LED no enciende | Colocar el **jumper RGB**; verificar GPIO45 |
| Los mensajes salen cortados al dormir | Falta `uart_wait_tx_idle_polling()` antes de dormir |
| No compila (`touch_sens.h`) | Requiere **ESP-IDF ≥ v5.4**; se probó con **v6.0.2** |
| `Failed to connect` al flashear | Secuencia **BOOT + RST** (modo bootloader) |

## Referencias

- [../../detalles/sleep-y-consumo-esp32s2.md](../../detalles/sleep-y-consumo-esp32s2.md)
- [../../../clase-2/detalles/touch-capacitivo-esp32s2.md](../../../clase-2/detalles/touch-capacitivo-esp32s2.md)
- [../../../clase-2/detalle-punto-fijo-y-float.md](../../../clase-2/detalle-punto-fijo-y-float.md)
- Espressif, *ESP32-S2 Series Datasheet* v1.9, §5.6 *Current Consumption*.
- Espressif, *ESP32-S2 Technical Reference Manual* v1.3, cap. 9
  *Low-Power Management*.
- Espressif, *ESP-IDF Programming Guide — Sleep Modes (ESP32-S2)*.
- Guía de conexión e instalación:
  [../../../bibliografia/Guia-Kaluga-1-macOS-ESP-IDF-VSCode.md](../../../bibliografia/Guia-Kaluga-1-macOS-ESP-IDF-VSCode.md)
