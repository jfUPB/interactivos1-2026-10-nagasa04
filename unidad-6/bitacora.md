# Unidad 6

## Bitácora de proceso de aprendizaje

### Actividad 01

### Objetivo
Analizar el caso Strudel + p5.js para entender qué problema arquitectónico resuelve la ejecución temporal con timestamp y cómo se conecta con lo trabajado en Unidades 4 y 5.

### 1) ¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?

<details>
<summary>Respuesta</summary>

Recibir un mensaje significa que el evento llegó por la red al navegador.
Ejecutarlo significa que el sistema decide cuándo producir el efecto visual.

En un sistema en tiempo real, no siempre se ejecuta al llegar porque la red introduce variaciones (jitter). Si se ejecuta apenas llega, las animaciones quedan desfasadas respecto al audio.

Por eso, en este caso de estudio, los eventos se encolan y se ejecutan cuando el reloj local alcanza el timestamp del mensaje.

</details>

### 2) ¿Por qué un sistema audiovisual puede necesitar timestamp además de los datos del evento?

<details>
<summary>Respuesta</summary>

Los datos del evento dicen qué hacer (por ejemplo, tipo de sonido, intensidad o duración), pero el timestamp dice cuándo hacerlo.

Sin timestamp:
- El orden y el ritmo dependen del retardo de red.
- Aparecen desajustes entre golpe musical y respuesta visual.

Con timestamp:
- El sistema sincroniza la ejecución con un tiempo objetivo.
- Se reduce el desfase audiovisual.
- El comportamiento es más estable aunque cambie la latencia de red.

En resumen: datos = contenido del evento; timestamp = coordinación temporal del evento.

</details>

### 3) ¿Qué aspectos de la arquitectura de U4 y U5 permanecen intactos aunque la fuente ya no sea hardware?

<details>
<summary>Respuesta</summary>

Se mantienen intactos los principios de desacople:

- Capa de adaptación:
  - Antes: adapter de hardware serial (ASCII/binario).
  - Ahora: capa que transforma mensajes musicales de Strudel a formato consumible por visuales.

- Capa de transporte (bridge):
  - Sigue separada del rendering.
  - Solo enruta mensajes entre origen y aplicación visual.

- Capa de lógica visual (FSM / sketch):
  - No depende del protocolo físico.
  - Reacciona a eventos normalizados.

La idea central no cambia: se puede cambiar la fuente de datos (microcontrolador o app musical) sin romper el resto del sistema, siempre que se conserve el contrato de eventos.

</details>

### Conclusión
La novedad de esta unidad no es solo parsear eventos, sino planificar su ejecución temporal. Eso amplía el enfoque de U4/U5: además de adaptar datos, ahora se controla el momento de ejecución para mantener sincronía audiovisual.






















## Bitácora de aplicación 

### Actividad 02

### Objetivo
Integrar una aplicación musical hecha con Strudel a un sistema visual en p5.js sin romper la arquitectura desacoplada. La idea central es separar recepción de mensajes, normalización, cola temporal y renderizado.

### 1) ¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?

<details>
<summary>Respuesta</summary>

Recibir un mensaje significa que el evento ya llegó al sistema por WebSocket.
Ejecutarlo significa que el sistema decide el momento exacto en que ese evento produce un efecto visual.

En este caso, no se debe dibujar apenas llega el mensaje. Primero se guarda en una cola y después se activa cuando el reloj local alcanza el timestamp indicado por el evento.

Eso evita que la red o la latencia alteren la sincronización entre música y visuales.

</details>

### 2) ¿Por qué un sistema audiovisual puede necesitar timestamp además de los datos del evento?

<details>
<summary>Respuesta</summary>

Los datos del evento dicen qué ocurrió: por ejemplo, qué sonido se disparó, su delta o duración, y otros parámetros musicales.
El timestamp dice cuándo debe ejecutarse visualmente ese evento.

En un sistema audiovisual el timestamp es necesario porque:

- permite sincronizar la respuesta visual con la música;
- compensa retrasos de red;
- mantiene el orden temporal correcto de los eventos;
- hace posible programar animaciones aunque los mensajes lleguen desordenados o con latencia variable.

En resumen: los datos describen el evento; el timestamp coordina su ejecución.

</details>

### 3) ¿Qué aspectos de la arquitectura de las Unidades 4 y 5 permanecen intactos?

<details>
<summary>Respuesta</summary>

Aunque la fuente de datos ya no sea hardware, la arquitectura sigue separando responsabilidades:

- **Adapter**: antes traducía serial ASCII o binario; ahora traduce mensajes de Strudel a un formato estable.
- **Bridge**: sigue siendo un intermediario que solo reenvía mensajes normalizados.
- **Frontend / FSM**: sigue consumiendo datos ya procesados y no debe interpretar mensajes crudos.

Lo que permanece intacto es la idea de desacoplar origen, transporte, normalización y renderizado. Cambia la fuente de datos, pero no el contrato interno del sistema.

</details>

### 4) Estructura de integración usada en el repositorio

<details>
<summary>Flujo general</summary>

En el repositorio de referencia, el flujo queda así:

1. Strudel genera eventos musicales en el navegador.
2. Strudel envía mensajes por WebSocket al puerto 8080.
3. Un Adapter recibe y normaliza esos mensajes.
4. `bridge.js` reenvía el mensaje normalizado.
5. `sketch.js` guarda el evento en una cola.
6. `draw()` ejecuta la animación cuando el reloj local alcanza el timestamp.

</details>

<details>
<summary>Contrato normalizado sugerido</summary>

Un formato claro y fácil de explicar es este:

```json
{
  "type": "strudel",
  "timestamp": 1710000000000,
  "payload": {
    "eventType": "noteEvent",
    "s": "tr909bd",
    "delta": 0.25
  }
}
```

Ese contrato es útil porque deja explícito qué sonido llegó, cuándo debe ejecutarse y cuánto debe durar su respuesta visual.

</details>

### 5) Configuración de Strudel y prueba funcional

<details>
<summary>Qué se debe configurar</summary>

- Strudel corre en el navegador.
- Los eventos se envían por WebSocket al puerto 8080.
- El puente reenvía a la app visual.
- La visualización se activa por timestamp y no por llegada inmediata.

Sonidos mínimos que debe soportar el sistema:

- `bd`
- `sd` o `cp`
- `hh`

Parámetros mínimos usados:

- tipo de sonido;
- timestamp;
- delta o duración equivalente.

</details>

<details>
<summary>Prueba de sincronización</summary>

La prueba correcta consiste en verificar que:

- los mensajes llegan primero a la cola;
- la animación no arranca al recibir el WebSocket;
- el evento se activa justo cuando `now >= timestamp`;
- la duración visual sigue `delta` o el tiempo equivalente.

Esto demuestra que la sincronización temporal está separada de la recepción del mensaje.

</details>

### 6) Problemas encontrados y solución

<details>
<summary>Problema principal</summary>

El riesgo más grande era mezclar demasiada lógica en un solo lugar.

Si el bridge interpreta el mensaje, decide colores y también dibuja, se rompe la arquitectura.
Si el frontend procesa mensajes crudos directamente, deja de ser mantenible.

</details>

<details>
<summary>Solución aplicada</summary>

- Recepción: el mensaje entra por WebSocket.
- Normalización: el Adapter convierte el mensaje a un contrato estable.
- Cola temporal: el frontend guarda el evento hasta que su timestamp sea válido.
- Renderizado: `drawRunning` o su equivalente solo dibuja usando el estado ya calculado.

Así se mantiene la separación de responsabilidades y se evita acoplar la red con la animación.

</details>

### Conclusión
Esta actividad añade una capa nueva sobre lo visto en las Unidades 4 y 5: además de adaptar formatos, ahora hay que programar el momento de ejecución. La arquitectura desacoplada sigue funcionando porque cada capa hace una sola tarea: recibir, normalizar, programar y renderizar.











































## Bitácora de reflexión

# Actividad 03

## Pregunta 1: Diagrama detallado del flujo de datos

### Descripción visual del flujo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SISTEMA AUDIOVISUAL DESACOPLADO                          │
└─────────────────────────────────────────────────────────────────────────────┘

   NAVEGADOR (Tab 1)                      NAVEGADOR (Tab 2)
   ──────────────────                      ─────────────────

   Strudel App                             p5.js Visual App
   ───────────                             ────────────────
   - Genera eventos          WebSocket     - Recibe eventos normalizados
   - Emite por WS            (8080)        - Los almacena en cola
   - Normaliza? NO.                        
                                ↓
   ┌──────────────────┐      ┌──────────────────────────────────────────┐
   │                  │      │         bridgeServer.js (Node.js)         │
   │  const pat =     │      │  ┌─────────────────────────────────────┐  │
   │  s("[bd...]")    │      │  │ WebSocket Server (8080) - Strudel    │  │
   │  $: stack(...)   │      │  │ ├─ Recibe: {args, address, ts}      │  │
   │                  │      │  │ └─ Delega a StrudelAdapter          │  │
   │ envía al 8080 ───│─────→│                                        │  │
   │                  │      │  ┌─────────────────────────────────────┐  │
   │                  │      │  │ StrudelAdapter.normalize()            │  │
   │                  │      │  │ ├─ Extrae: s, delta, timestamp      │  │
   │                  │      │  │ ├─ Genera contrato normalizado      │  │
   │                  │      │  │ └─ Retorna: {type, timestamp, ...}  │  │
   │                  │      │                                        │  │
   │                  │      │  ┌─────────────────────────────────────┐  │
   │                  │      │  │ broadcastToP5() - Reenvía           │  │
   │                  │      │  │ └─ Envía mensaje normalizado al 8081│  │
   │                  │      │                                        │  │
   │                  │      └─────────────────────────────────────────┘  │
   └──────────────────┘                          ║
                                                 ║ WebSocket
                                                 ║ (8081)
                                                 ↓
                                    ┌──────────────────────────┐
                                    │   sketch.js (p5.js)      │
                                    │                          │
                                    │  socket.onmessage() {    │
                                    │  ├─ eventQueue.push()    │
                                    │  └─ sort(timestamp)      │
                                    │  }                       │
                                    └──────────────────────────┘
                                                 ║
                                                 ↓
                                    ┌──────────────────────────┐
                                    │   draw() - FSM Loop      │
                                    │                          │
                                    │  now = Date.now()        │
                                    │  while(queue && now>=ts) │
                                    │  ├─ shift evento         │
                                    │  └─ activeAnimations++   │
                                    └──────────────────────────┘
                                                 ║
                                                 ↓
                                    ┌──────────────────────────┐
                                    │  drawAnimatedElement()   │
                                    │                          │
                                    │  ├─ drawKick()          │
                                    │  ├─ drawSnare()         │
                                    │  └─ drawHat()           │
                                    │                          │
                                    │  → Canvas pintado        │
                                    └──────────────────────────┘
```

### Componentes del flujo

| Componente | Responsabilidad | Límite |
|---|---|---|
| **Strudel** | Genera eventos musicales; envía por WS | No interpreta la visual |
| **WebSocket (8080)** | Transporte entrada Strudel → Bridge | No procesa lógica |
| **StrudelAdapter** | Normaliza mensaje crudo a contrato | No toca rendering, no decide visual |
| **bridgeServer.js** | Recibe, delega a adapter, reenvía | No tiene lógica musical o visual |
| **WebSocket (8081)** | Transporte salida Bridge → p5.js | Transporte puro |
| **socket.onmessage()** | Recibe evento, lo guarda en cola | No renderiza, no valida lógica visual |
| **eventQueue[]** | Almacena eventos ordenados por timestamp | Solo datos, no ejecución |
| **draw() / FSM** | Comprueba tiempo, activa animaciones | Decide qué dibujar basado en tiempo |
| **drawAnimatedElement()** | Renderiza formas según tipo de sonido | Puro rendering, dato ya decidido |

---

## Pregunta 2: Tabla comparativa - Unidades 4, 5 y 6

| Aspecto | **Unidad 4 (ASCII)** | **Unidad 5 (Binario)** | **Unidad 6 (Strudel)** |
|---|---|---|---|
| **Fuente de datos** | Micro:bit (Hardware USB Serial) | Micro:bit (Hardware USB Serial) | Strudel (Software, WebSocket) |
| **Formato del mensaje** | ASCII: `$T:...\|X:...\|Y:...\|CHK:...\n` | Binario: `0xAA` + 8 bytes estructurados | JSON: `{args, address, timestamp}` |
| **Protocolo de transporte** | Serial Port 9600/115200 baud | Serial Port 115200 baud | WebSocket (JSON over WS) |
| **Problema técnico principal** | Checksum de validación; parseo manual | Endianness, alineación de bytes | Normalización de múltiples formatos de args |
| **Mecanismo de validación** | Checksum final (suma simple) | Byte de control en posición 7 | Cheque de tipo de objeto JSON |
| **Lugar donde ocurre la traducción** | p5.serialport.js shim (Web Serial API) | p5.serialport.js shim (Web Serial API) | StrudelAdapter.normalize() en bridge.js |
| **Papel del tiempo/sincronización** | Timestamp en el mensaje (T:) | Timestamp en el mensaje (bytes 0-2) | Timestamp presente en data de Strudel; validación en draw() |
| **Qué se valida** | Formato ASCII, rango de valores | Estructura binaria, checksum | Presencia de args y address, tipo de datos |

**Patrón observado:** Los tres evolcionan desde recepción directa (U4/U5) a cola temporal (U6), mejorando sincronización y tolerancia a latencia.

---

## Pregunta 3: ¿Por qué esta unidad sigue perteneciendo a la misma arquitectura?

### Análisis: Patrones que persisten

Aunque **la fuente de datos cambió de hardware a software**, la arquitectura sigue siendo idéntica en **estructura y responsabilidades**:

#### 1) **Adapter Pattern**
- **U4:** Serial ASCII → Objeto normalizado
- **U5:** Serial Binario → Objeto normalizado
- **U6:** JSON de Strudel → Objeto normalizado

El Adapter sigue adaptando un formato externo a un contrato interno.

#### 2) **Bridge Pattern**
- **U4/U5:** bridgeServer entre serial y p5.js
- **U6:** bridgeServer entre WebSocket(8080) y WebSocket(8081)

El Bridge sigue siendo un intermediario **puro** que no interpreta datos.

#### 3) **Separación de capas**
```
Externo → Adapter → Bridge → Frontend → Render
```

En todas las unidades:
- **Recepción:** El dato entra sin procesarse
- **Normalización:** Se ajusta a un contrato interno estable
- **Transporte:** Se reenvía sin lógica
- **Consumo:** El frontend obtiene datos ya limpios
- **Rendering:** Se dibuja basado en lo que ya fue decidido

#### 4) **Cola temporal (NEW en U6)**
Es una **extensión coherente**, no una ruptura:
- U4/U5: Los estímulos llegan casi en tiempo real (serial direct)
- U6: Los estímulos pueden llegar desordenados (red variable), por eso la cola

La cola es una **captura de la misma separación:** Recepción ≠ Ejecución.

#### Conclusión
La Unidad 6 no es una arquitectura nueva. Es la **misma arquitectura aplicada a una fuente de datos diferente**, mejorando el mecanismo de sincronización. El sistema sigue siendo:

**Desacoplado: origen → normalización → transporte → programación temporal → renderizado**

---

## Pregunta 4: Decisiones de mapeo visual y justificación

### Diseño de las animaciones

#### Mapeo sonido → Visual

| Sonido | Visual | Justificación |
|---|---|---|
| **bd** (kick drum) | Círculo expandible desde centro | El kick es grave, profundo, expande desde adentro |
| **sd, cp** (snare, clap) | Barra horizontal contrayéndose | El snare es percusivo y "corta"; la contracción horizontal sugiere un impacto transversal |
| **hh** (hi-hat) | Cuadrado pequeño rotando | El hi-hat es agudo, metálico, rápido; la rotación sugiere movimiento constante |

#### Parámetros visuales decididos

1. **Color:**
   - Rojo (BD): Lo bajo, profundo, energético
   - Cian (Snare): Medio, claro, definido
   - Amarillo (Hat): Alto, brillante, luminoso

2. **Duración:**
   - Usa `delta` (parámetro musical) → se convierte a ms
   - Mínimo 80ms para garantizar visibilidad
   - `duration = Math.max(80, delta * 1000)`

3. **Posición:**
   - Aleatoria dentro de márgenes (20%-80% del canvas)
   - Justificación: No distrae; simula espacialidad

4. **Progresión (lerp):**
   - Lineal según `progress = elapsed / duration`
   - Proporcional a delta: sonidos más largos animan más tiempo

#### ¿Por qué este mapeo tienen sentido?

- **Sincronía auditiva-visual:** Cada sonido tiene una correspondencia clara con una forma
- **Inteligibilidad:** Es obvio cuál sonido genera cuál efecto
- **Variedad:** No todos los sonidos producen el mismo cambio
- **Respeto al dato:** Usa parámetros reales (delta, sound type), no valores fijos
- **Escalabilidad:** Agregar un nuevo sonido es agregar un `case` en `drawAnimatedElement()`

---

## Pregunta 5: Integración futura de una tercera aplicación

### Escenario: "Quiero conectar una aplicación de MIDI en lugar de (o además de) Strudel"

#### Qué se conservaría ✅

1. **La estructura de bridgeServer.js**
   - Sigue siendo un WebSocket server con dos puertos
   - Puerto de entrada: `MIDI_PORT = 8082` (nuevo)
   - Puerto de salida: `/p5JS_PORT = 8081` (igual)

2. **El patrón Adapter**
   - Crear `adapters/MidiAdapter.js`
   - Mismo método: `normalize(rawMidiMessage)` → contrato estable
   - Retorna: `{type: 'midi', timestamp, payload: {note, velocity, ...}}`

3. **El método broadcastToP5()**
   - El bridge **no cambia**; solo reenvía lo que cada adapter normaliza

4. **La cola temporal en sketch.js**
   - Funciona igual para eventos MIDI o Strudel
   - Solo verifica `msg.type` para diferenciar origen
   - Mismo proceso: queue → draw() → render

5. **Las funciones de drawing**
   - Se reutilizan o se extienden con nuevos `case` en `drawAnimatedElement()`

#### Qué cambiaría ⚠️

1. **En bridgeServer.js:** Agregar nuevo `WebSocket.Server` para MIDI
   ```js
   const wssStrudel = new WebSocket.Server({ port: 8080 });
   const wssMidi = new WebSocket.Server({ port: 8082 });  // NUEVO
   const wssP5 = new WebSocket.Server({ port: 8081 });
   ```

2. **En bridgeServer.js:** Registrar el nuevo adapter
   ```js
   const MidiAdapter = require('./adapters/MidiAdapter');
   const midiAdapter = new MidiAdapter();
   
   wssMidi.on('connection', (ws) => {
     ws.on('message', (raw) => {
       const normalized = midiAdapter.normalize(raw);  // NUEVO
       broadcastToP5(normalized);
     });
   });
   ```

3. **En sketch.js:** Extender la lógica de mapeo
   ```js
   if (msg.type === 'midi') {  // NUEVO
     eventQueue.push({
       timestamp: msg.timestamp,
       sound: msg.payload.note,
       delta: msg.payload.duration ?? 0.25,
       source: 'midi',  // Para diferenciar visualización si es necesario
     });
   }
   ```

4. **En drawAnimatedElement():** Agregar visualización MIDI
   ```js
   case 'midi':
     drawMidiNote(anim, p);  // Nueva función
     break;
   ```

#### Arquitectura para 3+ aplicaciones

```
Strudel(8080) ─→ ┐
Midi(8082)    ───→ StrudelAdapter / MidiAdapter → bridgeServer → p5.js(8081)
OSC(8083)     ─→ ┘                                    + OscAdapter
                        broadcastToP5()
```

**Cada nueva fuente:**
- Obtiene su **propio puerto de entrada**
- Obtiene su **propio Adapter**
- Comparte `broadcastToP5()` y el rendering final

---

## Conclusión

La Actividad 03 demuestra que:

1. **El flujo es robusto:** La cola temporal y el timestamp permiten múltiples fuentes sin conflicto
2. **La arquitectura es extensible:** Agregar una nueva aplicación no requiere tocar la visual ni el bridge
3. **La separación es real:** Cada componente cumple una sola responsabilidad clara
4. **El patrón escala:** Lo que funciona para 1 fuente funciona para N fuentes

Esto es lo que significa una **arquitectura desacoplada:** no importa qué genera los datos, el sistema de recepción → normalización → programación temporal → renderizado permanece idéntico.
