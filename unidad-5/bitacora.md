# Unidad 5
## Bitácora de proceso de aprendizaje
### Actividad 01

#### Objetivo 🎯

Analizar y comprender la **transición de un protocolo de comunicación ASCII a un protocolo binario** en la transmisión de datos desde micro:bit. Esta actividad implica:

- Observar las diferencias técnicas
- Participar en la discusión
- Registrar observaciones críticas
- Medir impacto en eficiencia

---

### Paso 1: Del ASCII al Binario — ¿Qué cambia?

#### Contexto: Protocolo ASCII Anterior (Unidad 4)

En la Unidad 4 trabajamos con un protocolo ASCII que enviaba datos como **texto legible**:

```python
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    
    # Empaquetado en ASCII (formato texto)
    data = "{},{},{},{}\n".format(xValue, yValue, aState, bState)
    uart.write(data)
    sleep(100)  # 10 Hz (100ms entre envíos)
```

**Ejemplo de datos enviados**:
```
Valores: xValue=500, yValue=524, aState=True, bState=False

Texto ASCII resultante: "500,524,True,False\n"
                         ↑   ↑    ↑    ↑     ↑
                         1   1    1    1     1 byte
Tamaño total: 19 bytes (muy variable según valores)
```

---

#### Nuevo: Protocolo Binario (Unidad 5)

Ahora reemplazamos **únicamente la línea de empaquetado**:

```python
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    
    # Empaquetado en BINARIO (formato compacto)
    data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
    uart.write(data)
    sleep(100)  # 10 Hz (sin cambios)
```

**Ejemplo de datos enviados**:
```
Mismos valores: xValue=500, yValue=524, aState=True, bState=False

Bytes binarios resultantes: 01 F4 02 0C 01 00
                            [     xValue    ] [     yValue    ] [A] [B]
Tamaño total: SIEMPRE 6 bytes (tamaño FIJO)
```

---

### Paso 2: Entender struct.pack()

#### ¿Qué es struct.pack()?

`struct.pack()` es una función de Python que **empaqueta valores Python en bytes binarios** según un formato especificado.

**Sintaxis**:
```python
struct.pack(formato, *valores)
```

#### Desglose del Formato: `'>2h2B'`

| Parte | Significado |
|-------|-------------|
| `>` | **Endianness**: big-endian (bytes más significativos primero) |
| `2h` | **2 signed shorts**: dos enteros de 2 bytes CON SIGNO (-32768 a 32767) |
| `2B` | **2 unsigned bytes**: dos enteros de 1 byte SIN SIGNO (0 a 255) |

**Total de bytes**: 2 + 2 + 1 + 1 = **6 bytes siempre**

#### Mapeo de Valores → Formato

```
Código formato: '>2h2B'
                 ↓ ↓ ↓
                 │ │ └─ aState (0 ó 1) → 1 byte sin signo
                 │ │    bState (0 ó 1) → 1 byte sin signo
                 │ └──── yValue (-2048 a 2047) → 2 bytes con signo
                 └─────── xValue (-2048 a 2047) → 2 bytes con signo

struct.pack('>2h2B', xValue, yValue, aState, bState)
                      └──────┬──────┘ └────┬────┘
                             │            │
                          2 shorts    2 bytes
```

---

### Paso 3: Comparación ASCII vs Binario

#### Tabla Comparativa

| Aspecto | ASCII (Unidad 4) | Binario (Unidad 5) |
|---------|-----|--------|
| **Formato de datos** | Texto legible: `"500,524,1,0\n"` | Bytes compactos: `01 F4 02 0C 01 00` |
| **Tamaño variable** | SÍ (de 7 a 25+ bytes) | NO (siempre 6 bytes) |
| **Tamaño promedio** | ~15-19 bytes | 6 bytes |
| **Reducción de ancho de banda** | 100% (baseline) | **60-70% menor** |
| **Velocidad transmisión** | ~190-230 bytes/seg | ~60 bytes/seg |
| **Latencia** | 1.5-2ms/dato | ~0.3-0.5ms/dato |
| **Legibilidad humana** | ✅ Fácil leer en terminal | ❌ Requiere decodificar |
| **Checksum** | Más complejo (texto) | Más simple (binario) |
| **Parsing en receptor** | String split + conversión | Unpack directo |

---

### Paso 4: Ventajas y Desventajas

#### ✅ Ventajas del Protocolo Binario

#### 1. **Menor consumo de ancho de banda**
```
ASCII: "500,524,1,0\n"        → 11 bytes
Binario: 01 F4 02 0C 01 00    → 6 bytes

Ahorro: (11-6)/11 = 45% menos datos
```

#### 2. **Latencia reducida**
```
A 115200 baud (14,400 bytes/seg):

ASCII (11 bytes):   11 / 14,400 seg ≈ 0.76 ms
Binario (6 bytes):  6 / 14,400 seg ≈ 0.42 ms

Mejora: 45% más rápido
```

#### 3. **Tamaño predecible**
```
Siempre 6 bytes → Buffer de tamaño fijo
✅ Receptor sabe exactamente cuándo termina el paquete
❌ ASCII requiere buscar '\n' (variable)
```

#### 4. **Parsing más eficiente**
```python
# ASCII: mucha lógica
parts = data.split(',')
x = int(parts[0])
y = int(parts[1])
a = bool(parts[2])  # "True" o "False" → conversión

# Binario: directo
x, y, a, b = struct.unpack('>2h2B', data)
# ✅ Una línea, sin conversión
```

#### 5. **Menor error de parsing**
```
ASCII: riesgo de "True" vs "true", espacios extra, etc
Binario: no hay ambigüedad, los bytes son los bytes
```

#### 6. **Escalabilidad**
```
Si agregamos otro sensor:
ASCII: "500,524,1,0,temp,humidity\n" → 30+ bytes (muy grande)
Binario: 01 F4 02 0C 01 00 [+ más bytes] → siempre compacto
```

---

#### ❌ Desventajas del Protocolo Binario

#### 1. **No legible sin decodificación**
```
Ves: 01 F4 02 0C 01 00
¿Qué significa? Necesitas saber el formato struct.pack()

ASCII: Ves "500,524,1,0" → ¡evidente!
```

#### 2. **Debugging más difícil**
```
ASCII: puedes ver los datos en terminal HyperTerminal, PuTTY, etc
Binario: necesitas herramienta especial o decodificador

Solución: Logger binario o analizador de protocolos
```

#### 3. **Sensibilidad al endianness**
```
Big-endian vs Little-endian puede causar confusión

500 en big-endian:    01 F4
500 en little-endian: F4 01  ← ¡Diferente!

Ambos lados deben acordar el formato (>= big-endian)
```

#### 4. **Requiere tipo exactamente**
```
ASCII: "500" funciona aunque sea int, float, string
Binario: struct.pack espera exactamente 'h' (2-byte signed int)

Si envías float en lugar de int → ERROR de tipo
```

#### 5. **No es extensible al mismo tamaño**
```
ASCII: puedes agregar campos sin límite razonable
Binario: cambiar formato requiere cambiar tamaño de paquete
         → compatibilidad break entre versiones
```

---

### Paso 5: Análisis Detallado de Conversión Hexadecimal

#### Ejemplo Completo: xValue=500, yValue=524, aState=True, bState=False

##### Paso 5.1: Convertir valores a binario

**xValue = 500 (signed short, 2 bytes)**
```
500 en binario: 0000 0001 1111 0100
               ↑ byte alto   ↑ byte bajo

En hexadecimal:
  Byte alto (bits 8-15):   0000 0001 = 0x01
  Byte bajo (bits 0-7):    1111 0100 = 0xF4
  
Orden big-endian: 01 F4
```

**Verificación**:
```
01 F4 (hex) = (01 × 256) + F4 = 256 + 244 = 500 ✓
```

---

**yValue = 524 (signed short, 2 bytes)**
```
524 en binario: 0000 0010 0000 1100
               ↑ byte alto   ↑ byte bajo

En hexadecimal:
  Byte alto (bits 8-15):   0000 0010 = 0x02
  Byte bajo (bits 0-7):    0000 1100 = 0x0C
  
Orden big-endian: 02 0C
```

**Verificación**:
```
02 0C (hex) = (02 × 256) + 0C = 512 + 12 = 524 ✓
```

---

**aState = True → int(True) = 1 (unsigned byte, 1 byte)**
```
1 en binario: 0000 0001
```

**En hexadecimal**:
```
0000 0001 = 0x01
```

---

**bState = False → int(False) = 0 (unsigned byte, 1 byte)**
```
0 en binario: 0000 0000
```

**En hexadecimal**:
```
0000 0000 = 0x00
```

---

##### Paso 5.2: Paquete Completo

```
struct.pack('>2h2B', 500, 524, 1, 0)

      xValue=500  yValue=524  aState=1  bState=0
           │          │          │        │
           ▼          ▼          ▼        ▼
Hex:    01 F4    02 0C        01        00

Orden en paquete: [xValue bytes][yValue bytes][aState byte][bState byte]
                   01 F4         02 0C          01           00

Resultado final: 01 F4 02 0C 01 00
```

---

#### Tabla de Referencia: Conversiones Hexadecimales

| Valor | Tipo | Rango | Ejemplo (500) | Ejemplo (1) |
|-------|------|-------|---------------|------------|
| **Short (2h)** | signed int 2 bytes | -32768 a 32767 | 01 F4 | N/A |
| **Byte (B)** | unsigned int 1 byte | 0 a 255 | N/A | 01 |

---

### Paso 6: Análisis de Eficiencia

#### Consumo de Ancho de Banda (10 Hz)

**Protocolo ASCII**:
```
Tamaño promedio: 15 bytes
Frecuencia: 10 Hz (cada 100ms)

Datos/segundo = 15 bytes × 10 Hz = 150 bytes/seg

A 115200 baud:
  Utilización: 150 × 8 bits / 115200 bits/seg = 10.4%
  Disponible para protocolo overhead: 89.6%
```

**Protocolo Binario**:
```
Tamaño fijo: 6 bytes
Frecuencia: 10 Hz (sin cambios)

Datos/segundo = 6 bytes × 10 Hz = 60 bytes/seg

A 115200 baud:
  Utilización: 60 × 8 bits / 115200 bits/seg = 4.2%
  Disponible para protocolo overhead: 95.8%
```

**Mejora**:
```
(150 - 60) / 150 = 60% reducción de datos brutos

Equivalencia: con binario podrías enviar a 100 Hz
           en lugar de 10 Hz y aún usar menos ancho de banda
```

---

### Latencia de Transmisión

**ASCII (15 bytes)**:
```
Tiempo = 15 bytes × 8 bits/byte / 115200 bits/seg
       = 120 bits / 115200 bits/seg
       = 1.04 ms
```

**Binario (6 bytes)**:
```
Tiempo = 6 bytes × 8 bits/byte / 115200 bits/seg
       = 48 bits / 115200 bits/seg
       = 0.42 ms
```

**Mejora**:
```
(1.04 - 0.42) / 1.04 = 60% menos latencia
```

---

### Paso 7: Observaciones Críticas (Registro)

#### Observación 1: Tamaño es el rey 👑

**Insight**: La principal ventaja de binario es el **tamaño fijo y predecible**.

```
❌ ASCII: máquina de estados compleja (buscar \n, buffer variable)
✅ Binario: algoritmo trivial (leer 6 bytes, fin)
```

**Impacto en el receptor**:
- Microcontroller puede pre-allocate buffer exacto
- Sin riesgo de overflow por tamaño variable
- Parsing O(1) en lugar de O(n)

---

#### Observación 2: Tradeoff Legibilidad vs Eficiencia

```
Escala de tradeoff:

ASCII ◄─────────────────────────► Binario
 ✅ Legible          ✅ Eficiente
 ❌ Grande           ❌ Opaco
 ❌ Lento            ✅ Rápido
```

**Decisión de diseño**: ¿Cuándo elegir cada uno?

| Contexto | Elecci Recomendada |
|----------|-------------------|
| **Debugging en desarrollo** | ASCII (legible en terminal) |
| **Producción (IoT/embedded)** | Binario (eficiencia) |
| **Uso educativo** | ASCII (enseña conceptos) |
| **Sistema crítico (real-time)** | Binario (latencia) |

---

#### Observación 3: Endianness es crucial

```python
# Big-endian (>) - "network order"
struct.pack('>h', 500)  → 01 F4

# Little-endian (<) - x86/ARM típico  
struct.pack('<h', 500)  → F4 01

# ERROR SILENCIOSO: si receptor recibe 01 F4 pero espera F4 01
  Resultado: 500 se interpreta como 63489 ❌
```

**Lección**: Ambos lados **deben acordar** endianness.

---

#### Observación 4: Prototipado vs Producción

**Fase 1 (Unidad 4): ASCII**
```
✅ Fácil de entender
✅ Fácil de debuggear
✅ Flexible para cambios
✅ Legible en terminal (hyper terminal, etc)
❌ Ineficiente
```

**Fase 2 (Unidad 5): Binario**
```
✅ Eficiente
✅ Predecible
✅ Escalable
✅ CPU del receptor no trabaja tanto en parsing
❌ Necesita decodificador especial
❌ Menos flexible
```

**Lección**: Binario es una **optimización posterior**, NO el punto de partida.

---

### Paso 8: Predicción para la Siguiente Actividad

#### ¿Qué viene después?

```
Unidad 5, Actividad 02:
├─ Implementar MicrobitBinaryAdapter.js
├─ Parsear struct.pack() en Node.js
├─ Validar integridad (sin checksum ASCII)
├─ Medir mejora de latencia real
└─ Actualizar p5.js (sin cambios, es transparente)
```

---

### Preguntas para Reflexión 🤔

#### P1: ¿Por qué struct.pack() es mejor que hacer la conversión manualmente?

**Respuesta registrada**:
- Evita errores de endianness
- Código más legible
- Funciones optimizadas en C (código subyacente)
- Estándar industria (portabilidad)

---

#### P2: ¿Qué sucedería si el micro:bit envia big-endian pero Node.js intenta leer little-endian?

**Respuesta registrada**:
```python
# Micro:bit envía (big-endian):
struct.pack('>h', 500) → 01 F4

# Node.js lee (little-endian, ERROR):
struct.unpack('<h', b'\x01\xF4') → 63489

# Resultado: sensor lee valores completamente errados
```

**Impacto**: Dibuja polígonos en posiciones aleatorias, usuario confundido

---

#### P3: ¿Cuáles sensores tenemos en micro:bit?

**Respuesta registrada**:
- Acelerómetro (3 ejes, aunque solo usamos X, Y)
- Brújula magnética
- Termómetro
- 2 botones (A, B)
- GPIO pins (más sensores posibles)

**¿Qué cambiaría en binario?**
```
Protocolo actual: '>2h2B' = 6 bytes (X, Y, A, B)

Con 3 ejes (X, Y, Z): '>3h2B' = 8 bytes
Con temperatura: '>2h2Bh' = 8 bytes
Con múltiples sensores: > 10 bytes (pero aún < ASCII)
```

#### Lo que aprendimos:

1. **Struct.pack()**: Empaqueta valores Python a bytes binarios
2. **Formato '>2h2B'**: 2 short big-endian + 2 unsigned bytes
3. **Tamaño**: Siempre 6 bytes vs variable ASCII
4. **Latencia**: 60% menos con binario
5. **Tradeoff**: Eficiencia vs legibilidad
6. **Big-endian**: Ambos extremos deben coincidir

#### Código clave a recordar:

```python
# Micro:bit
import struct
data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
uart.write(data)

# Node.js (próxima actividad)
const unpacked = struct.unpack('>2h2B', buffer);
```

## Bitácora de aplicación 


## Bitácora de reflexión
