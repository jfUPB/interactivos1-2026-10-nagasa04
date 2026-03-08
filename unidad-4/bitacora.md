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

<details>

<summary>microbitV2Adapter.js</summary>

``` js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

// Este adaptador implementa el protocolo nuevo del hardware y emite datos
// normalizados en el formato que el resto de la aplicación espera.
//
// El protocolo es (10 Hz, 115200 baudios):
// $T:tiempo|X:acel_x|Y:acel_y|A:estado_a|B:estado_b|CHK:checksum\n
// Checksum: abs(X)+abs(Y)+A+B
// Si el checksum no coincide, la trama se descarta con una advertencia.

class MicrobitV2Adapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();

    // Parámetros de conexión
    this.path = path; // puerto serie, p.ej. COM3 / /dev/ttyACM0
    this.baud = baud; // debe coincidir con el firmware del dispositivo

    // Estado interno
    this.port = null; // instancia SerialPort
    this.buf = ""; // buffer de datos hasta encontrar \n
    // modo verbo para debug (muestra tramas corruptas, etc.)
    this.verbose = verbose;
  }

  // Conecta al puerto serie y empieza a escuchar datos.
  async connect() {
    if (this.connected) return; // evitar doble conexión
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    // Creamos el objeto SerialPort pero no lo abrimos automáticamente.
    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    // Abrimos el puerto (promesa para poder usar await)
    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    // Estado interno y callback de conexión
    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    // Escuchar eventos: datos entrantes, errores y cierre de puerto.
    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  // Desconecta el adaptador cerrando el puerto serie.
  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }

    // Limpiar estado para permitir reconexiones posteriores.
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  // Callback que se ejecuta cada vez que llegan bytes por serial.
  // La trama está delimitada por un salto de línea '\n'.
  _onChunk(chunk) {
    // acumulamos bytes en el buffer
    this.buf += chunk.toString("utf8");

    // procesamos cada línea completa (hasta \n)
    let idx;
    while ((idx = this.buf.indexOf("\n")) >= 0) {
      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue; // línea vacía (p.ej. \n extra)

      const parsed = this._parseLine(line);
      if (parsed) {
        // El adaptador cumple el contrato: emite un objeto {x,y,btnA,btnB}
        this.onData?.(parsed);
      }
    }

    // Si por alguna razón el buffer se hace muy grande, lo limpiamos para
    // no consumir memoria indefinidamente.
    if (this.buf.length > 4096) this.buf = "";
  }

  // Parsea una línea completa de texto y valida el checksum.
  // Si la trama es inválida, devuelve null para que no se propague.
  _parseLine(line) {
    // Cada trama debe empezar con '$'
    if (!line.startsWith("$")) {
      if (this.verbose) console.warn("Invalid frame start:", line);
      return null;
    }

    // Removemos el '$' inicial y separamos por '|' para obtener pares clave:valor
    const parts = line.slice(1).split("|");
    const obj = {};

    for (const part of parts) {
      const [k, v] = part.split(":");
      if (!k || v === undefined) continue;
      obj[k] = v;
    }

    // Extraemos los valores esperados del diccionario
    const x = Number(obj.X);
    const y = Number(obj.Y);
    const a = Number(obj.A);
    const b = Number(obj.B);
    const chk = Number(obj.CHK);

    // Validaciones básicas: todo debe ser numérico.
    if (!Number.isFinite(x) || !Number.isFinite(y) || !Number.isFinite(a) || !Number.isFinite(b) || !Number.isFinite(chk)) {
      if (this.verbose) console.warn("Invalid numeric values in line:", line);
      return null;
    }

    // Calculamos el checksum según la especificación del hardware.
    const computed = Math.abs(x) + Math.abs(y) + (a ? 1 : 0) + (b ? 1 : 0);
    if (computed !== chk) {
      // Trama corrupta: no actualizamos estado, pero avisamos en consola
      console.warn("Corrupt frame (checksum mismatch):", line, "expected", computed, "got", chk);
      return null;
    }

    // Emitimos el objeto limpio esperado por el resto de la aplicación.
    return {
      x: x | 0,
      y: y | 0,
      btnA: Boolean(a),
      btnB: Boolean(b),
    };
  }

  // Si hay un error grave (p.ej. puerto desconectado), avisamos y nos
  // desconectamos para evitar estados inconsistentes.
  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    // Este adaptador es solo de lectura; no se espera enviar comandos al micro:bit.
  }
}

module.exports = MicrobitV2Adapter;
```

</details>

## Bitácora de reflexión

