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


### Actividad 04

<details>

<summary>index.html</summary>

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sketch</title>
    <link rel="stylesheet" type="text/css" href="style.css">
    <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
    <script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script>
  </head>
  <body>
    <script src="sketch.js"></script>
  </body>
</html>
```

</details>

<details>

<summary>sketch.js</summary>

``` js
// ── FSM ───────────────────────────────────────────────────────────────────
const ENTRY = "ENTRY";
const EXIT = "EXIT";

class Timer {
  constructor(owner, eventToPost, duration) {
    this.owner = owner;
    this.event = eventToPost;
    this.duration = duration;
    this.startTime = 0;
    this.active = false;
  }

  start(newDuration = null) {
    if (newDuration !== null) this.duration = newDuration;
    this.startTime = millis();
    this.active = true;
  }

  stop() {
    this.active = false;
  }

  update() {
    if (this.active && millis() - this.startTime >= this.duration) {
      this.active = false;
      this.owner.postEvent(this.event);
    }
  }
}

class FSMTask {
  constructor() {
    this.queue = [];
    this.timers = [];
    this.state = null;
  }

  postEvent(ev) {
    this.queue.push(ev);
  }

  addTimer(event, duration) {
    let t = new Timer(this, event, duration);
    this.timers.push(t);
    return t;
  }

  transitionTo(newState) {
    if (this.state) this.state(EXIT);
    this.state = newState;
    this.state(ENTRY);
  }

  update() {
    for (let t of this.timers) {
      t.update();
    }
    while (this.queue.length > 0) {
      let ev = this.queue.shift();
      if (this.state) this.state(ev);
    }
  }
}

// ── Temporizador ──────────────────────────────────────────────────────────
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};

class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();
    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;
    this.keyHistory = [];
    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);
  }

  get currentState() {
    return this.state;
  }

  postEvent(ev) {
    super.postEvent(ev);
    if (ev === EVENTS.DEC || ev === EVENTS.INC) {
      this.keyHistory.push(ev);
      if (this.keyHistory.length > 3) this.keyHistory.shift();
    }
  }

  isSequence(seq) {
    return this.keyHistory.join('') === seq;
  }

  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
      this.keyHistory = [];
    } else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start();
    } else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      }
    } else if (ev === EVENTS.DEC) {
      if (this.isSequence(EVENTS.DEC + EVENTS.INC + EVENTS.DEC)) {
        this.transitionTo(this.estado_config);
      } else {
        this.transitionTo(this.estado_paused);
      }
    } else if (ev === EXIT) {
      this.myTimer.stop();
    }
  };

  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };

  estado_paused = (ev) => {
    if (ev === ENTRY) {
    } else if (ev === EVENTS.DEC) {
      if (this.isSequence(EVENTS.DEC + EVENTS.INC + EVENTS.DEC)) {
        this.transitionTo(this.estado_config);
      } else {
        this.transitionTo(this.estado_armed);
      }
    }
  };
}

// ── p5.js ─────────────────────────────────────────────────────────────────
let temporizador;
let port;
let connectBtn;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  textAlign(CENTER, CENTER);

  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );

  renderer.set(temporizador.estado_config, () => drawConfig(temporizador.configValue));
  renderer.set(temporizador.estado_armed, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds, false));
  renderer.set(temporizador.estado_paused, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds, true));
  renderer.set(temporizador.estado_timeout, () => drawTimeout());

  port = createSerial();
  connectBtn = createButton('🔌 Conectar micro:bit');
  connectBtn.position(10, 10);
  connectBtn.mousePressed(() => {
    if (!port.opened()) {
      port.open('MicroPython', 115200);
    } else {
      port.close();
    }
  });
}

function draw() {
  if (!port.opened()) {
    connectBtn.html('🔌 Conectar micro:bit');
  } else {
    connectBtn.html('✅ Desconectar');
  }

  if (port.availableBytes() > 0) {
    let data = port.readUntil('\n');
    if (data) {
      data = data.trim().toUpperCase();
      if (data === 'A') temporizador.postEvent(EVENTS.DEC);
      if (data === 'B') temporizador.postEvent(EVENTS.INC);
      if (data === 'X') temporizador.postEvent(EVENTS.START);
    }
  }

  temporizador.update();
  renderer.get(temporizador.currentState)?.();
}

function drawConfig(val) {
  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);
  textSize(18);
  fill(200);
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total, paused) {
  background(20, 20, 20);
  let pulse = sin(frameCount * 0.1) * 10;

  noFill();
  strokeWeight(20);
  if (paused) {
    stroke(200, 0, 0, 100);
  } else {
    stroke(255, 100, 0, 50);
  }
  ellipse(width / 2, height / 2, 250);

  stroke(paused ? color(200, 0, 0) : color(255, 150, 0));
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100 + pulse);
  text(val + (paused ? "\nPAUSADO" : ""), width / 2, height / 2);
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent(EVENTS.DEC);
  if (key === "b" || key === "B") temporizador.postEvent(EVENTS.INC);
  if (key === "s" || key === "S") temporizador.postEvent(EVENTS.START);
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```

</details>

## Bitácora de reflexión

### Actiidad 05

<details>

<summary>fsm.js</summary>

``` js
const ENTRY = "ENTRY";
const EXIT = "EXIT";

class Timer {
  constructor(owner, eventToPost, duration) {
    this.owner = owner;
    this.event = eventToPost;
    this.duration = duration;
    this.startTime = 0;
    this.active = false;
  }

  start(newDuration = null) {
    if (newDuration !== null) this.duration = newDuration;
    this.startTime = millis();
    this.active = true;
  }

  stop() {
    this.active = false;
  }

  update() {
    if (this.active && millis() - this.startTime >= this.duration) {
      this.active = false;
      this.owner.postEvent(this.event);
    }
  }
}

class FSMTask {
  constructor() {
    this.queue = [];
    this.timers = [];
    this.state = null;
  }

  postEvent(ev) {
    this.queue.push(ev);
  }

  addTimer(event, duration) {
    let t = new Timer(this, event, duration);
    this.timers.push(t);
    return t;
  }

  transitionTo(newState) {
    if (this.state) this.state(EXIT);
    this.state = newState;
    this.state(ENTRY);
  }

  update() {
    for (let t of this.timers) {
      t.update();
    }
    while (this.queue.length > 0) {
      let ev = this.queue.shift();
      if (this.state) this.state(ev);
    }
  }
}
```

</details>

<details>

<summary>index.html</summary>

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>Sketch</title>

    <link rel="stylesheet" type="text/css" href="style.css">

    <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
    <!-- opcional: para comunicarse con micro:bit vía USB serial -->
    <script src="https://cdn.jsdelivr.net/npm/p5.serialport@1.0.3/lib/p5.serialport.js"></script>
  </head>

  <body>
    <script src="fsm.js"></script>
    <script src="sketch.js"></script>
  </body>
</html>
```

</details>

<details>

<summary>microbit_local.py</summary>

``` py
# Código para micro:bit local (MicroPython)
# actúa como puente entre el micro:bit remoto y el PC

from microbit import *
import radio
import utime

# usar el mismo grupo que el remoto
radio.config(group=42)
radio.on()

while True:
    # leer mensajes del remoto y reenviarlos por UART
    msg = radio.receive()
    if msg:
        uart.write(msg + '\n')
    # también aceptamos botones locales
    if button_a.was_pressed():
        uart.write('A\n')
        radio.send('A')  # opcional: aviso al remoto
    if button_b.was_pressed():
        uart.write('B\n')
        radio.send('B')
    utime.sleep_ms(100)
```

</details>

<details>

<summary>microbit_remote.py</summary>

``` py
# Código para micro:bit remoto (MicroPython)
# envía comandos por radio al micro:bit local

from microbit import *
import radio
import utime

# el grupo debe ser único para tu pareja de micro:bits
radio.config(group=42)
radio.on()

while True:
    if button_a.was_pressed():
        radio.send('A')
    if button_b.was_pressed():
        radio.send('B')
    if accelerometer.was_gesture('shake'):
        radio.send('X')
    utime.sleep_ms(100)
```

</details>

<details>

<summary>sketch.js</summary>

``` js
// copia de la Actividad 4: el temporizador y la máquina de estados
// permanecen exactamente igual en esta carpeta 05. No se modifica la
// lógica; los cambios de la actividad 5 se hacen en los micro:bits y en
// la documentación.
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};

const UI = {
  dialSize: 250,
  ringWeight: 20,
  bigText: 100,
  configText: 120,
  helpText: 18,
};


class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    // history para detectar A-B-A
    this.keyHistory = [];

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);

  }

  get currentState() {
    return this.state;
  }

  // override postEvent para capturar la historia de teclas
  postEvent(ev) {
    super.postEvent(ev);
    if (ev === EVENTS.DEC || ev === EVENTS.INC) {
      // guardamos sólo las letras A y B
      this.keyHistory.push(ev);
      if (this.keyHistory.length > 3) this.keyHistory.shift();
    }
  }

  isSequence(seq) {
    return this.keyHistory.join('') === seq;
  }

  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
      this.keyHistory = []; // limpiamos el historial al entrar en configuración
    }
    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };


  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start();
    } else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      }
    } else if (ev === EVENTS.DEC) {
      // tecla A: pausa o secuencia
      if (this.isSequence(EVENTS.DEC + EVENTS.INC + EVENTS.DEC)) {
        this.transitionTo(this.estado_config);
      } else {
        this.transitionTo(this.estado_paused);
      }
    } else if (ev === EXIT) {
      this.myTimer.stop();
    }

  };

  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  }

  // estado nuevo: pausado
  estado_paused = (ev) => {
    if (ev === ENTRY) {
      // nothing special, timer is already stopped by exit of armed
    } else if (ev === EVENTS.DEC) {
      // pulsamos A mientras está pausado
      // primero verificamos secuencia ABA antes de reanudar
      if (this.isSequence(EVENTS.DEC + EVENTS.INC + EVENTS.DEC)) {
        this.transitionTo(this.estado_config);
      } else {
        // si no hay secuencia, volvemos a modo armado
        this.transitionTo(this.estado_armed);
      }
    }
    // no comprobamos la secuencia en ENTRY/EXIT para evitar recursion
  };
}

let temporizador;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
  textAlign(CENTER, CENTER);

  renderer.set(temporizador.estado_config, () => drawConfig(temporizador.configValue));
  renderer.set(temporizador.estado_armed, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds, false));
  renderer.set(temporizador.estado_paused, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds, true));
  renderer.set(temporizador.estado_timeout, () => drawTimeout());
}

function draw() {
  temporizador.update();
  renderer.get(temporizador.currentState)?.();
}

function drawConfig(val) {
  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);
  textSize(18);
  fill(200);
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total, paused) {
  background(20, 20, 20);
  let pulse = sin(frameCount * 0.1) * 10;

  noFill();
  strokeWeight(20);
  if (paused) {
    stroke(200, 0, 0, 100);
  } else {
    stroke(255, 100, 0, 50);
  }
  ellipse(width / 2, height / 2, 250);

  stroke(paused ? color(200, 0, 0) : color(255, 150, 0));
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100 + pulse);
  text(val + (paused ? "\nPAUSADO" : ""), width / 2, height / 2);
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent(EVENTS.DEC);
  if (key === "b" || key === "B") temporizador.postEvent(EVENTS.INC);
  if (key === "s" || key === "S") temporizador.postEvent(EVENTS.START);
}

// micro:bit serial support -------------------------------------------------
let serial;

function setupSerial() {
  serial = new p5.SerialPort();
  serial.on('data', serialEvent);
  // cambiar el puerto según corresponda; /dev/ttyACM0 es común en Linux
  serial.open('/dev/ttyACM0');
}

function serialEvent() {
  let data = serial.readStringUntil('\n');
  if (!data) return;
  data = data.trim().toUpperCase();
  if (data === 'A') temporizador.postEvent(EVENTS.DEC);
  if (data === 'B') temporizador.postEvent(EVENTS.INC);
  // puedes enviar otras letras desde el micro:bit para más acciones
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```

</details>

