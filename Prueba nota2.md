# Presentacion proyecto
##### Caos Organico: Jean Arp y suika game

###### Codigo de Arduini¿o
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
