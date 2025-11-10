# Presentacion proyecto
##### Caos Organico: Jean Arp y Suika Game

<img src="https://raw.githubusercontent.com/DannyCatalan/Interfaz-II/refs/heads/main/img/circuito%20final.jpg"/>

###### Codigo de Arduino
```js
// --- Pines de conexión ---
const int btnColor = 4;
const int btnSize  = 6;
const int btnDrop  = 8;
const int btnReset = 10;

const int joyX = A0;
const int joyY = A1;
const int joyBtn = 2;  // si usas el botón del joystick

// LED RGB (de cátodo común)
const int ledR = 9;
const int ledG = 11;
const int ledB = 13;

void setup() {
  Serial.begin(9600);

  // Botones con resistencia pull-up interna
  pinMode(btnColor, INPUT_PULLUP);
  pinMode(btnSize,  INPUT_PULLUP);
  pinMode(btnDrop,  INPUT_PULLUP);
  pinMode(btnReset, INPUT_PULLUP);
  pinMode(joyBtn, INPUT_PULLUP);

  // LED RGB como salida
  pinMode(ledR, OUTPUT);
  pinMode(ledG, OUTPUT);
  pinMode(ledB, OUTPUT);

  apagarLED();
}

void loop() {
  // Lectura de botones (invertida por INPUT_PULLUP)
  int bColor = !digitalRead(btnColor);
  int bSize  = !digitalRead(btnSize);
  int bDrop  = !digitalRead(btnDrop);
  int bReset = !digitalRead(btnReset);

  // Lectura del joystick
  int xVal = analogRead(joyX);
  int yVal = analogRead(joyY);
  xVal = map(xVal, 0, 1023, -100, 100);
  yVal = map(yVal, 0, 1023, -100, 100);

  // Enviar datos a Processing
  Serial.print(xVal); Serial.print(",");
  Serial.print(yVal); Serial.print(",");
  Serial.print(bColor); Serial.print(",");
  Serial.print(bSize); Serial.print(",");
  Serial.print(bDrop); Serial.print(",");
  Serial.println(bReset);

  // Recibir mensajes desde Processing
  if (Serial.available() > 0) {
    String msg = Serial.readStringUntil('\n');
    msg.trim();

    if (msg == "RED") parpadeoRojo();
    else if (msg == "GREEN") parpadeoVerde();
    else if (msg == "BLUE") parpadeoAzul();
    else if (msg == "RESET") apagarLED();
  }

  delay(100);
}

// --- Funciones de LED RGB ---
void parpadeoRojo() {
  analogWrite(ledR, 255);
  analogWrite(ledG, 0);
  analogWrite(ledB, 0);
  delay(100);
  apagarLED();
}

void parpadeoVerde() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 255);
  analogWrite(ledB, 0);
  delay(100);
  apagarLED();
}

void parpadeoAzul() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 0);
  analogWrite(ledB, 255);
  delay(100);
  apagarLED();
}

void apagarLED() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 0);
  analogWrite(ledB, 0);
}
```

###### Codigo de Processing
```js
// --- Comunicación Serial con Arduino ---
import processing.serial.*;
Serial myPort;

// --- Variables del juego ---
ArrayList<Forma> formas;
Forma activa;
boolean cayendo = false;

// Paleta de colores inspirada en Jean Arp ("Plant Hammer")
color[] paleta = {
  #F5D5AE, #EBA89A, #D7C7D0, #9FC7AA, #7AB8BF
};

void setup() {
  size(800, 800);
  smooth();
  colorMode(HSB, 360, 100, 100);
  noStroke();

  // --- Configurar el puerto correcto (CAMBIA si no es COM6) ---
  String portName = "COM6";
  println("Intentando conectar con " + portName);
  myPort = new Serial(this, portName, 9600);
  myPort.bufferUntil('\n');

  // --- Inicialización del juego ---
  formas = new ArrayList<Forma>();
  activa = new Forma(width/2, 100, paleta[0], 40);
}

void draw() {
  background(245, 15, 95);

  // Dibujar todas las formas
  for (Forma f : formas) {
    f.display();
  }

  // Dibujar la forma activa
  activa.display();

  // Movimiento y caída
  if (cayendo) {
    activa.caer();
  }

  // Si la forma activa toca el suelo
  if (activa.y > height - activa.tam/2) {
    formas.add(activa);
    activa = new Forma(width/2, 100, paleta[int(random(paleta.length))], 40);
    cayendo = false;
  }

  // Verificar colisiones entre formas
  verificarColisiones();
}

// --- Lectura de datos desde Arduino ---
void serialEvent(Serial p) {
  String data = trim(p.readStringUntil('\n'));
  if (data == null) return;

  String[] v = split(data, ',');
  if (v.length != 6) return;

  int joyX = int(v[0]);
  int joyY = int(v[1]);
  int bColor = int(v[2]);
  int bSize  = int(v[3]);
  int bDrop  = int(v[4]);
  int bReset = int(v[5]);

  // Movimiento horizontal con el joystick
  activa.x += joyX * 0.05;
  activa.x = constrain(activa.x, activa.tam/2, width - activa.tam/2);

  // --- Controles de botones ---
  if (bColor == 1) {
    activa.cambiarColor();
  }

  if (bSize == 1) {
    activa.cambiarTam();
  }

  if (bDrop == 1 && !cayendo) {
    cayendo = true;
    myPort.write("GREEN\n");  // Indica caída (LED verde)
  }

  if (bReset == 1) {
    reiniciarJuego();
    myPort.write("RESET\n");
  }
}

// --- Verificar colisiones entre formas ---
void verificarColisiones() {
  for (int i = 0; i < formas.size(); i++) {
    Forma f = formas.get(i);
    float d = dist(f.x, f.y, activa.x, activa.y);
    if (d < (f.tam/2 + activa.tam/2)) {
      // Si son del mismo color y tamaño → fusionar
      if (f.col == activa.col && abs(f.tam - activa.tam) < 5) {
        myPort.write("BLUE\n"); // LED azul por fusión
        formas.remove(i);
        activa = new Forma(f.x, f.y, f.col, f.tam * 1.3);
        return;
      } else {
        // Si son diferentes → colapso
        myPort.write("RED\n"); // LED rojo por colapso
        reiniciarJuego();
        return;
      }
    }
  }
}

// --- Reiniciar el juego ---
void reiniciarJuego() {
  formas.clear();
  activa = new Forma(width/2, 100, paleta[int(random(paleta.length))], 40);
  cayendo = false;
  println("Juego reiniciado");
}

// --- Clase de Forma orgánica ---
class Forma {
  float x, y, tam;
  color col;

  Forma(float x, float y, color c, float t) {
    this.x = x;
    this.y = y;
    this.col = c;
    this.tam = t;
  }

  void display() {
    fill(col);
    beginShape();
    for (float a = 0; a < TWO_PI; a += 0.3) {
      float r = tam/2 + 10 * noise(x * 0.01 + cos(a), y * 0.01 + sin(a));
      vertex(x + cos(a) * r, y + sin(a) * r);
    }
    endShape(CLOSE);
  }

  void caer() {
    y += 3;
  }

  void cambiarColor() {
    col = paleta[int(random(paleta.length))];
  }

  void cambiarTam() {
    tam = random(30, 80);
  }
}
```
#### Primeros codigos!
###### Componentes e indicaciones del circuito:

```js
boton 1 (pin 4) - controla color
boton 2 (pin 6) - controla tamaño
boton 3 (pin 8) - controla caida
boton 4 (pin 10) - resetea el juego
```
```js
Joystick X/Y/boton
X - Horizontal
Y - Vertical (no hace nada)
boton - (no hace nada)
```
```js
LED 3 RGB - rojo, verde, azul.
Rojo - Colapso y fin del juego.
Verde - Se deja caer la forma
Azul - Se juntan 2 figuras iguales.
```

#### Probar nuevos codigos en casa.
###### Codigo Arduino
```js
// --- Pines del joystick y botones ---
int joyX = A0;
int joyY = A1;
int joyBtn = 2;

int btn1 = 4; // Color
int btn2 = 5; // Tamaño
int btn3 = 6; // Forma
int btn4 = 7; // Reinicio

// --- LED RGB ---
int ledR = 8;
int ledG = 9;
int ledB = 10;

void setup() {
  Serial.begin(9600);

  pinMode(joyX, INPUT);
  pinMode(joyY, INPUT);
  pinMode(joyBtn, INPUT_PULLUP);

  pinMode(btn1, INPUT_PULLUP);
  pinMode(btn2, INPUT_PULLUP);
  pinMode(btn3, INPUT_PULLUP);
  pinMode(btn4, INPUT_PULLUP);

  pinMode(ledR, OUTPUT);
  pinMode(ledG, OUTPUT);
  pinMode(ledB, OUTPUT);

  apagarLED();
}

void loop() {
  // --- Leer joystick y botones ---
  int x = map(analogRead(joyX), 0, 1023, -100, 100);
  int y = map(analogRead(joyY), 0, 1023, -100, 100);
  int btnJoy = !digitalRead(joyBtn);

  int b1 = !digitalRead(btn1);
  int b2 = !digitalRead(btn2);
  int b3 = !digitalRead(btn3);
  int b4 = !digitalRead(btn4);

  // --- Enviar datos a Processing ---
  Serial.print(x); Serial.print(",");
  Serial.print(y); Serial.print(",");
  Serial.print(b1); Serial.print(",");
  Serial.print(b2); Serial.print(",");
  Serial.print(b3); Serial.print(",");
  Serial.print(b4); Serial.print(",");
  Serial.println(btnJoy);

  // --- Recibir mensajes desde Processing ---
  if (Serial.available() > 0) {
    String msg = Serial.readStringUntil('\n');
    msg.trim();

    if (msg == "RED") parpadeoRojo();
    else if (msg == "BLUE") parpadeoAzul();
    else if (msg == "GREEN_ON") encenderVerde();   // ✅ ahora existe
    else if (msg == "GREEN_OFF") apagarLED();      // ✅ apaga verde
    else if (msg == "RESET") apagarLED();
  }

  delay(100);
}

// --- Funciones LED RGB ---
void parpadeoRojo() {
  analogWrite(ledR, 255);
  analogWrite(ledG, 0);
  analogWrite(ledB, 0);
  delay(150);
  apagarLED();
}

void parpadeoAzul() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 0);
  analogWrite(ledB, 255);
  delay(150);
  apagarLED();
}

void encenderVerde() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 255);
  analogWrite(ledB, 0);
}

void apagarLED() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 0);
  analogWrite(ledB, 0);
}
```
###### Codigo Processing
```js
import processing.serial.*;
Serial myPort;

// --- Control y formas ---
ArrayList<Forma> formas;
Forma activa;
boolean cayendo = false;
int formaTipo = 0;

// --- Comunicación con Arduino ---
int joyX, joyY;
boolean joyBtn;
boolean[] botones = new boolean[4];

color[] paleta = {
  #F5D5AE, #EBA89A, #D7C7D0, #9FC7AA, #7A8B8F
};

void setup() {
  size(800, 800);
  colorMode(HSB, 360, 100, 100);
  noStroke();

  // --- Puerto Serial ---
  String portName = "COM6";  // 👈 ajusta según tu puerto real
  myPort = new Serial(this, portName, 9600);
  myPort.bufferUntil('\n');

  formas = new ArrayList<Forma>();
  activa = new Forma(width/2, 100, paleta[0], formaTipo);
}

void draw() {
  background(245, 15, 95);

  actualizarControles();
  for (Forma f : formas) f.display();
  activa.display();

  if (cayendo) {
    activa.caer();
    myPort.write("GREEN_ON\n"); // LED verde encendido mientras cae
    if (activa.y > height - 50) { // colisión básica
      cayendo = false;
      formas.add(activa);
      activa = new Forma(width/2, 100, paleta[int(random(paleta.length))], formaTipo);
      myPort.write("GREEN_OFF\n");
      myPort.write("BLUE\n"); // LED azul al caer
    }
  } else {
    myPort.write("GREEN_OFF\n");
  }
}

// --- Leer datos del Arduino ---
void serialEvent(Serial p) {
  String inString = p.readStringUntil('\n');
  if (inString == null) return;

  inString = trim(inString);
  String[] parts = split(inString, ',');
  if (parts.length < 7) return;

  joyX = int(parts[0]);
  joyY = int(parts[1]);
  botones[0] = parts[2].equals("1"); // btnColor
  botones[1] = parts[3].equals("1"); // btnSize
  botones[2] = parts[4].equals("1"); // btnForma
  botones[3] = parts[5].equals("1"); // btnReset
  joyBtn     = parts[6].equals("1"); // botón del joystick
}

// --- Lógica de control ---
void actualizarControles() {
  if (botones[0]) { // cambia color
    activa.cambiarColor(paleta[int(random(paleta.length))]);
    myPort.write("RED\n");
    delay(200);
  }

  if (botones[1]) { // cambia tamaño
    activa.cambiarTamano(random(40, 120));
    delay(200);
  }

  if (botones[2]) { // cambia forma (Jean Arp)
    formaTipo = (formaTipo + 1) % 3;
    activa.cambiarForma(formaTipo);
    myPort.write("RED\n"); // rojo = cambio de forma
    delay(200);
  }

  if (joyBtn && !cayendo) { // botón del joystick inicia caída
    cayendo = true;
  }

  if (botones[3]) { // reset
    formas.clear();
    activa = new Forma(width/2, 100, paleta[0], formaTipo);
    myPort.write("RESET\n");
    delay(200);
  }
}

// --- Clase de Forma (inspirada en Jean Arp) ---
class Forma {
  float x, y, tam;
  color c;
  int tipo;

  Forma(float x_, float y_, color c_, int tipo_) {
    x = x_;
    y = y_;
    c = c_;
    tam = 80;
    tipo = tipo_;
  }

  void display() {
    fill(c);
    pushMatrix();
    translate(x, y);

    switch (tipo) {
      case 0:
        ellipse(0, 0, tam, tam * 0.9);
        break;
      case 1:
        beginShape();
        for (float a = 0; a < TWO_PI; a += 0.2) {
          float r = tam * 0.4 + noise(a, frameCount * 0.02) * tam * 0.3;
          vertex(cos(a) * r, sin(a) * r);
        }
        endShape(CLOSE);
        break;
      case 2:
        beginShape();
        for (float a = 0; a < TWO_PI; a += 0.25) {
          float r = tam * 0.3 + sin(a * 3 + frameCount * 0.02) * tam * 0.2;
          vertex(cos(a) * r, sin(a) * r);
        }
        endShape(CLOSE);
        break;
    }
    popMatrix();
  }

  void caer() {
    y += 5;
  }

  void cambiarColor(color nuevo) {
    c = nuevo;
  }

  void cambiarTamano(float nuevoTam) {
    tam = nuevoTam;
  }

  void cambiarForma(int nuevoTipo) {
    tipo = nuevoTipo;
  }
}
```

###### Componentes e indicaciones del circuito:

```js
boton 1 (pin 4) - controla color
boton 2 (pin 6) - controla tamaño
boton 3 (pin 8) - controla la forma aleatoriamente
boton 4 (pin 10) - resetea el juego
```
```js
Joystick X/Y/boton
X - Horizontal
Y - Vertical (no hace nada)
boton - Deja caer la forma
```
```js
LED 3 RGB - rojo, verde, azul.
Rojo - Colapso y fin del juego.
Verde - Se deja caer la forma, se mantiene brillando hasta caer y colicionar con otra.
Azul - Se juntan 2 figuras iguales.
```
