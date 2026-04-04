# Unidad 6

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


## Bitácora de reflexión
