# Unidad 4

## Bitácora de proceso de aprendizaje

### Actividad 01

### Resumen
En esta actividad se analizó el flujo de datos del micro:bit (firmware fijo) para preparar la integración con una pieza de arte generativo en p5.js.

<details>
<summary>Protocolo identificado</summary>

- UART a `115200` baudios.
- Frecuencia aproximada: `10 Hz`.
- Trama CSV por línea:
  - `xValue,yValue,aState,bState\n`
- Ejemplo:
  - `-123,456,True,False`

</details>

### Hallazgos importantes

<details>
<summary>Datos que entrega el hardware</summary>

- `xValue` y `yValue`: acelerómetro en ejes X/Y.
- `aState` y `bState`: estado de botones A/B (`True` o `False`).
- Cada línea representa una muestra completa del estado del dispositivo.

</details>

<details>
<summary>Riesgos técnicos detectados</summary>

- Ruido o líneas incompletas en serial pueden romper parseo si no se valida.
- Si el parseo no normaliza tipos, la UI puede comportarse de forma incorrecta.

</details>

### Estrategia de integración definida

<details>
<summary>Plan para conectar con p5.js</summary>

1. Leer línea serial completa hasta `\n`.
2. Separar por comas.
3. Convertir `x/y` a número y `aState/bState` a booleano.
4. Mapear variables:
   - `xValue` -> control horizontal/radio.
   - `yValue` -> control vertical/resolución.
   - Botones -> acciones de dibujo (activar, relleno, color, etc.).

</details>

### Herramientas usadas

<details>
<summary>Entorno de trabajo</summary>

- SerialTerminal para observar tramas en vivo.
- p5.js para la pieza generativa.

</details>

### Siguiente paso
Implementar el adapter y el sketch de integración en la siguiente actividad, manteniendo la arquitectura del caso de estudio.








### Bitácora de aplicación 

### Actividad 02

### Resumen
Se integró el firmware nuevo del micro:bit respetando la arquitectura existente. La corrección clave fue en el adaptador V2 para parsear bien la trama y validar checksum.

<details>
<summary>Protocolo usado (firmware fijo)</summary>

- Formato: `$T:tiempo|X:acel_x|Y:acel_y|A:estado_a|B:estado_b|CHK:checksum\n`
- Baudios: `115200`
- Checksum: $CHK = |X| + |Y| + |A| + |B|$

</details>

### Problemas más importantes y solución

<details>
<summary>1) Conectaba pero no dibujaba</summary>

- Causa: el parser no coincidía con el formato real del firmware.
- Solución: actualizar `MicrobitV2Adapter.js` para parseo exacto de `T/X/Y/A/B/CHK` y validaciones de rango/checksum.

</details>

<details>
<summary>2) Error EADDRINUSE en puerto 8081</summary>

- Causa: había otro proceso Node usando `8081`.
- Solución: cerrar el PID activo y reiniciar bridge.

</details>

### Corrección técnica clave

<details>
<summary>Qué valida ahora el adapter V2</summary>

- Inicio de trama con `$`.
- Seis campos por `|`: `T`, `X`, `Y`, `A`, `B`, `CHK`.
- `X` y `Y` en rango `[-2048, 2047]`.
- `A` y `B` como `0/1`.
- Checksum correcto; si falla, se descarta trama y se hace `console.warn`.

</details>

<details>
<summary>Contrato que se mantuvo hacia el sistema</summary>

- Salida estándar: `{ x, y, btnA, btnB }`.
- No se rompió bridge ni frontend.

</details>

### Prueba rápida

<details>
<summary>Pasos de ejecución</summary>

1. Conectar micro:bit por USB.
2. Ejecutar:
   - `node .\bridgeServer.js --device microbit-v2 --serialPort COM3 --baud 115200 --wsPort 8081 --verbose`
3. Abrir la app y pulsar Connect.
4. Probar:
   - A: dibuja.
   - Inclinación X/Y: cambia forma.
   - B: activa relleno.

</details>

## Bitácora de reflexión

### Actividad 03

### Bitácora - Unidad 4 - Actividad 03

### Objetivo
Documentar el diagrama detallado del sistema físico interactivo construido en la actividad anterior, incluyendo componentes, flujo de información e interacciones clave.

### Diagrama general del sistema

```text
[Mundo físico]                      [Backend - Node.js]                   [Frontend - Navegador]
                 
 micro:bit              UART      MicrobitV2Adapter             WS    bridgeClient.js          
 - acelerómetro X/Y    115200     bridgeServer.js               --->  PainterTask (FSM)        
 - botones A/B         -------->  - parseo/validación                 drawRunning() + canvas   
 - firmware fijo                  - broadcast de JSON                 p5.js                    
                 
```

<details>
<summary>Qué representa este diagrama</summary>

- El hardware nunca habla directo con p5.js.
- El adapter convierte trama serial cruda a objeto estándar.
- El bridge transporta datos por WebSocket.
- La FSM del frontend decide cómo actualizar estado y dibujar.

</details>

### Componente 1: Hardware (micro:bit)

<details>
<summary>Entrada, salida y formato de trama</summary>

- Interfaz de salida: UART a 115200 baudios.
- Frecuencia: ~10 Hz (una trama cada 100 ms).
- Formato de trama:

```text
$T:tiempo|X:acel_x|Y:acel_y|A:estado_a|B:estado_b|CHK:checksum\n
```

- Significado de campos:
  - T: timestamp desde arranque.
  - X/Y: acelerómetro en rango -2048..2047.
  - A/B: botones en 0/1.
  - CHK: suma esperada = |X| + |Y| + A + B.

</details>

### Componente 2: MicrobitV2Adapter.js

<details>
<summary>Responsabilidad y flujo interno</summary>

Responsabilidad principal:
- Leer bytes seriales.
- Armar líneas completas por `\n`.
- Parsear y validar cada trama.
- Emitir objeto limpio `{ x, y, btnA, btnB }`.

Flujo interno:
1. `_onChunk(chunk)` acumula en buffer.
2. Cuando encuentra `\n`, extrae una línea.
3. `parseV2Line(line)` valida formato y campos.
4. Calcula checksum local y compara con CHK recibido.
5. Si es válida: `this.onData?.({ x, y, btnA, btnB })`.
6. Si es corrupta: se descarta y se hace `console.warn`.

</details>

<details>
<summary>Regla de validación crítica</summary>

```text
CHK_valido <=> CHK_recibido === (|X| + |Y| + A + B)
```

Si no coincide:
- No se actualiza la vista.
- Se mantiene el último estado válido.
- Se registra advertencia para diagnóstico.

</details>

### Componente 3: bridgeServer.js (transporte)

<details>
<summary>Papel del servidor WebSocket</summary>

- Recibe eventos `onData` del adapter.
- Construye mensajes WS tipo `microbit`.
- Hace broadcast a clientes conectados.
- Gestiona estado (`ready`, `connected`, `disconnected`, `error`).

Ejemplo de payload transmitido:

```json
{
  "type": "microbit",
  "x": -245,
  "y": 12,
  "btnA": true,
  "btnB": false,
  "t": 1710699345000
}
```

</details>

### Componente 4: bridgeClient.js (frontend)

<details>
<summary>Cómo entrega datos a la FSM</summary>

- Abre WebSocket hacia `ws://127.0.0.1:8081`.
- Recibe JSON del bridge.
- Si `type === "microbit"`, dispara callback de datos.
- Envía el payload hacia `PainterTask` con `EVENTS.DATA`.

Resultado:
- La lógica visual no depende del puerto serial.
- Solo depende del contrato JSON estándar.

</details>

### Componente 5: PainterTask (FSM) y renderizado

<details>
<summary>Estados y transiciones</summary>

Estados principales:
- `estado_esperando`: sin dibujo, esperando conexión.
- `estado_corriendo`: procesa datos y renderiza.

Transiciones:
- `CONNECT` -> pasa a `estado_corriendo`.
- `DISCONNECT` -> vuelve a `estado_esperando`.

Eventos relevantes:
- `DATA`: actualiza estado interno con valores de hardware.

</details>

<details>
<summary>Mapeos e interacción visual</summary>

Reglas de control usadas en el arte generativo:
- X del acelerómetro -> radio del polígono.
- Y del acelerómetro -> resolución (cantidad de lados).
- Botón A -> habilita dibujo.
- Botón B -> habilita relleno.

Regla de transición:
- Al detectar liberación de B (`prevB && !btnB`) se cambia color.

</details>

### Flujo de información extremo a extremo

```text
1) Usuario inclina micro:bit / pulsa botones
2) Firmware emite trama serial
3) Adapter parsea y valida CHK
4) Si trama válida: emite {x,y,btnA,btnB}
5) bridgeServer hace broadcast WS
6) bridgeClient recibe y postea EVENTS.DATA
7) PainterTask.updateLogic actualiza estado
8) drawRunning renderiza en canvas p5.js
```

<details>
<summary>Escenario de trama corrupta</summary>

- Llega trama con CHK inválido.
- Adapter detecta inconsistencia.
- Trama descartada (sin `onData`).
- Canvas no cambia abruptamente.
- Se reporta warning en consola.

</details>

### Interacciones entre componentes

| Origen | Destino | Interacción | Resultado |
|---|---|---|---|
| micro:bit | MicrobitV2Adapter | Trama UART | Datos listos para parseo |
| MicrobitV2Adapter | bridgeServer.js | `onData({x,y,btnA,btnB})` | Payload estándar backend |
| bridgeServer.js | bridgeClient.js | Mensaje WebSocket | Entrega en tiempo real |
| bridgeClient.js | PainterTask | `EVENTS.DATA` | Actualización de estado |
| PainterTask | Canvas p5.js | `drawRunning()` | Visual generativa interactiva |

### Puntos críticos del sistema

<details>
<summary>Riesgos y control aplicado</summary>

- Integridad de datos seriales:
  - Control: checksum obligatorio.
- Ruido/tramas corruptas:
  - Control: descarte silencioso + warning.
- Acoplamiento entre capas:
  - Control: contrato JSON estable.
- Consistencia de interacción:
  - Control: FSM y estado previo de botones.

</details>

### Conclusión
El sistema quedó documentado con detalle suficiente para reconstrucción y sustentación. Se identifican claramente componentes, flujo de datos e interacciones, manteniendo la arquitectura desacoplada de la unidad.
