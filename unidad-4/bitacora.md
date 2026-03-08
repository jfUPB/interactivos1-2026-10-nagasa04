# Unidad 4

## Bitácora de proceso de aprendizaje

### Actividad 01

### Descripción del caso

Nos entregaron un dispositivo (un micro:bit con firmware fijo) que transmite por UART (serial) sus datos de sensores (acelerómetro y botones) en un protocolo de texto sencillo. El objetivo es estudiar este flujo y preparar la adaptación de una pieza de arte generativo en p5.js para que sea controlada por este hardware.

El firmware del micro:bit (que no podemos cambiar) funciona así:

```python
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = "{},{},{},{}\n".format(xValue, yValue, aState,bState)
    uart.write(data)
    sleep(100) # Envia datos a 10 Hz
```

### Protocolo de comunicación

Cada 100 ms el dispositivo envía una línea con 4 valores separados por comas (CSV) y terminada en salto de línea (`\n`).

- `xValue` y `yValue`: valores del acelerómetro en los ejes X e Y (rango aproximado -1024..1024).
- `aState`: `True`/`False` según si el botón A está presionado.
- `bState`: `True`/`False` según si el botón B está presionado.

Ejemplo de línea:

```
-123,456,True,False
```

### Herramientas disponibles

- **SerialTerminal**: para abrir el puerto serie y visualizar los datos que envía el micro:bit.
- **p5.js**: pieza de arte generativo que vamos a adaptar para que responda a estos datos.



### Bitácora de aplicación 

### Actividad 02

### Qué se hizo

### 1) Crear un nuevo adaptador (`MicrobitV2Adapter.js`)

- Lee el stream serial y arma líneas usando `\n` como delimitador.
- Parseo robusto de la trama `$...` usando split y key:value.
- Calcula el checksum y descarta las tramas corruptas.
- Emite exactamente el mismo objeto JSON que espera el resto del sistema:
  `{ x: number, y: number, btnA: bool, btnB: bool }`.

### 2) Mantener la arquitectura (no se modificó `bridgeServer.js` ni `bridgeClient.js`)

Para que el servidor siga funcionando sin cambios, `MicrobitASCIIAdapter.js` ahora
exporta (rebautiza) a `MicrobitV2Adapter`.
De esta manera, el servidor sigue cargando `MicrobitASCIIAdapter.js` como siempre,
pero recibe los datos de acuerdo al nuevo protocolo.

### 3) Adaptar la pieza de arte generativo al patrón FSM

El código original del equipo (basado en `mouseX`, `mouseY`, `mouseIsPressed` y
`keyIsPressed`) se trasladó a:
- **`updateLogic()`**: convierte los valores crudos de acelerómetro en parámetros
  de dibujo (radio y resolución de círculo) y actualiza el estado según botones.
- **`drawRunning()`**: dibuja la forma poligonal usando ese estado, sin lógica extra.

### 4) Cumplir con la especificación de la pieza entregada

- **Eje Y del acelerómetro** controla la resolución del círculo.
- **Eje X del acelerómetro** controla el radio.
- **Botón A** actúa como el mouse presionado (activa el dibujo).
- **Botón B** actúa como `keyIsPressed` (controla el `fill`).



## Bitácora de reflexión
