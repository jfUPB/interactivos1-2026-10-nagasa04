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

``` phyton
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



## Bitácora de reflexión




