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

## Bitácora de reflexión

# Actividad 03

### Objetivo 🎯

Construir en la bitácora un **diagrama detallado y completo** del sistema físico interactivo realizado en la Actividad 02.

Este diagrama debe permitir que **cualquier persona** (profesor, evaluador, compañero) entienda:
- ¿Cuáles son los **componentes** del sistema?
- ¿Cómo fluye la **información** entre ellos?
- ¿Qué **interacciones** suceden?
- ¿Dónde están los **puntos críticos** (validación, transiciones)?

---

### Panorama General del Sistema

El sistema es un **arte generativo interactivo** controlado por micro:bit. Un dispositivo físico envía datos (acelerómetro + botones) que se convierten en arte visual en tiempo real.

```
[MUNDO FÍSICO]              [COMPUTADORA]                 [NAVEGADOR]
┌──────────────┐         ┌─────────────────────┐         ┌──────────┐
│  Micro:bit   │────────→│  Node.js            │────────→│  p5.js   │
│ (aceleróm.)  │ Serial  │ (WebSocket Server)  │  WS     │  (Canvas)│
│  (botones)   │ 115200  │ (MicrobitV2Adapter) │         │          │
└──────────────┘         └─────────────────────┘         └──────────┘
     |                           |                            |
     │ Datos continuos           │ JSON normalizado           │ Renderizado
     │ 10 Hz                     │ por segundo                │ ~60fps
     └─────────────────────────────────────────────────────────┘
```

---

## Componentes del Sistema (Detallado)

### 1️⃣ Hardware: Micro:bit con Firmware Fijo

**Características**:
- **Procesador**: ARM Cortex-M0
- **Sensores**: Acelerómetro (MEMS)
- **Entrada de usuario**: Botones A y B
- **Salida**: Puerto UART (puerto serie)

**Qué envía**:
```
┌────────────────────────────────────────────────────────┐
│  Trama serial cada 100ms (10 Hz)                       │
├────────────────────────────────────────────────────────┤
│ Formato: $T:tiempo|X:acel_x|Y:acel_y|A:btn|B:btn|CHK  │
│                                                  \n    │
│ Ejemplo: $T:45020|X:-245|Y:12|A:1|B:0|CHK:258 \n      │
│                                                        │
│ Significado:                                          │
│ - T:45020     → Timestamp 45020ms desde arranque      │
│ - X:-245      → Eje X acelerómetro = -245             │
│ - Y:12        → Eje Y acelerómetro = 12              │
│ - A:1         → Botón A presionado (1=sí, 0=no)      │
│ - B:0         → Botón B soltado                      │
│ - CHK:258     → Checksum (validación)                │
└────────────────────────────────────────────────────────┘
```

**Rango de valores**:
| Campo | Rango | Descripción |
|-------|-------|-------------|
| X, Y | -2048 a 2047 | Acelerómetro (16 bits con signo) |
| A, B | 0 o 1 | Digital (botón no presionado = 0, presionado = 1) |
| CHK | 0-999 | Checksum: \|X\| + \|Y\| + A + B |

---

### 2️⃣ Adaptador Serial: MicrobitV2Adapter.js

**Responsabilidad**: Leer puerto serie, validar datos, emitir JSON limpio

**Ubicación**: Backend (Node.js) → `adapters/MicrobitV2Adapter.js`

**Flujo interno**:

```
Bytes del puerto serie
        ↓
┌──────────────────────┐
│  Buffer acumulativo  │  _onChunk(chunk)
│  this.buf += chunk   │
└──────────────────────┘
        ↓  (busca \n)
┌──────────────────────┐
│  Línea completa      │  Ej: "$T:45020|X:-245|Y:12|A:1|B:0|CHK:258"
│  this.buf.indexOf(\n)│
└──────────────────────┘
        ↓
┌──────────────────────┐
│  Parsing             │  _parseLine(line)
│  Split por "|"       │
│  Split por ":"       │
│  Convertir a números │
└──────────────────────┘
        ↓
┌──────────────────────┐
│  Validación Checksum │  computed = |X| + |Y| + A + B
│  computed === chk?   │  Si NO coincide → return null
└──────────────────────┘
        ↓ (SI válida)
┌──────────────────────┐
│  Emitir JSON         │  {x:-245, y:12, btnA:true, btnB:false}
│  this.onData?.(obj)  │
└──────────────────────┘
        ↓
bridgeServer.js (recibe)
```

**Ejemplo: Trama corrupta**:
```
Trama recibida: "$T:45020|X:-245|Y:12|A:1|B:0|CHK:100\n"
                                                      ↑ Erróneo

Cálculo:  computed = |-245| + |12| + 1 + 0 = 258
          chk (recibido) = 100
          
Resultado: 258 ≠ 100 → return null
           NO emite onData()
           console.warn("Corrupt frame...")
           Canvas MANTIENE estado anterior (seguro)
```

---

### 3️⃣ Servidor WebSocket: bridgeServer.js

**Responsabilidad**: Conectar hardware con navegador, retransmitir datos

**Ubicación**: Backend (Node.js) → `bridgeServer.js`

**Arquitectura**:

```
┌─────────────────────────────────────────────┐
│         bridgeServer.js (Node.js)           │
├─────────────────────────────────────────────┤
│                                              │
│  WebSocketServer (puerto 8081)               │
│       ↑ (clientes del navegador)            │
│       │                                      │
│  ┌────┴────────────────────────────────┐    │
│  │ adapter.onData(obj) → broadcast()   │    │
│  │ Recibe {x,y,btnA,btnB}              │    │
│  │ Envía JSON a TODOS los clientes     │    │
│  └────┬────────────────────────────────┘    │
│       │                                      │
│  ┌────┴────────────────────────────────┐    │
│  │ Estado de conexión + eventos        │    │
│  │ status: "ready|connected|error"     │    │
│  └────────────────────────────────────┘    │
│                                              │
└─────────────────────────────────────────────┘
```

**JSON enviado por WS**:
```javascript
// Tipo 1: Estado
{
  "type": "status",
  "state": "connected",
  "detail": "sim running at 30Hz",
  "t": 1710699345000
}

// Tipo 2: Datos hardware
{
  "type": "microbit",
  "x": -245,
  "y": 12,
  "btnA": true,
  "btnB": false,
  "t": 1710699345000
}
```

---

### 4️⃣ Cliente WebSocket: bridgeClient.js

**Responsabilidad**: Conectar navegador con servidor, traducir JSON a eventos

**Ubicación**: Frontend (navegador) → `bridgeClient.js`

**Máquina de estados interna**:

```
┌──────────────────┐
│   WebSocket      │
│   (cerrado)      │
└────────┬─────────┘
         │
    User: Click "Connect"
         ↓
    ┌──────────────────┐
    │ ws.open()        │
    │ new WebSocket()  │
    └────────┬─────────┘
             │
        WS onopen
             ↓ (conexión stablecida)
        ┌─────────────────┐
        │ send({cmd:...}) │
        │ Listo para datos│
        └────────┬────────┘
                 │
        WS onmessage
                 ↓
        ┌──────────────────────────┐
        │ Parse JSON               │
        │ tipo=="microbit" ?        │
        └────────┬─────────────────┘
                 │
         YES (datos hardware)
                 ↓
        ┌──────────────────────────┐
        │ this._onData?.(msg)      │
        │ painter.postEvent(DATA)  │
        └──────────────────────────┘
```

---

### 5️⃣ Máquina de Estados: PainterTask (sketch.js)

**Responsabilidad**: Gestionar lifecycle de la aplicación y renderizado

**Ubicación**: Frontend (navegador) → `sketch.js` (clase PainterTask)

**Diagrama de estados**:

```
┌─────────────────────────────────────────────────────────────┐
│                  Máquina de Estados (FSM)                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌───────────────────────────┐                               │
│  │  estado_esperando         │                               │
│  │                           │                               │
│  │  • cursor() visible       │                               │
│  │  • esperando conexión     │                               │
│  │  • parado (no dibuja)     │                               │
│  └──────────┬────────────────┘                               │
│             │                                                │
│             │ EVENTS.CONNECT                                │
│             │ (usuario hace clic "Connect")                 │
│             ↓                                                │
│  ┌───────────────────────────┐                               │
│  │  estado_corriendo         │                               │
│  │  (ENTRY)                  │                               │
│  │  • noCursor()             │                               │
│  │  • background(255)        │                               │
│  │  • prevA=false,prevB=false│                               │
│  │                           │                               │
│  │  (PROCESANDO)             │                               │
│  │  • Cada frame: draw()     │                               │
│  │  • Si DATA event:         │                               │
│  │    → updateLogic()        │                               │
│  │    → drawRunning()        │                               │
│  │  • Si prevB && !btnB:     │                               │
│  │    → cambiar color        │                               │
│  │                           │                               │
│  │  (EXIT)                   │                               │
│  │  • cursor()               │                               │
│  └──────────┬────────────────┘                               │
│             │                                                │
│             │ EVENTS.DISCONNECT                             │
│             │ (error o usuario desconecta)                  │
│             ↓                                                │
│             └────→ estado_esperando                         │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**updateLogic(data)** - Mapeo de valores:

```javascript
// ENTRADA: {x:-245, y:12, btnA:true, btnB:false}

// PASO 1: Mapeo eje X
const radius = map(x, -2048, 2047, -width/2, width/2)
              = map(-245, -2048, 2047, -360, 360)
              ≈ -43.6 pixels

// PASO 2: Mapeo eje Y
const circleResolution = int(map(y, -2048, 2047, 2, 10))
                        = int(map(12, -2048, 2047, 2, 10))
                        ≈ 6 vértices (hexágono)

// PASO 3: Guardar en estado
this.rxData = {
  radius: -43.6,
  circleResolution: 6,
  btnA: true,      ← Activar dibujo
  btnB: false,     ← Sin relleno
  ready: true
}

// PASO 4: Detectar transición de botón B
if (prevB && !btnB) {
  this.c = color(random(255), random(255), random(255))
  // Cambiar color si se suelta B
}
```

**drawRunning()** - Renderizado puro:

```javascript
// Dibuja SOLO si btnA === true (como mouseIsPressed original)

if (mb.btnA) {
  push();
  translate(width/2, height/2);   // Centro del canvas
  rotate(radians(painter.angle)); // Rotación acumulativa
  
  // Relleno controlado por botón B
  if (mb.btnB) {
    fill(34, 45, 122, 50);        // Azul semitransparente
  } else {
    noFill();                       // Solo outline
  }
  
  // Dibujar polígono regular
  stroke(painter.c);              // Color (cambia al soltar B)
  beginShape();
  for (let i = 0; i <= circleResolution; i++) {
    const θ = TAU / circleResolution * i;
    const x = cos(θ) * radius;
    const y = sin(θ) * radius;
    vertex(x, y);
  }
  endShape();
  
  painter.angle += 1;  // Rotación continua
  pop();
}
```

---

### 6️⃣ Renderizado: p5.js Canvas

**Responsabilidad**: Mostrar arte generativo visual

**Canvas**:
- Tamaño: adaptado a ventana (`windowWidth` x `windowHeight`)
- Contenido: polígonos rotatorios
- Controlado por valores de acelerómetro

**Ejemplo visual** (texto):
```
Cuando acelerómetro X = 0, Y = 100:
  radius ≈ 0 (punto pequeño)
  circleResolution = 8 (octágono)
  → Se ve como punto/estrella pequeña

Cuando acelerómetro X = 1500, Y = -500:
  radius ≈ 250 (grande)
  circleResolution = 3 (triángulo)
  → Se ve como triángulo grande girando

Cuando botón A presionado Y botón B presionado:
  Relleno azul semitransparente + outline
  → Efecto opaco

Cuando botón A presionado Y botón B soltado:
  Solo outline, sin relleno
  → Efecto transparente
```

---

### Flujo Completo de Datos

### Ejemplo: Usuario inclina micro:bit y presiona botón B

```
TIEMPO 0ms: Usuario rota micro:bit hacia la derecha (X=500)
│
├─→ Acelerómetro detecta: X=500
│
├─→ Micro:bit firmware genera trama:
│   $T:50|X:500|Y:-300|A:1|B:1|CHK:301\n
│
└─→ ENVÍA por UART (115200 baud)

TIEMPO ~1ms: Datos llegan al puerto serial del PC
│
├─→ MicrobitV2Adapter._onChunk() recibe bytes
│
├─→ Acumula en buffer hasta \n
│
├─→ _parseLine() extrae campos:
│   X=500, Y=-300, A=1, B=1, CHK=301
│
├─→ Valida checksum:
│   computed = |500| + |-300| + 1 + 1 = 802
│   ¡ESPERA! 802 ≠ 301 → Trama CORRUPTA
│
├─→ return null
│   NO emite onData()
│   canvas MANTIENE dibujo anterior
│
└─→ console.warn("Corrupt frame...")

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
(Si trama fuera válida, siguería así:)

TIEMPO ~1ms: MicrobitV2Adapter.onData({x:500, y:-300, btnA:true, btnB:true})
│
├─→ bridgeServer.js recibe evento
│
├─→ broadcast() envía JSON por WebSocket:
│   {type:"microbit", x:500, y:-300, btnA:true, btnB:true, t:...}
│
└─→ A TODOS los navegadores conectados

TIEMPO ~2ms: bridgeClient.js en navegador recibe JSON
│
├─→ onmessage() dispara
│
├─→ Verifica type==="microbit" ✓
│
├─→ Emite callback this._onData?.(msg)
│
├─→ painter.postEvent({
│     type: EVENTS.DATA,
│     payload: {x:500, y:-300, btnA:true, btnB:true}
│   })
│
└─→ Encola evento en FSM

TIEMPO ~3ms: painter.update() procesa evento DATA
│
├─→ PainterTask en estado_corriendo
│
├─→ Call updateLogic({...})
│
│  ├─ radius = map(500, -2048, 2047, -360, 360) ≈ 89
│  │
│  ├─ circleResolution = int(map(-300, -2048, 2047, 2, 10)) ≈ 3
│  │
│  ├─ this.rxData.btnA = true
│  │ this.rxData.btnB = true
│  │
│  ├─ Comparar: prevB && !btnB
│  │ prevB=true (anterior), btnB=true (actual)
│  │ true && false = false → NO cambiar color
│  │
│  └─ this.prevB = true (guardar estado)
│
└─→ updateLogic() completa

TIEMPO ~4ms: draw() llama drawRunning()
│
├─→ mb.btnA === true → SÍ dibuja
│
├─→ push()
│ translate(360, 360)     // Centro
│ rotate(radians(142))    // Algún ángulo acumulado
│
├─→ mb.btnB === true → Aplicar fill
│ fill(34, 45, 122, 50)
│
├─→ beginShape()
│ for i=0 to 3:
│   θ = 360°/3 * i = 0°, 120°, 240°
│   vertex(cos(θ)*89, sin(θ)*89)
│ endShape()
│
├─→ painter.angle += 1
│
├─→ pop()
│
└─→ Canvas updated (triángulo azul girando)

TIEMPO ~16ms: Siguiente frame de p5.js (~60fps)
│
└─→ Si no hay nuevo evento DATA, dibuja con valores anteriores
```

---

### Interacciones Clave Entre Componentes

### 1. Hardware ↔ Adaptador
| Acción | Disparador | Resultado |
|--------|-----------|-----------|
| Inclinar micro:bit | Cambio acelerómetro | Nueva trama serial |
| Presionar botón A | Contacto eléctrico | A=1 en siguiente trama |
| Soltar botón B | Apertura contacto | B=0 en siguiente trama |

### 2. Adaptador ↔ Servidor
| Acción | Disparador | Resultado |
|--------|-----------|-----------|
| Trama válida | Checksum correcto | adapter.onData() emite JSON |
| Trama corrupta | Checksum ≠ | Silenciosamente descartada |
| Error puerto | Puerto desconectado | adapter.onError() notifica |

### 3. Servidor ↔ Navegador
| Acción | Disparador | Resultado |
|--------|-----------|-----------|
| Usuario clic "Connect" | bridgeClient.open() | WS conecta, envía cmd startup |
| Datos hardware válidos | adapter.onData() | broadcast() a todos clientes |
| Cliente cierra ventana | WS onclose | bridgeServer desconecta adapter |

### 4. Navegador ↔ Canvas
| Acción | Disparador | Resultado |
|--------|-----------|-----------|
| btnA=true | Datos recibidos | drawRunning() dibuja |
| btnB presionado→soltado | Transición detectada | Cambiar color aleatorio |
| X acelerómetro cambia | Data event | updateLogic() actualiza radius |
| Y acelerómetro cambia | Data event | updateLogic() actualiza resolución |

### Decisiones de Diseño Documentadas

### ✅ Por qué Adaptador Pattern

**Problema**: Hardware tiene protocolo diferente
**Solución**: Crear MicrobitV2Adapter que parsea nuevo formato
**Beneficio**: bridgeServer.js sin cambios → arquitectura respetada

### ✅ Por qué Validación Checksum

**Problema**: Puertos serie pueden corromper datos
**Solución**: Calcular checksum y comparar
**Beneficio**: Solo datos válidos afectan canvas → usuario seguro

### ✅ Por qué FSM en lugar de directo a canvas

**Problema**: Lógica de transiciones es compleja
**Solución**: Máquina de estados (ENTRY/EXIT/eventos)
**Beneficio**: Código predecible, fácil testeable, escalable

### ✅ Por qué separar updateLogic() y drawRunning()

**Problema**: Mezclar mapeos con renderizado es confuso
**Solución**: updateLogic() mapea, drawRunning() dibuja
**Beneficio**: Debugging fácil, lógica centralizada, renderizado puro

### ✅ Por qué prevA/prevB en scope de clase

**Problema**: Detectar transiciones (cambios) en tiempo real
**Solución**: Guardar estado anterior para comparación
**Beneficio**: Puede detectar "botón soltado" (edge), no solo "presionado"

---

### Validación del Sistema (Testing)

### Prueba 1: Trama válida
```
Input:  $T:100|X:500|Y:-200|A:1|B:0|CHK:701\n
         (|500| + |-200| + 1 + 0 = 701 ✓)

Output: {x:500, y:-200, btnA:true, btnB:false}
→ Canvas actualiza: triángulo/hexágono girando
```

### Prueba 2: Trama corrupta
```
Input:  $T:100|X:500|Y:-200|A:1|B:0|CHK:100\n
         (Checksum erróneo)

Output: null (no emite)
→ console.warn("Corrupt frame...")
→ Canvas mantiene estado anterior
```

### Prueba 3: Botón A liberado
```
Input: {x:500, y:-200, btnA:false, btnB:true}

Output: drawRunning() hace: if (mb.btnA) → false
→ Canvas STOP dibuja
→ (puede mostrar dibujo anterior si no se borra)
```

### Prueba 4: Botón B transición
```
Frame N:   prevB=true,  btnB=true
Frame N+1: prevB=true,  btnB=false  ← LIBERACIÓN
           if (prevB && !btnB) → TRUE
           → Cambiar color aleatorio
           → Siguiente polígono será otro color
```


### Limitaciones Conocidas

### ❌ Limitación 1
**Problema**: Si usuario presiona botón A continuamente, no hay "liberación"
**Impacto**: Color no cambia hasta soltar A y presionar B nuevamente
**Solución futura**: Detectar "doble clic" de A para cambiar color independientemente

### ❌ Limitación 2
**Problema**: Si micro:bit sendsRapidly (> 10Hz), algunos frames se pierden
**Impacto**: Dibuja menos frecuente de lo que podría
**Solución futura**: Buffer circular en adaptador para guardar frames

### ❌ Limitación 3
**Problema**: Si conexión WS cae, no reconecta automáticamente
**Impacto**: Usuario debe hacer clic "Connect" de nuevo
**Solución futura**: Reconexión automática con backoff exponencial

### ❌ Limitación 4
**Problema**: Checksum es suma simple (detecta ~99% corrupción pero no 100%)
**Impacto**: Datos muy corruptos podrían pasar
**Solución futura**: CRC16 para detectar todas las corrupturas
