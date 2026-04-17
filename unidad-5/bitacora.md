# Unidad 5

## Bitácora de proceso de aprendizaje

### Actividad 1
### Objetivo
Analizar el cambio de protocolo de ASCII a binario en micro:bit, identificar el problema de sincronización y documentar por qué el framing lo resuelve.

### Paso 1 - Del ASCII al binario: qué cambia

<details>
<summary>Observación principal</summary>

- En ASCII, los datos viajan como texto legible y de longitud variable.
- En binario, los datos viajan como bytes compactos y de longitud fija.

</details>

<details>
<summary>Ventajas de binario frente a ASCII</summary>

- Menor tamaño por paquete.
- Menor latencia de transmisión.
- Parseo más rápido y predecible.
- Menor consumo de ancho de banda.

</details>

<details>
<summary>Desventajas de binario</summary>

- No es legible a simple vista.
- Más difícil de depurar en terminal.
- Requiere acordar formato exacto (tipos y endianness).

</details>

<details>
<summary>Conversión solicitada a hexadecimal</summary>

Para:
- xValue = 500
- yValue = 524
- aState = True
- bState = False

Formato: >2h2B
- xValue (500) -> 01 F4
- yValue (524) -> 02 0C
- aState (1) -> 01
- bState (0) -> 00

Paquete binario final (sin framing):
01 F4 02 0C 01 00

</details>

### Paso 2 - Sin sincronización: por qué falla

<details>
<summary>Por qué aparecen valores absurdos sin framing</summary>

- El puerto serial entrega un flujo continuo de bytes, no paquetes con fronteras.
- Si el receptor empieza a leer en un offset incorrecto, mezcla bytes de paquetes distintos.
- Eso produce lecturas corruptas (ej: X=3073, Y=513).

</details>

<details>
<summary>Respuesta: por qué en ASCII no pasaba tanto</summary>

ASCII tenía delimitador de fin de línea (\n).
- El receptor esperaba hasta encontrar \n.
- Eso reconstruía cada mensaje completo antes de parsear.

</details>

<details>
<summary>Respuesta: por qué en binario no sirve usar \n como delimitador</summary>

- En binario, cualquier byte 0-255 puede aparecer como dato.
- 0x0A (\n) puede aparecer dentro de X, Y, A o B.
- El receptor podría cortar paquete en una posición falsa.

</details>

### Paso 3 - Protocolo final con framing

<details>
<summary>Estructura final del paquete</summary>

- Byte 0: header = 0xAA
- Bytes 1-2: X (int16 BE)
- Bytes 3-4: Y (int16 BE)
- Byte 5: A (uint8)
- Byte 6: B (uint8)
- Byte 7: checksum = (suma bytes 1..6) % 256

</details>

<details>
<summary>Respuesta: tamaño con framing</summary>

- Sin framing: 6 bytes.
- Con framing: 8 bytes.
- Overhead: 2 bytes extra.

</details>

<details>
<summary>Respuesta: si un dato contiene 0xAA</summary>

Sí, podría aparecer 0xAA dentro de datos y parecer header falso.

Cómo se resuelve:
1. El receptor busca 0xAA como posible inicio.
2. Extrae 8 bytes.
3. Verifica checksum.
4. Si checksum falla, descarta ese inicio y sigue buscando.

Conclusión:
- El checksum evita aceptar falsos positivos de header.
- Header + checksum juntos hacen el protocolo robusto.

</details>

### Conclusiones de la actividad
<details>
<summary>Resumen final</summary>

- Pasar de ASCII a binario mejora eficiencia y latencia.
- El costo es menor legibilidad y mayor cuidado en parseo.
- Sin framing, el binario se desalinea fácilmente.
- Con framing (header + checksum), el receptor puede resincronizarse y validar integridad.

</details>


## Bitácora de aplicación 

### Actividad 02

### Objetivo
Crear un nuevo adaptador para protocolo serial binario sin modificar la arquitectura existente.

### Proceso de construcción

<details>
<summary>1) Punto de partida y restricciones</summary>

- Se tomó como base la arquitectura existente del caso de estudio.
- Restricción crítica respetada:
   - En `bridgeServer.js` solo se permite importar el nuevo adapter y agregar el caso `microbitBinary`.
   - No tocar `sketch.js`, `bridgeClient.js`, `fsm.js`, `BaseAdapter.js`.

</details>

<details>
<summary>2) Implementación de adapters/MicrobitBinaryAdapter.js</summary>

Se implementó el adapter con estos pasos:

- Hereda de `BaseAdapter`.
- Abre puerto serial con `SerialPort`.
- Acumula bytes en un buffer (`Buffer.concat`).
- Aplica framing buscando `0xAA`.
- Toma tramas de 8 bytes y valida checksum:
   - `(byte1 + byte2 + byte3 + byte4 + byte5 + byte6) % 256`
- Si checksum falla:
   - descarta trama,
   - registra `console.warn`,
   - continúa resincronizando.
- Si checksum es válido:
   - parsea `x` y `y` con `readInt16BE`,
   - convierte botones a boolean,
   - emite `this.onData?.({ x, y, btnA, btnB })`.

</details>

<details>
<summary>3) Registro del adapter en bridgeServer.js (cambio mínimo)</summary>

Solo se hicieron 2 cambios:

1. Importación:
    - `const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");`
2. Caso en `createAdapter()`:
    - `if (DEVICE === "microbitbinary") { ... return new MicrobitBinaryAdapter(...) }`

</details>

<details>
<summary>Restricciones cumplidas</summary>

- Se creó `adapters/MicrobitBinaryAdapter.js`.
- En `bridgeServer.js` solo se hizo:
   1. Importar `MicrobitBinaryAdapter`.
   2. Agregar caso `microbitBinary` en `createAdapter()`.
- No se modificó `sketch.js`, `bridgeClient.js`, `fsm.js`, `BaseAdapter.js`.

</details>

<details>
<summary>Corrección importante detectada en el enunciado</summary>

En el ejemplo del enunciado aparece:

- `AA 01 F4 02 0C 01 00 FE`

Pero el checksum correcto para esos bytes es:

- `0x01 + 0xF4 + 0x02 + 0x0C + 0x01 + 0x00 = 260`
- `260 % 256 = 4 = 0x04`

Es decir, para ese payload el checksum correcto sería `0x04`, no `0xFE`.

</details>

### Pruebas

<details>
<summary>Prueba intermedia A: error por dependencia faltante</summary>

Error detectado:

```text
Error: Cannot find module 'ws'
```

Solución aplicada:

- Verificar/recuperar `package.json` del caso base.
- Ejecutar `npm install` en `Unidad 5/Actividad 02`.

</details>

<details>
<summary>Prueba intermedia B: error por archivo faltante en adapters</summary>

Error detectado:

```text
Error: Cannot find module './adapters/SimAdapter'
```

Solución aplicada:

- Restaurar estructura base completa de la actividad.
- Verificar carpeta `adapters/` con los archivos necesarios.

</details>

<details>
<summary>Prueba intermedia C: puerto serial incorrecto</summary>

Error detectado:

```text
BRIDGE STATUS: error connect failed: Opening COM5: File not found
```

Diagnóstico:

- Se listaron puertos reales del sistema y del paquete `serialport`.
- Resultado: micro:bit estaba en `COM3` (no en `COM5`).

Solución aplicada:

- Ejecutar con `--serialPort COM3`.

</details>

<details>
<summary>Prueba intermedia D: puerto WebSocket ocupado</summary>

Error detectado:

```text
Error: listen EADDRINUSE: address already in use :::8081
```

Solución aplicada:

- Cerrar proceso que ocupaba 8081 o usar puerto alterno (8082/8083) para validar.

</details>

<details>
<summary>Prueba intermedia E: acceso serial denegado (puerto en uso)</summary>

Al disparar `connect` desde cliente WS:

```text
{"type":"status","state":"ready","detail":"bridge (microbitbinary)",...}
{"type":"status","state":"error","detail":"connect failed: Opening COM3: Access denied",...}
```

Log del bridge:

```text
[INFO] [NETWORK] Client requested adapter connect
[ERROR] [ADAPTER] connect failed: Opening COM3: Access denied
```

Posible causa:

- Otro programa estaba usando el puerto COM3 (monitor serial/IDE).

Solución:

- Cerrar el software que esté usando COM3 y reconectar.

</details>

<details>
<summary>Comandos usados durante validación</summary>

```bash
node bridgeServer.js --device microbitBinary --wsPort 8081 --serialPort COM3 --baud 115200 --verbose
node bridgeServer.js --device microbitBinary --wsPort 8083 --serialPort COM3 --baud 115200 --verbose
```

</details>

<details>
<summary>Evidencia real de funcionamiento del bridge en modo microbitBinary</summary>

Captura de consola obtenida:

```text
[2026-04-04T14:59:13.159Z] [INFO] WS listening on ws://127.0.0.1:8082 device=microbitbinary
[2026-04-04T14:59:13.162Z] [INFO] micro:bit found at COM3
```

Esto confirma:

- El modo `microbitBinary` se registra correctamente.
- El adapter se instancia sin errores de constructor/import.
- El puerto serial correcto se detecta (`COM3`).

</details>

<details>
<summary>Evidencia de datos válidos y checksum descartado</summary>

Durante esta sesión se logró evidencia de arranque y de errores de conexión, pero no se pudo capturar una traza estable con `Binary parsed` porque hubo conflicto de acceso al COM (`Access denied`).

Con COM libre, la evidencia esperada en consola es:

```text
Binary parsed: { x: 500, y: 524, btnA: true, btnB: false }
Binary frame discarded (checksum mismatch). expected=4 received=254
Binary parsed: { x: 120, y: -300, btnA: false, btnB: true }
```

Nota:

- El adapter ya implementa y registra `console.warn` al descartar checksum inválido.
- Solo falta ejecutar con el puerto COM3 sin bloqueo para capturar esas líneas en vivo.

</details>

### Resultado

Se implementó el adaptador binario respetando estrictamente la arquitectura pedida y se documentaron pruebas intermedias, errores reales y soluciones aplicadas.

Siguiente paso recomendado para cerrar evidencia final:

1. Cerrar cualquier app que use COM3.
2. Ejecutar bridge en `--verbose`.
3. Mover micro:bit y presionar botones para capturar líneas `Binary parsed` y, si aparece, algún descarte por checksum.



## Bitácora de reflexión

### Actividad 03

### 1) Tabla comparativa: ASCII vs binario

<details>
<summary>Comparación resumida</summary>

| Característica | MicrobitV2Adapter.js (ASCII) | MicrobitBinaryAdapter.js (Binario) |
|---|---|---|
| Tamaño por paquete | Variable, normalmente más largo | Fijo, 8 bytes |
| Delimitación / framing | `\n` al final de la línea | Header `0xAA` + tamaño fijo |
| Integridad | `CHK = |X| + |Y| + |A| + |B|` | `(bytes 1..6) % 256` |
| Complejidad del parser | Media | Alta |
| Depuración | Muy fácil en terminal serial | Más difícil, requiere decodificar bytes |

</details>

### 2) ¿Por qué la arquitectura desacoplada permite agregar otro protocolo?

<details>
<summary>Explicación</summary>

La arquitectura separa responsabilidades:

- **Adapter**: convierte el protocolo físico a un contrato común `{ x, y, btnA, btnB }`.
- **Bridge**: transporta esos datos sin conocer el protocolo real.
- **FSM + sketch.js**: consumen solo el contrato normalizado.

Por eso se puede cambiar de ASCII a binario sin tocar el frontend ni el transporte: solo cambia el adapter. El resto del sistema sigue recibiendo exactamente la misma estructura de datos.

</details>

### 3) ¿Cuándo preferir binario o ASCII?

<details>
<summary>Cuándo usar binario</summary>

- Sensores de alta frecuencia donde importa reducir tamaño y latencia.
- Sistemas embebidos donde el ancho de banda es limitado.
- Ejemplo: telemetría de un dron o de una máquina industrial que manda datos muchas veces por segundo.

</details>

<details>
<summary>Cuándo usar ASCII</summary>

- Prototipos y prácticas donde lo importante es ver el dato en claro.
- Depuración manual desde un terminal serial.
- Ejemplo: un prototipo educativo donde el estudiante quiere leer rápido `x,y,a,b` sin decodificar bytes.

</details>

### 4) Diagrama de flujo actualizado

<details>
<summary>Flujo de datos con protocolo binario</summary>

```mermaid
graph LR
  A[Sensor / firmware binario] -->|UART binario| B[MicrobitBinaryAdapter.js]
  B -->|{x, y, btnA, btnB}| C[bridgeServer.js]
  C -->|WebSocket| D[bridgeClient.js]
  D -->|Datos normalizados| E[sketch.js / FSM]
```

</details>

<details>
<summary>Componentes que cambiaron</summary>

- El adapter: de ASCII a `MicrobitBinaryAdapter.js`.
- El firmware del micro:bit: ahora envía frames binarios con header y checksum.

</details>

<details>
<summary>Componentes que permanecieron intactos</summary>

- `bridgeClient.js`
- `sketch.js`
- `fsm.js`
- `bridgeServer.js` como transporte general, sin cambios de lógica fuera del adapter

</details>






