# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 01
- ¿Qué es un sistema físico interactivo?
  
| Son componentes tanto fisicos como digitales que trabajan para detectar las interacciones de los usuarios o del entorno.
- ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
  
| Creando experiencias visuales por medio de diseño generativo que hace sentir al usuario diferentes experiencias 

### Actividad 02
- ¿Qué es el diseño/arte generativo?
  
| Es una exprecion creativa moderna que resulta de la fusion de: datos, storytelling, programacion, interacion y forma. Es un sistema que crea el arte.
- ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
  
| Podria hacer una ia que estudie mi arte y crear un diseño de un sistema que cree multiples salidas de este y tal vez lo haga mas facil hacerlo

### Actividad 03

- python.microbit
<img width="381" height="308" alt="image" src="https://github.com/user-attachments/assets/abb39265-dc85-42b1-b695-eaf9bc407511" />

<img width="1373" height="626" alt="image" src="https://github.com/user-attachments/assets/b9d84eb2-809c-4083-829c-edc43bb89d2d" />

https://github.com/user-attachments/assets/3737996d-03cd-4fa3-95f8-83822534e357


-p5.js
<img width="1406" height="603" alt="image" src="https://github.com/user-attachments/assets/05927900-7bb7-4e5f-b9b4-24ae103459e9" />

<img width="417" height="428" alt="image" src="https://github.com/user-attachments/assets/d2b9b08f-dfc5-4a15-83c8-f463eb098f1f" />

<img width="484" height="484" alt="image" src="https://github.com/user-attachments/assets/04ff4403-9a79-4a73-a490-33340f69cac9" />

### Actividad 04 

- ¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.

| was_pressed() : detecta eventos unicos como un clic

| is_pressed() : envia multiplas mensajes si el boton se mantiene precionado

| No funcionaba el programa con was_pressed() debido a que es un evento demasiado corto y el loop no lo atrapaba en cambio con is_pressed() mantiene el estado "true"


- button_a.was_pressed() para detectar si el botón ha sido presionado.
- button_a.is_pressed() si quieres saber si el botón está presionado en ese momento.
- was_pressed() es más adecuado para detectar eventos únicos como un clic.
- is_pressed(), el programa podría enviar múltiples mensajes si el botón se mantiene presionado.
- inicializar la comunicación serial con uart.init(baudrate=115200)
- utilizamos uart.write('A') para enviar el mensaje ‘A’ cuando se presiona el botón A.
- connectBtnClick() Esta función se ejecuta cuando el usuario hace click en el botón de conexión



## Bitácora de aplicación 

### Actividad 05

- El programa de p5.js.
- El programa de micro:bit.
- Explica detalladamente cómo funciona el sistema físico interactivo que has creado.

- micro:bit codigo
  
``` python
from microbit import *

# Activamos el puerto serial
uart.init(baudrate=115200)

while True:
    # Botón A → mover a la izquierda
    if button_a.is_pressed():
        uart.write("A")
    # no es  else: porque mandaria el bit constantemente
    # Botón B → mover a la derecha
    elif button_b.is_pressed():
        uart.write("B")

    # Pausa para evitar saturar el serial
    sleep(100)
```

- p5.js codigo
  
``` js
let port;
let connectBtn;

// Nueva variable: posición horizontal del círculo
let x;

// Se mantiene el control de inicialización
let connectionInitialized = false;

function setup() {
  // tamaño del lienzo
  createCanvas(400, 400);
  background(220);

  // Posición inicial del círculo
  x = width / 2;

  port = createSerial();

  connectBtn = createButton("Connect to micro:bit");
  connectBtn.position(80, 300);
  connectBtn.mousePressed(connectBtnClick);
}

function draw() {
  // Color del fondo (entre mas bajito el numero mas oscuro)
  background(220);

  // Limpieza del buffer solo una vez al conectar
  if (port.opened() && !connectionInitialized) {
    port.clear();
    connectionInitialized = true;
  }

  // Lectura de datos seriales
  if (port.availableBytes() > 0) {
    let dataRx = port.read(1);

    // Si llega "A", movemos a la izquierda
    if (dataRx == "A") {
      x -= 5;
    }

    // Si llega "B", movemos a la derecha
    else if (dataRx == "B") {
      x += 5;
    }
  }

  // Evitamos que el círculo se salga del canvas
  x = constrain(x, 20, width - 20);

  // Dibujamos el círculo (antes era un rectángulo)
  //color
  fill(200, 0, 200);
  noStroke();
  // el primer numero es la division vertical del canvas y el segundo es que tan grande es el circulo
  circle(x, height / 2, 40);

  // Texto dinámico del botón
  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  } else {
    connectBtn.html("Disconnect");
  }
}

function connectBtnClick() {
  if (!port.opened()) {
    port.open("MicroPython", 115200);
    connectionInitialized = false;
  } else {
    port.close();
  }
}
```

El codigo envia informacion en bits sobre si el micro:bit esta siendo precionado A o B. 
Si se preciona A se va moviendo a la izquierda en cambio si se preciona B el circulo se mueve a la derecha.

## Bitácora de reflexión









