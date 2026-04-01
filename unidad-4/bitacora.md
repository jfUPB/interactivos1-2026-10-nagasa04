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

### ❌ Limitación 4
**Problema**: Checksum es suma simple (detecta ~99% corrupción pero no 100%)
**Impacto**: Datos muy corruptos podrían pasar
**Solución futura**: CRC16 para detectar todas las corrupturas
