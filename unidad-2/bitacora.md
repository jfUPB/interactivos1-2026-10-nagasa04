# Unidad 2

## Bitácora de proceso de aprendizaje
### Actividad 01
- Estado: Es lo que esta esperando algo como una accion del usuario o una animacion
  - Los estados empiezan en en encendido (Circulo) o apagado (Flecha que va a pasar)  
- Evento: esperar a que ocurra algo como esperar a que un contador termine o que se termine una animacion
  - TIme out: cambiar de estado
- Una maquina de estado NO es un diagrama de flujo

1. ¿Cuáles son los estados en el programa?
- estado_waitInON
- estado_waitInOFF
2. ¿Cuáles son los eventos en el programa?
- ENTRY
- EXIT
- Timeout
3. ¿Cuáles son las acciones en el programa?
Las acciones del programa es cuando inicia el programa empieza a contar un tiempo y cuando el tiempo pasa prende el pixel dependiendo del estado en el que este

### Actividad 02
<img width="302" height="575" alt="image" src="https://github.com/user-attachments/assets/83f80128-a09a-4816-b80c-acc28ed1f52c" />

``` python
from microbit import *
import utime

# -------------------------------------------------
# CLASE TIMER
# Encargada de manejar el tiempo sin bloquear
# -------------------------------------------------
class Timer:
    def __init__(self, owner, event_to_post, duration):
        # owner: objeto que recibirá el evento (Semaforo)
        self.owner = owner
        # nombre del evento a enviar cuando termine el tiempo
        self.event = event_to_post
        # duración del temporizador en milisegundos
        self.duration = duration

        # momento en que el temporizador inicia
        self.start_time = 0
        # indica si el temporizador está activo
        self.active = False

    def start(self, new_duration=None):
        # permite cambiar la duración si se pasa un nuevo valor
        if new_duration is not None:
            self.duration = new_duration
        # guarda el tiempo actual
        self.start_time = utime.ticks_ms()
        # activa el temporizador
        self.active = True

    def stop(self):
        # desactiva el temporizador
        self.active = False

    def update(self):
        # solo revisa si el temporizador está activo
        if self.active:
            # calcula cuánto tiempo ha pasado
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                # cuando se cumple el tiempo, se apaga
                self.active = False
                # envía el evento "Timeout" al dueño
                self.owner.post_event(self.event)


# -------------------------------------------------
# CLASE SEMÁFORO (Máquina de Estados)
# -------------------------------------------------
class Semaforo:
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        # cola de eventos
        self.event_queue = []
        # lista de timers internos
        self.timers = []

        # posición del semáforo en el display
        self.x = _x
        self.y = _y

        # tiempos de cada estado
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow

        # se crea un temporizador inicial
        self.myTimer = self.createTimer("Timeout", self.timeInRed)

        # estado actual de la máquina
        self.estado_actual = None
        # el semáforo inicia en ROJO
        self.transicion_a(self.estado_waitInRed)

    def createTimer(self,event,duration):
        # crea un temporizador y lo guarda en la lista
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        # agrega eventos a la cola
        self.event_queue.append(ev)

    def update(self):
        # 1. actualizar todos los temporizadores
        for t in self.timers:
            t.update()

        # 2. procesar los eventos generados
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        # al salir de un estado se envía EXIT
        if self.estado_actual:
            self.estado_actual("EXIT")
        # se cambia el estado actual
        self.estado_actual = nuevo_estado
        # al entrar a un estado se envía ENTRY
        self.estado_actual("ENTRY")

    def clear(self):
        # apaga los tres LEDs del semáforo
        display.set_pixel(self.x, self.y, 0)     # rojo
        display.set_pixel(self.x, self.y+1, 0)   # amarillo
        display.set_pixel(self.x, self.y+2, 0)   # verde

    # -------------------------------------------------
    # ESTADO ROJO
    # -------------------------------------------------
    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            # al entrar, se apagan todos
            self.clear()
            # se enciende el rojo
            display.set_pixel(self.x, self.y, 9)
            # se inicia el temporizador del rojo
            self.myTimer.start(self.timeInRed)

        if ev == "Timeout":
            # cuando se acaba el tiempo, pasa a verde
            self.transicion_a(self.estado_waitInGreen)

    # -------------------------------------------------
    # ESTADO VERDE
    # -------------------------------------------------
    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            # se limpian los LEDs
            self.clear()
            # se enciende el verde
            display.set_pixel(self.x, self.y+2, 9)
            # se inicia el temporizador del verde
            self.myTimer.start(self.timeInGreen)

        if ev == "Timeout":
            # transición normal a amarillo
            self.transicion_a(self.estado_waitInYellow)

        if ev == "A":
            # EVENTO CLAVE DEL EJERCICIO:
            # si se presiona el botón A en verde,
            # se pasa inmediatamente a amarillo
            self.transicion_a(self.estado_waitInYellow)

    # -------------------------------------------------
    # ESTADO AMARILLO
    # -------------------------------------------------
    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            # se limpian los LEDs
            self.clear()
            # se enciende el amarillo
            display.set_pixel(self.x, self.y+1, 9)
            # se inicia el temporizador del amarillo
            self.myTimer.start(self.timeInYellow)

        if ev == "Timeout":
            # cuando termina el amarillo vuelve a rojo
            self.transicion_a(self.estado_waitInRed)


# -------------------------------------------------
# INSTANCIA DEL SEMÁFORO
# -------------------------------------------------
semaforo1 = Semaforo(0, 0, 2000, 1000, 500)

# -------------------------------------------------
# LOOP PRINCIPAL
# -------------------------------------------------
while True:
    # si se presiona el botón A, se envía el evento "A"
    if button_a.was_pressed():
        semaforo1.post_event("A")

    # se actualiza la máquina de estados
    semaforo1.update()

    # pequeña pausa para estabilidad
    utime.sleep_ms(20)
```

### Actividad 03

``` python
from microbit import *
import utime

# -------------------------------------------------
# CLASE TIMER
# Maneja el tiempo sin bloquear la ejecución
# -------------------------------------------------
class Timer:
    def __init__(self, owner, event_to_post, duration):
        # owner: objeto que recibirá el evento (Game)
        self.owner = owner
        # evento que se enviará al finalizar el tiempo
        self.event = event_to_post
        # duración del temporizador en milisegundos
        self.duration = duration

        # tiempo en que el temporizador inicia
        self.start_time = 0
        # indica si el temporizador está activo
        self.active = False

    def start(self, new_duration=None):
        # permite cambiar la duración si se pasa un nuevo valor
        if new_duration is not None:
            self.duration = new_duration
        # guarda el tiempo actual
        self.start_time = utime.ticks_ms()
        # activa el temporizador
        self.active = True

    def stop(self):
        # desactiva el temporizador
        self.active = False

    def update(self):
        # solo revisa si el temporizador está activo
        if self.active:
            # calcula cuánto tiempo ha pasado
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                # cuando el tiempo se cumple, se apaga
                self.active = False
                # se envía el evento "Timeout" al dueño
                self.owner.post_event(self.event)


# -------------------------------------------------
# CLASE GAME (Máquina de Estados)
# Controla las imágenes y los botones
# -------------------------------------------------
class Game:
    def __init__(self):
        # cola de eventos
        self.event_queue = []
        # lista de temporizadores
        self.timers = []

        # tiempos de cada estado
        self.timeInHeart = 2500
        self.timeInPacman = 1000
        self.timeInGhost = 2000

        # temporizador principal (se reutiliza)
        self.myTimer = self.createTimer("Timeout", self.timeInHeart)

        # estado actual de la máquina
        self.estado_actual = None
        # el juego inicia mostrando el corazón
        self.transicion_a(self.estado_waitInHeart)

    def createTimer(self,event,duration):
        # crea un temporizador y lo guarda
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        # agrega eventos a la cola
        self.event_queue.append(ev)

    def update(self):
        # 1. actualizar todos los temporizadores
        for t in self.timers:
            t.update()

        # 2. procesar los eventos pendientes
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        # notifica salida del estado actual
        if self.estado_actual:
            self.estado_actual("EXIT")
        # cambia el estado
        self.estado_actual = nuevo_estado
        # notifica entrada al nuevo estado
        self.estado_actual("ENTRY")


    # -------------------------------------------------
    # ESTADO CORAZÓN
    # -------------------------------------------------
    def estado_waitInHeart(self, ev):
        if ev == "ENTRY":
            # se muestra la imagen del corazón
            display.show(Image.HEART)
            # se inicia el temporizador del corazón
            self.myTimer.start(self.timeInHeart)

        if ev == "Timeout":
            # al terminar el tiempo, pasa a Pacman
            self.transicion_a(self.estado_waitInPacman)

        if ev == "A":
            # si se presiona A, salta inmediatamente a Ghost
            self.transicion_a(self.estado_waitInGhost)


    # -------------------------------------------------
    # ESTADO PACMAN
    # -------------------------------------------------
    def estado_waitInPacman(self, ev):
        if ev == "ENTRY":
            # se muestra la imagen de Pacman
            display.show(Image.PACMAN)
            # se inicia el temporizador de Pacman
            self.myTimer.start(self.timeInPacman)

        if ev == "Timeout":
            # al terminar el tiempo, pasa a Ghost
            self.transicion_a(self.estado_waitInGhost)

        if ev == "A":
            # si se presiona A, vuelve al corazón
            self.transicion_a(self.estado_waitInHeart)


    # -------------------------------------------------
    # ESTADO GHOST
    # -------------------------------------------------
    def estado_waitInGhost(self, ev):
        if ev == "ENTRY":
            # se muestra la imagen del fantasma
            display.show(Image.GHOST)
            # se inicia el temporizador del fantasma
            self.myTimer.start(self.timeInGhost)

        if ev == "Timeout":
            # al terminar el tiempo, vuelve al corazón
            self.transicion_a(self.estado_waitInHeart)

        if ev == "A":
            # si se presiona A, pasa a Pacman
            self.transicion_a(self.estado_waitInPacman)


# -------------------------------------------------
# INSTANCIA DEL JUEGO
# -------------------------------------------------
game = Game()

# -------------------------------------------------
# LOOP PRINCIPAL
# -------------------------------------------------
while True:
    # si se presiona el botón A,
    # se envía el evento "A" a la máquina de estados
    if button_a.was_pressed():
        game.post_event("A")

    # se actualiza la máquina de estados
    game.update()

    # pequeña pausa para estabilidad
    utime.sleep_ms(20)
```

- Creo que logre entender el funcionamientode el codigos mostrados


## Bitácora de aplicación 

### Actividad 04

``` python
from microbit import *
import utime

# --------------------------------------------------
# Función obligatoria para crear imágenes de llenado
# --------------------------------------------------
def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()

# --------------------
# Clase Timer (dada)
# --------------------
class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

# --------------------------------------------------
# Máquina de estados del temporizador
# --------------------------------------------------
class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []

        # Valor inicial del temporizador
        self.count = 20

        # Timer de 1 segundo
        self.myTimer = self.createTimer("Timeout", 1000)

        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # Actualiza timers
        for t in self.timers:
            t.update()

        # Procesa eventos
        while self.event_queue:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    # --------------------
    # ESTADO CONFIGURACIÓN
    # --------------------
    def estado_config(self, ev):
        if ev == "ENTRY":
            display.show(FILL[self.count])

        if ev == "A":
            if self.count < 25:
                self.count += 1
                display.show(FILL[self.count])

        if ev == "B":
            if self.count > 15:
                self.count -= 1
                display.show(FILL[self.count])

        if ev == "S":
            self.transicion_a(self.estado_countdown)

    # --------------------
    # ESTADO CUENTA REGRESIVA
    # --------------------
    def estado_countdown(self, ev):
        if ev == "ENTRY":
            display.show(FILL[self.count])
            self.myTimer.start(1000)

        if ev == "Timeout":
            self.count -= 1
            if self.count > 0:
                display.show(FILL[self.count])
                self.myTimer.start(1000)
            else:
                self.transicion_a(self.estado_end)

    # --------------------
    # ESTADO FINAL
    # --------------------
    def estado_end(self, ev):
        if ev == "ENTRY":
            display.show(Image.SKULL)
            audio.play(Sound.GIGGLE)

        if ev == "A":
            self.count = 20
            self.transicion_a(self.estado_config)

# --------------------
# Loop principal
# --------------------
task = Task()

while True:
    if button_a.was_pressed():
        task.post_event("A")

    if button_b.was_pressed():
        task.post_event("B")

    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)
```

<img width="300" height="547" alt="image" src="https://github.com/user-attachments/assets/62fcb7d3-af65-4f21-acb0-ed6a43f6f32a" />




## Bitácora de reflexión

### Actividad 05

Temporizador interactivo con maquinas de estado

### Paso 1: Entender el problema
El temporizador tiene 3 estados:

1. CONFIGURACIÓN: Pantalla muestra pixeles, botones A/B suben/bajan el tiempo (15-25), shake(sacudir el microbit) arma el temporizador.

2. CUENTA REGRESIVA: Se apaga un pixel cada segundo hasta llegar a 0.

3. ALARMA: Muestra calavera + sonido. Botón A vuelve a configuración.

### Paso 2: Diagrama en PlantUML
<img width="1168" height="308" alt="{DC57B5DB-8DC3-4322-A3EE-0E2AEFA8C642}" src="https://github.com/user-attachments/assets/c8f07bde-c16c-4b79-9079-c06d6293d24b" />


### Paso 3: El código completo

``` phyton
from microbit import *
import utime

# ============================================================
# 1. FUNCIÓN PARA CREAR IMÁGENES DE LLENADO (dada por el profe)
# ============================================================
# Genera 26 imágenes (0 a 25 pixeles encendidos).
# FILL[0] = todo apagado, FILL[25] = todo encendido.
# Los pixeles se llenan fila por fila, de arriba-izquierda
# a abajo-derecha.

def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()

# ============================================================
# 2. CLASE TIMER (dada por el profe)
# ============================================================
# Maneja tiempos sin usar sleep().
# Cuando el tiempo se cumple, posta un evento a la máquina.

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

# ============================================================
# 3. IMAGEN DE CALAVERA (para el estado de alarma)
# ============================================================

SKULL = Image(
    "09990:"
    "99099:"
    "09990:"
    "09990:"
    "90909"
)

# ============================================================
# 4. CLASE TASK — LA MÁQUINA DE ESTADOS
# ============================================================
# Aquí está toda la lógica del temporizador.
#
# ESTADOS:
#   estado_configuracion  -> el usuario ajusta los pixeles
#   estado_cuenta         -> cuenta regresiva (1 pixel/segundo)
#   estado_alarma         -> muestra calavera y suena speaker
#
# EVENTOS:
#   "A"       -> botón A presionado
#   "B"       -> botón B presionado
#   "S"       -> gesto shake detectado
#   "Timeout" -> el Timer de 1 segundo se cumplió
#   "ENTRY"   -> se acaba de entrar a este estado
#   "EXIT"    -> se está saliendo de este estado

class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []

        # Timer de 1 segundo para la cuenta regresiva
        self.countdown_timer = self.createTimer("Timeout", 1000)

        # Variables del temporizador
        self.pixels = 20          # valor inicial: 20 pixeles
        self.pixels_actual = 20   # pixeles que quedan en cuenta regresiva

        # Estado inicial
        self.estado_actual = None
        self.transicion_a(self.estado_configuracion)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers
        for t in self.timers:
            t.update()
        # 2. Procesar cola de eventos
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    # --------------------------------------------------------
    # ESTADO: CONFIGURACIÓN
    # --------------------------------------------------------
    # - Muestra en pantalla la cantidad de pixeles configurados.
    # - Botón A: sube pixeles (máximo 25).
    # - Botón B: baja pixeles (mínimo 15).
    # - Shake: arma el temporizador → pasa a cuenta regresiva.

    def estado_configuracion(self, ev):
        if ev == "ENTRY":
            # Al entrar, mostramos los pixeles configurados
            display.show(FILL[self.pixels])

        elif ev == "A":
            # Aumentar pixeles (máximo 25)
            if self.pixels < 25:
                self.pixels += 1
                display.show(FILL[self.pixels])

        elif ev == "B":
            # Disminuir pixeles (mínimo 15)
            if self.pixels > 15:
                self.pixels -= 1
                display.show(FILL[self.pixels])

        elif ev == "S":
            # Shake: armar temporizador → ir a cuenta regresiva
            self.pixels_actual = self.pixels
            self.transicion_a(self.estado_cuenta)

        elif ev == "EXIT":
            pass

    # --------------------------------------------------------
    # ESTADO: CUENTA REGRESIVA
    # --------------------------------------------------------
    # - Se apaga 1 pixel cada segundo (de abajo-derecha hacia
    #   arriba-izquierda).
    # - Cuando llega a 0 → pasa a alarma.
    # - No responde a botones durante la cuenta.

    def estado_cuenta(self, ev):
        if ev == "ENTRY":
            # Mostrar los pixeles actuales e iniciar el timer
            display.show(FILL[self.pixels_actual])
            self.countdown_timer.start(1000)

        elif ev == "Timeout":
            # Se cumplió 1 segundo: apagar un pixel
            self.pixels_actual -= 1

            if self.pixels_actual > 0:
                # Aún quedan pixeles: mostrar y reiniciar timer
                display.show(FILL[self.pixels_actual])
                self.countdown_timer.start(1000)
            else:
                # Llegó a 0: mostrar el último frame (todo apagado)
                # y pasar a alarma
                display.show(FILL[0])
                self.transicion_a(self.estado_alarma)

        elif ev == "EXIT":
            # Detener el timer al salir
            self.countdown_timer.stop()

    # --------------------------------------------------------
    # ESTADO: ALARMA
    # --------------------------------------------------------
    # - Muestra la calavera en pantalla.
    # - Suena el speaker.
    # - Botón A: vuelve a configuración con 20 pixeles.

    def estado_alarma(self, ev):
        if ev == "ENTRY":
            # Mostrar calavera y sonar
            display.show(SKULL)
            music.play(music.BADDY, wait=False)

        elif ev == "A":
            # Reiniciar: volver a configuración con 20 pixeles
            self.pixels = 20
            self.transicion_a(self.estado_configuracion)

        elif ev == "EXIT":
            pass

# ============================================================
# 5. CICLO PRINCIPAL
# ============================================================
# Aquí se generan los eventos de hardware y se actualiza
# la máquina de estados. NO se usa sleep() largo,
# solo sleep_ms(20) para no saturar el procesador.

import music

task = Task()

while True:
    # Generar eventos de los sensores
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    # Actualizar la máquina de estados y sus timers
    task.update()

    # Pequeña pausa para no saturar el CPU
    utime.sleep_ms(20)
```
### TEMPORIZADOR INTERACTIVO CON MÁQUINA DE ESTADOS — EXPLICACIÓN PASO A PASO

¿QUÉ ES UNA MÁQUINA DE ESTADOS?
Piensa en ella como un diagrama de flujo donde tu programa solo puede estar en UN estado a la vez, y salta de uno a otro cuando ocurre un EVENTO (presionar un botón, que pase un segundo, etc.).


LOS 3 ESTADOS DEL TEMPORIZADOR

1. CONFIGURACIÓN
   - Qué hace: Muestra pixeles en pantalla. Botón A sube, Botón B baja (entre 15 y 25).
   - Cómo se sale: Shake → va a Cuenta Regresiva.

2. CUENTA REGRESIVA
   - Qué hace: Cada segundo apaga un pixel (de abajo hacia arriba).
   - Cómo se sale: Cuando llega a 0 → va a Alarma.

3. ALARMA
   - Qué hace: Muestra calavera + sonido.
   - Cómo se sale: Botón A → vuelve a Configuración (reset a 20 pixeles).

¿POR QUÉ NO USAMOS sleep()?
----------------------------
Si usaras sleep(1000) para esperar 1 segundo, el programa se CONGELA y no puede detectar botones durante ese tiempo. En cambio, el Timer revisa constantemente si ya pasó el tiempo, y cuando se cumple, genera el evento "Timeout" sin bloquear nada.


¿CÓMO FUNCIONA EL FLUJO DE EVENTOS?
-------------------------------------
1. El "while True" del ciclo principal detecta si presionaste A, B o hiciste shake.
2. Esos eventos se meten en una COLA (event_queue).
3. task.update() primero revisa los timers (¿ya pasó 1 segundo?) y luego procesa cada evento de la cola, enviándolo al estado actual.
4. El estado actual decide qué hacer con ese evento.


¿CÓMO FUNCIONA FILL[n]?
------------------------
FILL es una lista de 26 imágenes (de FILL[0] a FILL[25]).
- FILL[0] tiene todos los LEDs apagados.
- FILL[20] tiene 20 LEDs encendidos (4 filas completas).
- FILL[25] tiene los 25 encendidos.
Así visualizamos cuántos "segundos" quedan.


¿CÓMO SE APAGAN LOS PIXELES DE ABAJO HACIA ARRIBA?
----------------------------------------------------
Simplemente decrementamos pixels_actual (de 20 a 19, luego 18, etc.) y mostramos FILL[pixels_actual]. Como FILL llena de arriba-izquierda a abajo-derecha, al reducir el número se "apagan" desde abajo-derecha hacia arriba-izquierda. Exactamente lo que pide el enunciado.
