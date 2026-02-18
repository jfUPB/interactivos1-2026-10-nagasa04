# Unidad 3

## Bitácora de proceso de aprendizaje

### Actividad 01

``` python
from microbit import *
import utime

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


class Semaforo:
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        self.event_queue = []
        self.timers = []
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow

        self.myTimer = self.createTimer("Timeout",self.timeInRed)

        # --- Nuevo timer para parpadeo nocturno ---
        self.blinkTimer = self.createTimer("Blink",300)

        self.estado_actual = None
        self.modo = "normal"  # --- Variable para controlar modo actual ---
        self.transicion_a(self.estado_waitInRed)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar timers
        for t in self.timers:
            t.update()

        # 2. Procesar eventos
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: 
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)

    # ------------------ ESTADOS NORMALES ------------------

    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)

        if ev == "Timeout":
            self.transicion_a(self.estado_waitInGreen)

        # --- Botón B activa modo nocturno ---
        if ev == "B":
            self.transicion_a(self.estado_nocturno)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)

        if ev == "Timeout":
            self.transicion_a(self.estado_waitInYellow)

        # --- Botón A activa modo peatonal ---
        if ev == "A":
            self.transicion_a(self.estado_waitInYellow)

        if ev == "B":
            self.transicion_a(self.estado_nocturno)

    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)

        if ev == "Timeout":
            if self.modo == "peatonal":
                self.transicion_a(self.estado_waitInRed)
                self.modo = "normal"
            else:
                self.transicion_a(self.estado_waitInRed)

        if ev == "B":
            self.transicion_a(self.estado_nocturno)

    # ------------------ MODO PEATONAL ------------------

    # --- Nuevo estado para activar transición segura a rojo ---
    def estado_peatonal(self, ev):
        if ev == "ENTRY":
            self.modo = "peatonal"
            self.transicion_a(self.estado_waitInYellow)

    # ------------------ MODO NOCTURNO ------------------

    # --- Nuevo estado nocturno con parpadeo amarillo ---
    def estado_nocturno(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.blink_on = True
            self.blinkTimer.start()

        if ev == "Blink":
            if self.blink_on:
                display.set_pixel(self.x,self.y+1,0)
            else:
                display.set_pixel(self.x,self.y+1,9)

            self.blink_on = not self.blink_on
            self.blinkTimer.start()

        # --- Botón A devuelve al modo normal ---
        if ev == "A":
            self.transicion_a(self.estado_waitInRed)


# ------------------ MAIN LOOP ------------------

semaforo1 = Semaforo(0,0,2000,1000,500)

while True:
    # --- Detectar botones y postear eventos ---
    if button_a.was_pressed():
        semaforo1.post_event("A")

    if button_b.was_pressed():
        semaforo1.post_event("B")

    semaforo1.update()
    utime.sleep_ms(20)
```

### Lógica del comportamiento
- Botón A (modo peatonal)

  - Si está en verde → pasa a amarillo
  - Luego rojo
  - Después vuelve al ciclo normal
  - Exactamente como un semáforo real. Nada improvisado.

- Botón B (modo nocturno)

  - Entra en estado especial
  - Parpadea amarillo usando un timer independiente
  - Botón A lo devuelve al modo normal

### Actividad 02

- main.py
``` python
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music

class Temporizador(FSMTask):
    def __init__(self):
        super().__init__()
        self.counter = 20
        self.myTimer = self.add_timer("Timeout",1000)

        # Control de pausa
        self.paused = False

        # Buffer para detectar secuencia A-B-A
        self.sequence = []

        self.transition_to(self.estado_config)
```

- Estado config
``` python
    def estado_config(self, ev):
        if ev == ENTRY:
            self.counter = 20
            display.show(FILL[self.counter])

        if ev == "A":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])

        if ev == "B":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])

        if ev == "S":
            self.transition_to(self.estado_armed)
```

- Estado armed
``` python
    def estado_armed(self, ev):

        if ev == ENTRY:
            self.paused = False
            self.sequence = []
            self.myTimer.start()

        # Botón A: pausa / reanuda
        if ev == "A":
            self.sequence.append("A")
            self.sequence = self.sequence[-3:]

            if not self.paused:
                self.myTimer.stop()
                self.paused = True
            else:
                self.myTimer.start()
                self.paused = False

        # Botón B: solo para secuencia
        if ev == "B":
            self.sequence.append("B")
            self.sequence = self.sequence[-3:]

        # Detectar A-B-A
        if self.sequence == ["A","B","A"]:
            self.transition_to(self.estado_config)
            return

        # Conteo normal
        if ev == "Timeout" and not self.paused:
            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])

                if self.counter == 0:
                    self.transition_to(self.estado_timeout)
                else:
                    self.myTimer.start()
```

- Estado timeout
``` python
    def estado_timeout(self, ev):
        if ev == ENTRY:
            display.show(Image.SKULL)
            music.play(music.FUNERAL)

        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)
```
## Bitácora de aplicación 



## Bitácora de reflexión
