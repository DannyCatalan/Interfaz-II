# Presentacion proyecto
##### Caos Organico: Jean Arp y suika game

###### Codigo arduino

```js
// --- ArpSuika: Control con Joystick + 4 Botones ---

const int joyX = A0;
const int joyY = A1;
const int joySW = 2;

const int buttons[4] = {4, 6, 8, 10};

void setup() {
  Serial.begin(9600);
  pinMode(joySW, INPUT_PULLUP);
  for (int i = 0; i < 4; i++) {
    pinMode(buttons[i], INPUT_PULLUP);
  }
}

void loop() {
  int xVal = analogRead(joyX);
  int yVal = analogRead(joyY);
  int swVal = digitalRead(joySW);

  Serial.print(xVal); Serial.print(",");
  Serial.print(yVal); Serial.print(",");
  Serial.print(swVal);

  for (int i = 0; i < 4; i++) {
    Serial.print(",");
    Serial.print(digitalRead(buttons[i]));
  }

  Serial.println();
  delay(50);
}
```

###### Codigo Processing

```js
import processing.serial.*;

Serial myPort;

float joyX, joyY;
boolean joyPress;
boolean[] buttons = new boolean[4];

ArrayList<Forma> formas;
Forma activa;
boolean cayendo = false;

// 🎨 Paleta de colores inspirada en "Plant Hammer" de Jean Arp
int[] arpColors = {
  color(230, 100, 95), // Rojo óxido / Marrón rojizo
  color(25, 60, 95),   // Naranja quemado / Tierra
  color(300, 40, 60),  // Púrpura oscuro / Vino
  color(21, 10, 98),   // Beige / Blanco cremoso
  color(0, 0, 100)     // Blanco puro
};

void setup() {
  size(800, 800);
  colorMode(HSB, 360, 100, 100);
  noStroke();
  formas = new ArrayList<Forma>();

  String portName = Serial.list()[0]; // Ajusta según tu puerto
  myPort = new Serial(this, portName, 9600);
  
  // La primera forma activa tendrá un tamaño y color aleatorio de la paleta
  activa = new Forma(width/2, 100, random(30, 50), getRandomArpColor(), int(random(4))); // Iniciamos con un tipo aleatorio
}

void draw() {
  background(210, 20, 95); // Fondo

  leerDatosArduino();

  // Movimiento joystick (horizontal) - SOLO si no está cayendo
  float mov = map(joyX, 0, 1023, -10, 10);
  if (!cayendo) activa.x += mov;

  // Mantener dentro de pantalla
  activa.x = constrain(activa.x, activa.s / 2, width - activa.s / 2);

  // Lógica de los Botones
  if (buttons[0] == true && !cayendo) { 
    cayendo = true; // Botón 1: Inicia la caída
  }

  if (buttons[1] == true) { 
    activa.cambiarColor(); // Botón 2: Cambia el color
    buttons[1] = false; 
  }

  if (buttons[2] == true) { 
    activa.cambiarForma(); // Botón 3: Cambia la forma
    buttons[2] = false; 
  }

  if (buttons[3] == true) { 
    resetJuego(); // Botón 4: Reset
    buttons[3] = false; 
  }

  // Lógica de caída y colisión (Estatismo y Fusión)
  if (cayendo) {
    activa.y += 5;

    boolean colision = false;

    // 1. Colisión con el suelo
    if (activa.y >= height - activa.s / 2) {
      colision = true;
      activa.y = height - activa.s / 2; // Asegura que quede justo en el suelo
    } 
    
    // 2. Colisión con otras formas
    if (!colision) {
      for (int i = 0; i < formas.size(); i++) {
        Forma otra = formas.get(i);
        if (checkCollision(activa, otra)) {
          colision = true;
          // Si colisiona, la detiene justo encima de la forma
          activa.y -= 5; 
          break; // Una vez que colisiona, detenemos la búsqueda
        }
      }
    }

    if (colision) {
      // 💥 Intentar Fusión 
      if (tryMerge(activa)) {
        // La fusión ocurrió, no se añade la forma activa, se crea una nueva inmediatamente
      } else {
        // No hubo fusión, la forma activa se convierte en una forma estática
        formas.add(activa);
      }

      // Prepara la siguiente forma
      activa = new Forma(width/2, 100, random(30, 50), getRandomArpColor(), int(random(4)));
      cayendo = false;
    }
  }

  // 🔄 Procesar Colisiones para Formas Estáticas
  // Esta sección permite que las formas ya colocadas se fusionen si se tocan debido a una fusión previa.
  for(int i = formas.size() - 1; i >= 0; i--) {
      Forma f = formas.get(i);
      tryMerge(f);
  }

  // Dibujar formas existentes
  for (Forma f : formas) f.mostrar();

  // Dibujar forma activa
  activa.mostrar();
}

/**
 * Intenta fusionar una forma dada con cualquier otra forma estática.
 * @return true si ocurrió una fusión.
 */
boolean tryMerge(Forma formaA) {
  boolean merged = false;
  for (int i = formas.size() - 1; i >= 0; i--) {
    Forma formaB = formas.get(i);

    // Evitar auto-fusión y fusión de la forma activa consigo misma si ya fue añadida
    if (formaA == formaB) continue; 
    
    // 🧪 Criterio de Fusión: Mismo TIPO de forma y Mismo COLOR
    if (formaA.tipo == formaB.tipo && formaA.c == formaB.c && checkCollision(formaA, formaB)) {
      
      // 1. Calcular nuevo tamaño y color (fusión)
      float nuevoTamano = formaA.s * 1.2; // Aumentamos el tamaño
      // Suma de colores: Obtenemos el nuevo color promediando (o sumando si fuera el caso)
      int nuevoColor = mergeColors(formaA.c, formaB.c);

      // 2. Determinar la posición de la nueva forma (usamos el centro de la forma más pesada/grande)
      float nuevoX = (formaA.x * formaA.s + formaB.x * formaB.s) / (formaA.s + formaB.s);
      float nuevoY = (formaA.y * formaA.s + formaB.y * formaB.s) / (formaA.s + formaB.s);
      
      // 3. Crear la nueva forma fusionada (será del siguiente tipo)
      Forma nuevaForma = new Forma(nuevoX, nuevoY, nuevoTamano, nuevoColor, (formaA.tipo + 1) % 4);

      // 4. Remover las formas viejas y añadir la nueva
      formas.remove(formaB);
      if (formas.contains(formaA)) {
          formas.remove(formaA);
      }

      formas.add(nuevaForma);
      merged = true;
      
      // Intentamos fusionar la nueva forma creada para una reacción en cadena (estilo Suika Game)
      tryMerge(nuevaForma); 
      break; 
    }
  }
  return merged;
}

/**
 * Detección de Colisión Simple (círculo) usando la distancia entre centros.
 */
boolean checkCollision(Forma a, Forma b) {
  float distanciaCentros = dist(a.x, a.y, b.x, b.y);
  float sumaRadios = (a.s / 2) + (b.s / 2);
  return distanciaCentros < sumaRadios;
}

/**
 * Función que "suma" dos colores HSB, promediando el Hue y tomando la Max. Saturación/Brillo.
 */
int mergeColors(int c1, int c2) {
    float h1 = hue(c1);
    float h2 = hue(c2);
    float s1 = saturation(c1);
    float s2 = saturation(c2);
    float b1 = brightness(c1);
    float b2 = brightness(c2);
    
    // Promedio de Matiz (Hue)
    float newH = (h1 + h2) / 2;
    if (abs(h1 - h2) > 180) { // Manejar el caso de cruzar el 0/360
        newH = (h1 + h2 + 360) / 2 % 360;
    }
    
    // Tomar la Saturación y Brillo más altos para colores más vibrantes
    float newS = max(s1, s2);
    float newB = max(b1, b2);
    
    return color(newH, newS, newB);
}


// --- Resto del código (leerDatosArduino, resetJuego, getRandomArpColor) ---

void leerDatosArduino() {
  if (myPort.available() > 0) {
    String data = myPort.readStringUntil('\n');
    if (data != null) {
      String[] values = trim(split(data, ','));
      if (values.length == 7) {
        joyX = float(values[0]);
        joyY = float(values[1]);
        joyPress = values[2].equals("0");
        for (int i = 0; i < 4; i++) {
          buttons[i] = values[i + 3].equals("0");
        }
      }
    }
  }
}

void resetJuego() {
  formas.clear();
  activa = new Forma(width/2, 100, random(30, 50), getRandomArpColor(), int(random(4)));
  cayendo = false;
}

int getRandomArpColor() {
  return arpColors[int(random(arpColors.length))];
}

// ---- Clase Forma (Incluye el 'tipo' para la fusión) ----
class Forma {
  float x, y, s;
  int c;
  int tipo; // 0: Blob Orgánico 1, 1: Blob Orgánico 2, 2: "Riñón", 3: Forma Angular 

  Forma(float x_, float y_, float s_, int c_, int tipo_) {
    x = x_;
    y = y_;
    s = s_;
    c = c_;
    tipo = tipo_; // Ahora el tipo se asigna al crear la forma
  }
  
  // Constructor para la forma activa
  Forma(float x_, float y_, float s_, int c_) {
    this(x_, y_, s_, c_, int(random(4))); // Usa el constructor completo con tipo aleatorio
  }


  void mostrar() {
    fill(c);
    pushMatrix();
    translate(x, y);
    if (tipo == 0) blob1();
    else if (tipo == 1) blob2();
    else if (tipo == 2) rinon();
    else formaAngular();
    popMatrix();
  }

  void cambiarColor() {
    c = getRandomArpColor();
  }

  void cambiarForma() {
    tipo = (tipo + 1) % 4;
  }

  void blob1() {
    beginShape();
    float offset = random(100);
    for (float a = 0; a < TWO_PI; a += 0.2) { 
      float r = s/2 + map(noise(cos(a) * 0.8 + offset, sin(a) * 0.8 + offset), 0, 1, -s/4, s/4); 
      vertex(cos(a) * r, sin(a) * r);
    }
    endShape(CLOSE);
  }
  
  void blob2() {
    beginShape();
    float offset = random(100);
    for (float a = 0; a < TWO_PI; a += 0.2) {
      float r = s/2 + map(noise(cos(a) * 0.6 + offset, sin(a) * 0.6 + offset), 0, 1, -s/3, s/3); 
      vertex(cos(a) * r * 1.2, sin(a) * r * 0.8);
    }
    endShape(CLOSE);
  }

  void rinon() {
    beginShape();
    float halfS = s / 2;
    curveVertex(0, halfS * 0.8); 
    curveVertex(0, halfS * 0.8);
    curveVertex(halfS * 0.6, halfS * 0.6);
    curveVertex(halfS * 0.8, 0);
    curveVertex(halfS * 0.6, -halfS * 0.6);
    curveVertex(0, -halfS * 0.8);
    curveVertex(-halfS * 0.6, -halfS * 0.6);
    curveVertex(-halfS * 0.8, 0);
    curveVertex(-halfS * 0.6, halfS * 0.6);
    curveVertex(0, halfS * 0.8); 
    endShape(CLOSE);
  }

  void formaAngular() {
    beginShape();
    float scale = s / 2;
    vertex(-scale * 0.8, -scale * 0.8);
    vertex(scale * 0.6, -scale * 0.8);
    vertex(scale * 0.6, -scale * 0.2);
    vertex(scale * 0.8, -scale * 0.2);
    vertex(scale * 0.8, scale * 0.8);
    vertex(-scale * 0.2, scale * 0.8);
    vertex(-scale * 0.2, scale * 0.6);
    vertex(-scale * 0.8, scale * 0.6);
    endShape(CLOSE);
  }
}
```

#### Nuevas pruebas 

###### Codigo de Arduini¿o
```js
// --- Pines de conexión ---
const int btnColor = 6;
const int btnSize  = 8;
const int btnDrop  = 10;
const int btnReset = 4;

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
