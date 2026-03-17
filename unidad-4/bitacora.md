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

### Objetivo

Construir un sistema físico interactivo que conecte el hardware del caso de estudio
(con firmware fijo con un nuevo protocolo serial) con la pieza de arte generativo
p5.js suministrada por el equipo de diseño.

✅ Debes usar **exactamente** la misma arquitectura de software existente:
- **Capa backend**: adaptadores (`adapters/*`) que leen el puerto serie y emiten
  datos limpios.
- **Capa de transporte**: `bridgeServer.js` y `bridgeClient.js` (no se modifica).
- **Capa frontend**: `sketch.js` con la máquina de estados (`PainterTask`) que
  actualiza estado y dibuja.

---

### Hardware / Protocolo del dispositivo (firmware fijo)

El sensor envía tramas ASCII a 115200 baudios, 10 Hz, con este formato:

```
$T:tiempo|X:acel_x|Y:acel_y|A:estado_a|B:estado_b|CHK:checksum\n
```

### Diccionario de datos

| Campo | Tipo | Rango | Descripción |
|-------|------|-------|-------------|
| T | int | 0-∞ | Timestamp en ms desde arranque del dispositivo |
| X | int | -2048 a 2047 | Eje X del acelerómetro |
| Y | int | -2048 a 2047 | Eje Y del acelerómetro |
| A | int | 0 o 1 | Botón A (0=liberado, 1=presionado) |
| B | int | 0 o 1 | Botón B (0=liberado, 1=presionado) |
| CHK | int | 0-999 | Checksum (validación de integridad) |

### Ejemplo válido

```
$T:45020|X:-245|Y:12|A:1|B:0|CHK:258\n
```

Verificación checksum:
```
CHK = |X| + |Y| + A + B
    = |-245| + |12| + 1 + 0
    = 245 + 12 + 1 + 0
    = 258 ✓
```

### Requisito de integridad

Si el `CHK` calculado ≠ `CHK` recibido:
- La trama se **descarta silenciosamente** (no se emite onData())
- Se emite `console.warn("Corrupt frame...")` en el servidor (para debugging)
- El canvas **NO se actualiza**

Esto garantiza que solo datos válidos afecten la visualización.

---

### Arquitectura implementada

```
┌─────────────────────────────────────────────────────────────┐
│                  FLUJO DE DATOS                              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Hardware (micro:bit)                                         │
│         ↓ (UART 115200 baud)                                 │
│  MicrobitV2Adapter                                            │
│  ├─ _onChunk(): acumula bytes en buffer                      │
│  ├─ _parseLine(): parsea trama $...|CHK:...|               │
│  └─ valida checksum → { x, y, btnA, btnB }                 │
│         ↓ (this.onData?.(parsed))                            │
│  bridgeServer.js (Node.js)                                   │
│  └─ retransmite por WebSocket                                │
│         ↓ (JSON por WS)                                      │
│  Browser (p5.js)                                             │
│  ├─ bridgeClient.js recibe JSON                              │
│  └─ dispara EVENTS.DATA                                      │
│         ↓                                                     │
│  PainterTask (FSM)                                           │
│  ├─ updateLogic(): mapea valores a parámetros               │
│  └─ diagrama de estados:                                     │
│      estado_esperando ↔ estado_corriendo                     │
│         ↓                                                     │
│  drawRunning()                                                │
│  └─ dibuja polígono en canvas                                │
│         ↓                                                     │
│  Canvas (viewport)                                            │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

### Qué se hizo (resumen detallado)

#### 1) Crear adaptador nuevo: `MicrobitV2Adapter.js`

**Responsabilidad**: Procesar nuevo protocolo serial y emitir datos normalizados

#### Método `_onChunk(chunk)`

Callback que se dispara cuando llegan bytes por puerto serial:

1. **Acumular bytes**: `this.buf += chunk.toString("utf8")`
   - Los datos raramente llegan de una sola vez
   - Acumulamos hasta encontrar `\n` para tener línea completa

2. **Extraer líneas**: Buscar `\n` y procesar líneas completas
   ```javascript
   // Si buf = "$T:45020|...\n$T:45030|..." 
   // Extraemos: "$T:45020|..."
   // Resto sigue en buf para siguiente iteración
   ```

3. **Parsear cada línea**: Llamar a `_parseLine()` para validar
   - Si devuelve valor (válida) → emitir `this.onData?(parsed)`
   - Si devuelve null (corrupta) → hacer nada

4. **Mantenimiento**: Si buffer crece > 4096 bytes, limpiar
   - Protección contra protocolos rotos o puertos colgados

#### Método `_parseLine(line)`

Parsea y valida una línea completa:

1. **Validar inicio**: Debe empezar con `$`
2. **Split estructurado**: Dividir por `|` en pares `clave:valor`
   ```
   Input: "$T:45020|X:-245|Y:12|A:1|B:0|CHK:258"
   Output dict: { T:"45020", X:"-245", Y:"12", A:"1", B:"0", CHK:"258" }
   ```

3. **Convertir a números**: `Number(obj.X)` etc
   - Validar que sean finitos (`Number.isFinite()`)

4. **VALIDACIÓN CRÍTICA – Checksum**:
   ```javascript
   computed = Math.abs(x) + Math.abs(y) + (a ? 1 : 0) + (b ? 1 : 0)
   if (computed !== chk) {
       console.warn("Corrupt frame...");
       return null;  // ← NO emitir
   }
   ```

5. **Emitir objeto normalizado**:
   ```javascript
   return {
       x: x | 0,          // Asegurar entero
       y: y | 0,
       btnA: Boolean(a),  // 1→true, 0→false
       btnB: Boolean(b)
   };
   ```

**Por qué esto funciona**: El adaptador "promete" el mismo formato que el anterior
→ `bridgeServer.js` sigue funcionando sin cambios → arquitectura respetada

#### 2) Redirigir adaptador: `MicrobitASCIIAdapter.js`

```javascript
module.exports = require("./MicrobitV2Adapter");
```

- El servidor sigue cargando `MicrobitASCIIAdapter.js` (por eso se llama así)
- Pero recibe el nuevo protocolo de `MicrobitV2Adapter`
- Transparente para el resto del sistema

#### 3) NO modificar transporte

`bridgeServer.js` y `bridgeClient.js` permanecen **intactos**

- `bridgeServer` recibe `{x, y, btnA, btnB}` y lo retransmite
- `bridgeClient` en navegador recibe el JSON y dispara `EVENTS.DATA`

#### 4) Adaptar pieza generativa: `sketch.js`

Patrón FSM con dos estados:

#### `estado_esperando`
- Usuario ve cursor
- Espera clic en "Connect"
- Transición: CONNECT event → `estado_corriendo`

#### `estado_corriendo`
- Procesa eventos DATA
- Llama `updateLogic()` para mapear valores
- Llama `drawRunning()` para dibujar

#### `updateLogic(data)` - Mapeos

```javascript
// EJE X: -2048 a 2047 → radio del polígono
const radius = map(data.x, -2048, 2047, -width/2, width/2);

// EJE Y: -2048 a 2047 → resolución (número de vértices 2-10)
const circleResolution = int(map(data.y, -2048, 2047, 2, 10));
```

**Por qué estos mapeos**:
- X controlaba `mouseX - width/2` en original (movimiento izq-dcha)
- Y controlaba `mouseY + 100` en original (movimiento arriba-abajo)
- Adaptamos rangos del hardware a canvas

**Detección de transiciones (botón B)**:
```javascript
// comparar estado ANTERIOR (prevB) con ACTUAL (btnB)
if (this.prevB && !this.rxData.btnB) {
    // Cambiar color cuando se SUELTA botón B
    this.c = color(random(255), random(255), random(255), ...);
}
```

#### `drawRunning()` - Renderizado puro

- Lee `this.rxData` (estado)
- Dibuja polígono regular con p5.js
- **ÚNICO input**: botones (A activa, B aplicaFill) y parámetros mapeados
- **SIN lógica**: solo `beginShape() → vertex() → endShape()`

---

### Demostración (C1: App funciona sin fallos)

#### Ejecución paso a paso

1. **Terminal - Iniciar servidor**:
   ```bash
   cd "c:\...\Actividad 02"
   node bridgeServer.js --device sim --wsPort 8081
   ```
   
   Esperado:
   ```
   [INFO] WS listening on ws://127.0.0.1:8081 device=sim
   ```

2. **Navegador - Abrir y conectar**:
   - Abrir `index.html`
   - Abriendo DevTools (F12) ver consola
   - Pulsar "Connect"
   
   Esperado en consola:
   ```
   WS open
   BRIDGE STATUS: connected, sim running at 30Hz
   Microbit ready to draw
   ```

3. **Interacción - Botones**:
   - Pulsar "Botón A" (físico o en UI)
   - Canvas muestra polígonos
   - Pulsar "Botón B" mientras A está presionado
   - Polígonos cambian de color

4. **Validación**:
   - Ningún error en consola
   - Canvas actualiza continuamente
   - Botones responden correctamente

---

### Análisis de decisiones (C2: Explica qué ves, cómo hiciste, por qué)

#### Pregunta: ¿Por qué validar checksum?

**Respuesta**:
- El hardware envía datos a 115200 baud a través de UART
- Las líneas pueden corromperse por ruido electromagnético
- Si no validamos, dibujamos datos basura
- El checksum (suma simple) cuesta casi nada y detecta la mayoría de corrupción
- **Decisión**: Descartar tramas corruptas silenciosamente pero registrar en logs

#### Pregunta: ¿Por qué  separar updateLogic() de drawRunning()?

**Respuesta**:
- **updateLogic()**: concentra toda la lógica (mapeos, transiciones)
- **drawRunning()**: solo dibuja, sin "pensar"
- Ventajas:
  - Fácil debuggear: si falla mapeo, está en updateLogic
  - P5.js + FSM = patrón probado
  - Código testeable: puedo testear mapeos sin GUI
  - Original también separaba: prototipo vs draw()

#### Pregunta: ¿Por qué usar prevA/prevB como variable de clase?

**Respuesta**:
```javascript
// MAL: if (!rxData.btnA) → detecta en 60fps, sin memoria de anterior
// BIEN: if (this.prevB && !this.rxData.btnB) → detecta transición
```
- `prevA/prevB` guarda estado del frame anterior
- Permite detectar **cambios** (edges), no solo valores
- Si estuviera en `rxData`, se perdería cada frame

#### Pregunta: ¿Qué pasa si llega trama corrupta?

**Respuesta**:
```
Escenario: llega "$T:45020|X:-245|Y:12|A:1|B:0|CHK:100\n"
(CHK debería ser 258)

1. _parseLine() calcula: computed = 245 + 12 + 1 + 0 = 258
2. Compara: 258 ≠ 100 → No coincide
3. console.warn("Corrupt frame...")
4. return null
5. _onChunk() hace if (parsed) → false → NO emite onData()
6. updateLogic() NO se ejecuta
7. Canvas mantiene su estado anterior

Resultado: usuario no ve cambio abrupto (seguro)
Sistema sigue esperando siguiente trama válida
```

#### Pregunta: ¿Cómo se mapean los valores?

**Respuesta**:
```javascript
// Acelerómetro X: -2048 a 2047
// Canvas: -width/2 a width/2 (pixel-based)
map(data.x, -2048, 2047, -width/2, width/2)

// Acelerómetro Y: -2048 a 2047
// Polígono: 2 a 10 vértices
map(data.y, -2048, 2047, 2, 10)
```

- `map()` es función de p5.js que hace: `(value - min1) / (max1 - min1) * (max2 - min2) + min2`
- Lineal
- Si X=0 → radius=0 (polígono degenerado, punto)
- Si X=-2048 → radius=-360 (polígono negado en piel, efecto espejo)

---

### Errores encontrados y corregidos

| Error | Causa | Síntoma | Corrección |
|-------|-------|---------|-----------|
| `btnA/btnB` no detectaban transiciones | Comparaban en cada frame sin memoria | Cambio de color no se triggers | Mover `prevA/prevB` a scope de clase, no `rxData` |
| Dibujo solo en centro fijo | Primer intento cambió traducción | No usaba acelerómetro | Volver a mapeo original: X→radio, Y→resolución |
| Estado anterior no se reiniciaba | Primer enter a `estado_corriendo` | Detectaba falsa transición de B | Agregar `prevA=false; prevB=false;` en ENTRY |

---

### Limitaciones y mejoras posibles

#### Limitación 1: Checksum simple
- Solo suma, no detecta errores de orden o duplicados
- **Mejora**: CRC16 o SHA1 (pero procesador limite)

#### Limitación 2: Sin buffer de datos
- Si hardware envía rápido, se pierde frame si bridge no lee a tiempo
- **Mejora**: Circular buffer en adaptador

#### Limitación 3: Sin persistencia
- Si hago refresh, pierdo dibujo
- **Mejora**: canvas.save() → localStorage

#### Limitación 4: Sin logs en archivo
- `console.warn()` solo en browser, no permanente
- **Mejora**: endpoint API para guardar logs en servidor

---

### Para la sustentación

**Prepara estas respuestas**:

1. **"¿Cómo el hardware se conecta con p5.js?"**
   → Adapter → WebSocket → FSM → canvas... (mostrar diagrama)

2. **"¿Qué sucede si la trama viene corrupta?"**
   → Se descarta, se registra warn, canvas no cambia

3. **"¿Por qué X controla radio e Y controla resolución?"**
   → Porque el prototipo original usaba mouseX y mouseY para eso

4. **"¿Dónde está la lógica de transición de color?"**
   → En updateLogic(), detección `prevB && !btnB`

5. **"¿Qué pasa si presiono botón A?"**
   → `btnA=true` → `drawRunning()` dibuja polígonos

## Bitácora de reflexión

### Actividad 03

### Componentes principales

1. **Hardware**
   - **Micro:bit** con firmware fijo (envía tramas seriales a 115200 baudios).
   - El micro:bit actúa como sensor (acelerómetro y botones) y transmisor de datos.

2. **Capa de backend (servidor)**
   - `bridgeServer.js` (Node.js): servidor WebSocket que hace de puente entre serial y navegador.
   - `adapters/MicrobitV2Adapter.js` (Adapter): lee la serie, valida el checksum y emite objetos JSON.

3. **Capa de transporte (WebSocket)**
   - `bridgeClient.js` (navegador): cliente WebSocket que recibe los objetos JSON y genera eventos.

4. **Capa frontend (p5.js)**
   - `sketch.js`: máquina de estados (PainterTask) que procesa los datos y dibuja en pantalla.

### Flujo de datos

1. El micro:bit envía tramas seriales al PC:
   - Formato: `$T:tiempo|X:...|Y:...|A:...|B:...|CHK:...\n`
   - Frecuencia: 10 Hz.

2. `MicrobitV2Adapter` (Node.js):
   - Recibe bytes del puerto serie.
   - Construye líneas completas hasta el `\n`.
   - Parsea los campos (X, Y, A, B, CHK).
   - Valida checksum. Si falla, **descarta** la trama y escribe un warning.
   - Emite `{ x, y, btnA, btnB }` cuando la trama es válida.

3. `bridgeServer.js` (Node.js):
   - Escucha conexiones WebSocket.
   - Cuando un cliente se conecta, envía mensajes de estado.
   - Reenvía cada objeto `{x,y,btnA,btnB}` como mensaje JSON tipo `microbit`.

4. `bridgeClient.js` (navegador):
   - Se conecta al servidor WS.
   - Recibe mensajes JSON y dispara `EVENTS.DATA` con los valores.

5. `PainterTask` (p5.js):
   - En `updateLogic(data)`: mapea valores crudos a parámetros de dibujo.
   - En `drawRunning()`: usa esos parámetros para dibujar la forma generativa.

### Diagrama del sistema

```
[Micro:bit (HW)]
    |  (serial 115200, líneas $...\n)
    v
[MicrobitV2Adapter.js]  <-- valida checksum, genera {x,y,btnA,btnB}
    |  (JSON internal)
    v
[bridgeServer.js]  <-- reenvía por WebSocket
    |  (JSON via WS)
    v
[bridgeClient.js]  <-- dispara EVENTS.DATA
    |  (evento interno)
    v
[PainterTask en sketch.js]  <-- updateLogic() + drawRunning()
    |  (render)
    v
[p5.js canvas] (arte generativo)
```

### Interacciones clave

- **Hardware → Backend**: El micro:bit envía datos continuos (10 Hz) al puerto serie.
- **Backend → Transporte**: El adaptador emite datos normalizados, el servidor los envía por WebSocket.
- **Transporte → Frontend**: El cliente recibe datos y los transforma en eventos `DATA`.
- **Frontend → Renderizado**: El renderer dibuja sólo si recibe datos válidos y el botón A está "presionado".

