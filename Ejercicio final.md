# Presentacion proyecto final
##### Caos Organico: Jean Arp y Suika Game

#### Circuito a utilizar: 
<img src="https://raw.githubusercontent.com/DannyCatalan/Interfaz-II/refs/heads/main/img/circuito%20final.jpg"/>

###### Componentes:
```js
1. Placa arduino uno
2. Protoboar
3. Botones
4. Cables
4. LED RGB catodo comun
5. Modulo joystick XY
```

###### Codigo de Arduino
```js
// --- Pines de conexión ---
const int btnColor = 4;
const int btnSize  = 6;
const int btnShape = 10; 
const int btnReset = 12; 

const int joyX = A0;
const int joyY = A1;
const int joyBtn = 2;  // botón del joystick

// LED RGB (cátodo común)
const int ledR = 7; 
const int ledG = 8; 
const int ledB = 9; 

// Estado de caída
bool cayendo = false;

void setup() {
  Serial.begin(9600);

  // Botones con resistencia pull-up interna
  pinMode(btnColor, INPUT_PULLUP);
  pinMode(btnSize,  INPUT_PULLUP);
  pinMode(btnShape, INPUT_PULLUP);
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
  int bShape = !digitalRead(btnShape);
  int bReset = !digitalRead(btnReset);
  int bDrop  = !digitalRead(joyBtn);

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
  Serial.print(bShape); Serial.print(",");
  Serial.print(bDrop); Serial.print(",");
  Serial.println(bReset);

  // Si se presiona el botón del joystick, activar caída
  if (bDrop) {
    cayendo = true;
    brilloVerde();  // LED verde constante mientras cae
  }

  // Recibir mensajes desde Processing
  if (Serial.available() > 0) {
    String msg = Serial.readStringUntil('\n');
    msg.trim();

    if (msg == "COLAPSO") parpadeoRojo();       // pantalla llena
    else if (msg == "MEZCLA") parpadeoAzul();   // figuras mezcladas
    else if (msg == "SUELO") {                  // figura llegó al suelo
      cayendo = false;
      apagarLED();
    }
    else if (msg == "RESET") {
      cayendo = false;
      apagarLED();
    }
  }

  delay(50);
}

// --- Funciones de LED RGB ---
void parpadeoRojo() {
  for (int i = 0; i < 3; i++) {
    analogWrite(ledR, 255);
    delay(150);
    apagarLED();
    delay(150);
  }
}

void brilloVerde() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 255);
  analogWrite(ledB, 0);
}

void parpadeoAzul() {
  for (int i = 0; i < 3; i++) {
    analogWrite(ledR, 0);
    analogWrite(ledG, 0);
    analogWrite(ledB, 255);
    delay(150);
    apagarLED();
    delay(150);
  }
}

void apagarLED() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 0);
  analogWrite(ledB, 0);
}
```

###### Codigo de Processing
```js
import processing.serial.*;
import java.util.ArrayList;

// ----- Imágenes -----
PImage fondo;
PImage[] figurasNivel = new PImage[8];

// ----- Serial -----
Serial puerto;
boolean puertoOk = false;

// ----- Juego -----
ArrayList<Figura> figuras = new ArrayList<Figura>();
float gravedad = 0.55;
float rozamiento = 0.98;
int maxObjetos = 36;
int baseSize = 120;
float sueloY;

// ----- Entrada desde Arduino -----
int xJoy = 0, yJoy = 0;
int bColor = 0, bSize = 0, bShape = 0, bDrop = 0, bReset = 0;

// ----- Estados configurables -----
boolean cayendo = false;
int nivelInicial = 1;       // cambia con btnShape
float escalaActual = 1.0;   // cambia con btnSize (0.85 / 1.0 / 1.25)
int modoColor = 0;          // 0 sin tint, 1 cálido sutil, 2 cálido vibrante, 3 frío sutil, 4 frío vibrante

// Control horizontal suavizado por joystick
float controlX = 0;

// ----- Paletas (abstracción orgánica) -----
// Cada color: R,G,B (sin alpha). Se elige por nivel para variedad orgánica.
int[][] paletaCalidoSutil = {
  {216, 180, 162}, // terracota suave
  {236, 210, 190}, // melocotón pálido
  {212, 196, 164}, // beige cálido
  {198, 174, 134}  // ocre suave
};

int[][] paletaCalidoVibrante = {
  {230, 85, 40},   // naranja rojo
  {220, 40, 120},  // magenta cálido
  {245, 190, 20},  // dorado intenso
  {10, 150, 130}   // turquesa acento (contraste orgánico)
};

int[][] paletaFrioSutil = {
  {168, 185, 206}, // azul pizarra suave
  {180, 210, 195}, // verde mar pálido
  {210, 210, 220}, // gris niebla
  {185, 170, 205}  // lavanda grisácea
};

int[][] paletaFrioVibrante = {
  {30, 190, 220},  // cian fuerte
  {40, 120, 240},  // azul eléctrico
  {140, 230, 60},  // lima vibrante
  {155, 60, 220}   // violeta intenso
};

void setup() {
  size(900, 700);
  imageMode(CENTER);
  noStroke();
  colorMode(RGB, 255);

  // Cargar imágenes desde carpeta "data"
  fondo = loadImage("fondo.jpg");
  if (fondo != null) fondo.resize(width, height);

  for (int i = 0; i < 8; i++) {
    figurasNivel[i] = loadImage("figura" + (i+1) + ".png");
    if (figurasNivel[i] == null) println("FALTA: figura" + (i+1) + ".png");
  }

  // Serial
  String[] ports = Serial.list();
  if (ports.length == 0) {
    println("ADVERTENCIA: No hay puertos seriales.");
  } else {
    // Ajusta el índice si tu Arduino no es [0]
    puerto = new Serial(this, ports[0], 9600);
    puerto.bufferUntil('\n');
    puertoOk = true;
  }

  sueloY = height - 50; // línea de suelo
}

void draw() {
  if (fondo != null) image(fondo, width/2, height/2);
  else background(240);

  // Control horizontal por joystick (suavizado)
  controlX = lerp(controlX, map(xJoy, -100, 100, -7, 7), 0.18);

  // Actualizar y dibujar figuras
  for (int i = figuras.size()-1; i >= 0; i--) {
    Figura f = figuras.get(i);
    f.actualizar();
    f.mostrar();
  }

  // Colisiones + fusión
  verificarColisionesYFusion();

  // Colapso si excede el umbral
  if (figuras.size() > maxObjetos) enviar("COLAPSO");

  dibujarHUD();
}

// ----- HUD -----
void dibujarHUD() {
  fill(0, 120);
  rectMode(CORNER);
  rect(10, 10, 300, 110, 8);
  fill(255);
  textAlign(LEFT, TOP);
  text("Figuras: " + figuras.size(), 20, 18);
  text("Nivel inicial: " + nivelInicial, 20, 36);
  text("Escala: " + nf(escalaActual, 1, 2), 20, 54);
  String col =
    modoColor == 0 ? "Sin tint" :
    modoColor == 1 ? "Cálido sutil" :
    modoColor == 2 ? "Cálido vibrante" :
    modoColor == 3 ? "Frío sutil" :
    "Frío vibrante";
  text("Paleta: " + col, 20, 72);
  stroke(255, 180);
  line(0, sueloY, width, sueloY);
  noStroke();
}

// ----- Serial: x,y,bColor,bSize,bShape,bDrop,bReset -----
void serialEvent(Serial p) {
  String data = p.readStringUntil('\n');
  if (data == null) return;
  data = trim(data);
  String[] v = split(data, ',');
  if (v.length != 7) return;

  try {
    xJoy = Integer.parseInt(v[0]);
    yJoy = Integer.parseInt(v[1]);
    bColor = Integer.parseInt(v[2]);
    bSize  = Integer.parseInt(v[3]);
    bShape = Integer.parseInt(v[4]);
    bDrop  = Integer.parseInt(v[5]);
    bReset = Integer.parseInt(v[6]);
  } catch (Exception e) { return; }

  // Caída: crea nueva figura
  if (bDrop == 1 && !cayendo) {
    cayendo = true;
    Figura f = new Figura(width/2, -baseSize, nivelInicial, escalaActual);
    f.vx = controlX * 0.5; // impulso inicial horizontal
    figuras.add(f);
    // "SUELO" se envía al tocar suelo
  }

  // Reset
  if (bReset == 1) {
    figuras.clear();
    cayendo = false;
    enviar("RESET");
  }

  // Ciclo de paletas
  if (bColor == 1) {
    modoColor = (modoColor + 1) % 5; // 0..4
  }

  // Escalas: 0.85 -> 1.0 -> 1.25 -> 0.85
  if (bSize == 1) {
    if (escalaActual < 0.9) escalaActual = 1.0;
    else if (escalaActual < 1.1) escalaActual = 1.25;
    else escalaActual = 0.85;
  }

  // Cambiar nivel inicial (1..8)
  if (bShape == 1) {
    nivelInicial = (nivelInicial % 8) + 1;
  }
}

// ----- Fusión Suika -----
void verificarColisionesYFusion() {
  for (int i = figuras.size()-1; i >= 0; i--) {
    Figura a = figuras.get(i);
    for (int j = i-1; j >= 0; j--) {
      Figura b = figuras.get(j);
      if (!a.activa || !b.activa) continue;

      float dx = a.x - b.x;
      float dy = a.y - b.y;
      float d2 = dx*dx + dy*dy;
      float minR = a.rCol + b.rCol;

      if (d2 < minR*minR) {
        // Separación mínima para evitar solape
        float d = max(0.0001, sqrt(d2));
        float nx = dx / d;
        float ny = dy / d;
        float empuje = (minR - d) * 0.5;
        a.x += nx * empuje;
        a.y += ny * empuje;
        b.x -= nx * empuje;
        b.y -= ny * empuje;

        // Fusión si niveles iguales
        if (a.nivel == b.nivel) {
          int nuevo = min(8, a.nivel + 1);
          Figura nueva = new Figura((a.x + b.x) * 0.5, (a.y + b.y) * 0.5, nuevo, 1.0);
          figuras.add(nueva);
          a.activa = false;
          b.activa = false;
          figuras.remove(i);
          figuras.remove(j);
          enviar("MEZCLA");
          break;
        }
      }
    }
  }
}

// ----- Envío a Arduino -----
void enviar(String msg) {
  if (puertoOk && puerto != null) puerto.write(msg + "\n");
}

// ----- Clase Figura -----
class Figura {
  float x, y, vx, vy;
  int nivel;
  float escala;
  float rCol;
  boolean activa = true;

  Figura(float x, float y, int nivel, float escala) {
    this.x = x;
    this.y = y;
    this.vx = 0;
    this.vy = 0;
    this.nivel = nivel;
    this.escala = escala;
    this.rCol = (baseSize * escala * 0.46); // radio aproximado para colisión
  }

  void actualizar() {
    // Control horizontal por joystick mientras cae
    vx += controlX * 0.06;

    // Física
    vy += gravedad;
    vx *= rozamiento;
    x += vx;
    y += vy;

    // Paredes
    float margen = baseSize * 0.5 * escala;
    if (x < margen) { x = margen; vx *= -0.35; }
    if (x > width - margen) { x = width - margen; vx *= -0.35; }

    // Suelo
    float h = baseSize * escala;
    if (y > sueloY - h*0.5) {
      y = sueloY - h*0.5;
      vy *= -0.24;
      if (abs(vy) < 0.45) vy = 0;
      enviar("SUELO");
      cayendo = false;
    }
  }

  void mostrar() {
    aplicarPaleta(nivel);
    PImage img = figurasNivel[nivel-1];
    float w = baseSize * escala;
    float h = baseSize * escala;

    if (img != null) {
      image(img, x, y, w, h);
    } else {
      // Si falta imagen, no usamos círculos; mostramos un rectángulo de alerta discreto
      noTint();
      fill(255, 80, 80, 200);
      rectMode(CENTER);
      rect(x, y, w*0.9, h*0.9, 12);
    }
    noTint();
  }
}

// ----- Tint según paleta y nivel -----
void aplicarPaleta(int nivel) {
  if (modoColor == 0) { // sin tint
    tint(255);
    return;
  }

  int idx = (nivel - 1) % 4;
  int r=255, g=255, b=255;

  if (modoColor == 1) { // cálido sutil
    r = paletaCalidoSutil[idx][0];
    g = paletaCalidoSutil[idx][1];
    b = paletaCalidoSutil[idx][2];
    tint(r, g, b, 235);
  } else if (modoColor == 2) { // cálido vibrante
    r = paletaCalidoVibrante[idx][0];
    g = paletaCalidoVibrante[idx][1];
    b = paletaCalidoVibrante[idx][2];
    // leve variación orgánica
    tint(constrain(r + random(-10, 10), 0, 255),
         constrain(g + random(-10, 10), 0, 255),
         constrain(b + random(-10, 10), 0, 255), 235);
  } else if (modoColor == 3) { // frío sutil
    r = paletaFrioSutil[idx][0];
    g = paletaFrioSutil[idx][1];
    b = paletaFrioSutil[idx][2];
    tint(r, g, b, 235);
  } else { // frío vibrante
    r = paletaFrioVibrante[idx][0];
    g = paletaFrioVibrante[idx][1];
    b = paletaFrioVibrante[idx][2];
    tint(constrain(r + random(-10, 10), 0, 255),
         constrain(g + random(-10, 10), 0, 255),
         constrain(b + random(-10, 10), 0, 255), 235);
  }
}
```

<img src="https://raw.githubusercontent.com/DannyCatalan/Interfaz-II/refs/heads/main/img/entrega.jpg"/>
<img src="https://raw.githubusercontent.com/DannyCatalan/Interfaz-II/refs/heads/main/img/fondo%20suika-arp.jpg"/>

