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

#### Nuevos codigos de prueba. 

###### Codigo Arduino 
```js
// --- Pines de conexión ---
// Botón 1: Controla el color de la forma
const int btnColor = 4;
// Botón 2: Controla el tamaño de la forma (mantengo la función, aunque en Suika va por nivel)
const int btnSize  = 6;
// Botón de Caída (Botón extra o botón del joystick, si se usa)
const int btnDrop  = 8; 
// Botón 3: Controla el cambio de las formas (nivel/tipo)
const int btnShape = 10; 
// Botón 4: Reinicia el juego
const int btnReset = 12; 

// Joystick
const int joyX = A0;
const int joyY = A1;

// LED RGB (de cátodo común)
const int ledR = 7;
const int ledG = 8;
const int ledB = 9;

void setup() {
  Serial.begin(9600);

  // Botones con resistencia pull-up interna
  pinMode(btnColor, INPUT_PULLUP);
  pinMode(btnSize,  INPUT_PULLUP);
  pinMode(btnDrop,  INPUT_PULLUP);
  pinMode(btnShape, INPUT_PULLUP); // Pin 10 para cambio de forma
  pinMode(btnReset, INPUT_PULLUP); // Pin 12 para reinicio

  // LED RGB como salida (salida analógica para control de brillo)
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
  int bShape = !digitalRead(btnShape); // Estado del botón de cambio de forma
  int bReset = !digitalRead(btnReset); // Estado del botón de reinicio

  // Lectura y mapeo del joystick
  int xVal = analogRead(joyX);
  int yVal = analogRead(joyY);
  xVal = map(xVal, 0, 1023, -100, 100); // Rango de -100 a 100 para un control fino en Processing
  yVal = map(yVal, 0, 1023, -100, 100);

  // Enviar datos a Processing
  // Formato: X, Y, ColorBtn, SizeBtn, DropBtn, ShapeBtn, ResetBtn
  Serial.print(xVal); Serial.print(",");
  Serial.print(yVal); Serial.print(",");
  Serial.print(bColor); Serial.print(",");
  Serial.print(bSize); Serial.print(",");
  Serial.print(bDrop); Serial.print(",");
  Serial.print(bShape); Serial.print(","); // Botón 3: Cambio de Forma
  Serial.println(bReset);                 // Botón 4: Reinicio

  // Recibir mensajes desde Processing para controlar el LED RGB
  if (Serial.available() > 0) {
    String msg = Serial.readStringUntil('\n');
    msg.trim();

    if (msg == "RED") parpadeoRojo();         // Colapso/Game Over
    else if (msg == "BLUE") parpadeoAzul();   // Fusión
    else if (msg == "GREEN_ON") encenderVerdeConstante(); // Forma cayendo
    else if (msg == "GREEN_OFF") apagarLED(); // Forma se detiene o reinicio
  }

  delay(50); // Reducir un poco el delay para una mejor respuesta
}

// --- Funciones de LED RGB ---

// Parpadeo Rojo: Para Game Over/Colapso
void parpadeoRojo() {
  analogWrite(ledR, 255);
  analogWrite(ledG, 0);
  analogWrite(ledB, 0);
  delay(100);
  apagarLED();
}

// Encendido Verde Constante: Para indicar que la forma está cayendo (brillo constante)
void encenderVerdeConstante() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 200); // Brillo constante, se apaga con GREEN_OFF
  analogWrite(ledB, 0);
}

// Parpadeo Azul: Para indicar Fusión/Mezcla
void parpadeoAzul() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 0);
  analogWrite(ledB, 255);
  delay(100);
  apagarLED();
}

// Apagar LED: Para el estado de reposo, o después de un parpadeo
void apagarLED() {
  analogWrite(ledR, 0);
  analogWrite(ledG, 0);
  analogWrite(ledB, 0);
}
```

###### Codigo Processing
```js
// --- Comunicación Serial con Arduino ---
import processing.serial.*;
Serial myPort;

// --- Variables del juego ---
PImage background_img;
ArrayList<Forma> formas;
Forma activa;
boolean cayendo = false;
int maxFormaTipo = 5; // Total de niveles de forma (0 a 5)
int score = 0;
int fusionCount = 0;

// Paleta de colores inspirada en Jean Arp ("Plant Hammer")
// Colores HSB, puedes ajustarlos para que coincidan con la obra:
color[] paleta = {
  color(30, 10, 95),   // Beige (F5D5AE)
  color(10, 40, 90),   // Coral suave (EBA89A)
  color(300, 15, 85),  // Lavanda (D7C7D0)
  color(130, 30, 75),  // Verde Menta (9FC7AA)
  color(190, 40, 75),  // Azul Cian (7AB8BF)
};

// --- Área de juego (Caja/Soporte/Matriz/Guía) ---
// Dimensiones estimadas del cuadrado transparente de la imagen (800x800)
final int GAME_WIDTH = 500;
final int GAME_HEIGHT = 650;
final int GAME_X = (800 - GAME_WIDTH) / 2; // Centrado en X
final int GAME_Y = 120; // Un poco por debajo del tope
final int GROUND_Y = GAME_Y + GAME_HEIGHT;
final int SPAWN_Y = GAME_Y + 50;
final int MAX_ACTIVE_HEIGHT = GAME_Y + 100; // Límite de Game Over

void setup() {
  size(800, 800);
  smooth();
  colorMode(HSB, 360, 100, 100);
  noStroke();

  // 1. Cargar imagen de fondo
  try {
    // La imagen debe estar en la carpeta del sketch.
    background_img = loadImage("fondo suika-arp.jpg");
    if (background_img == null) {
      println("ADVERTENCIA: No se pudo cargar la imagen 'fondo suika-arp.jpg'.");
    }
  } catch (Exception e) {
    println("ERROR al cargar la imagen: " + e.getMessage());
  }


  // 2. Configurar el puerto serial
  String[] portList = Serial.list();
  if (portList.length > 0) {
    // Intenta usar el primer puerto disponible o ajusta a tu puerto específico (ej: "COM5")
    String portName = portList[0]; 
    println("Intentando conectar con " + portName + "...");
    try {
      myPort = new Serial(this, portName, 9600);
      myPort.bufferUntil('\n');
    } catch (Exception e) {
      println("ERROR: No se pudo conectar al puerto serial. El juego continuará, pero el LED no funcionará.");
    }
  } else {
    println("ERROR: No se encontraron puertos seriales disponibles. El LED no funcionará.");
  }


  // 3. Inicialización del juego
  formas = new ArrayList<Forma>();
  activa = crearNuevaForma(GAME_X + GAME_WIDTH/2, SPAWN_Y);
}

void draw() {
  // 1. Dibujar Fondo (Imagen o color con guía)
  if (background_img != null) {
    image(background_img, 0, 0, width, height);
  } else {
    background(240); // Fondo gris claro si la imagen falla
    // Simular el área de juego
    fill(0, 0, 100, 50); // Blanco transparente
    rect(GAME_X, GAME_Y, GAME_WIDTH, GAME_HEIGHT);
  }

  // 2. Línea de Game Over
  stroke(0, 80, 80, 80); // Rojo semitransparente
  strokeWeight(3);
  line(GAME_X, MAX_ACTIVE_HEIGHT, GAME_X + GAME_WIDTH, MAX_ACTIVE_HEIGHT);
  noStroke();
  
  // Mostrar puntaje
  fill(0);
  textSize(20);
  textAlign(LEFT, TOP);
  text("Score: " + score + "\nFusions: " + fusionCount, 20, 20);


  // 3. Movimiento y lógica de caída
  if (cayendo) {
    activa.caer();
    // Enviar señal de LED verde constante
    if (myPort != null) myPort.write("GREEN_ON\n");
  }


  // 4. Verificar si la forma activa ha tocado el suelo o colisionado
  if (activa.y + activa.tam/2 >= GROUND_Y) {
    // Toca el suelo
    establecerForma(activa, GROUND_Y - activa.tam/2);
    activa = crearNuevaForma(GAME_X + GAME_WIDTH/2, SPAWN_Y);
  } else if (cayendo) {
    // Verifica colisiones con formas estáticas
    for (int i = formas.size() - 1; i >= 0; i--) {
      Forma f = formas.get(i);
      if (f.colisionaCon(activa)) {
        fusionarOEstablecer(f, i);
        break; // Solo interactúa con una forma por frame
      }
    }
  }

  // 5. Dibujar todas las formas estáticas
  for (Forma f : formas) {
    f.display(false); 
  }

  // 6. Dibujar la forma activa
  activa.display(true); 

  // 7. Constreñir la forma activa al área de juego (horizontalmente)
  activa.x = constrain(activa.x, GAME_X + activa.tam/2, GAME_X + GAME_WIDTH - activa.tam/2);
}

// --- Funciones de Juego ---

Forma crearNuevaForma(float x, float y) {
  // Las formas iniciales son aleatorias pero pequeñas (tipos 0, 1, o 2)
  int initialTipo = int(random(3)); 
  return new Forma(x, y, paleta[int(random(paleta.length))], initialTipo);
}

void establecerForma(Forma f, float finalY) {
    f.y = finalY;
    formas.add(f);
    cayendo = false;
    if (myPort != null) myPort.write("GREEN_OFF\n"); // Apagar LED verde
    verificarGameOver();
}

void fusionarOEstablecer(Forma formaEstatica, int indexEstatica) {
  // 1. Colisión sin caída. Ajustamos la posición para que toque a la forma estática.
  float distanciaCentros = dist(activa.x, activa.y, formaEstatica.x, formaEstatica.y);
  float ajusteY = (activa.tam/2 + formaEstatica.tam/2) - distanciaCentros;
  float finalY = activa.y - ajusteY;
  
  // 2. Lógica de fusión (Suika Game)
  if (formaEstatica.col == activa.col && activa.tipoForma == formaEstatica.tipoForma) {
    // Fusión exitosa (mismo color y tipo/nivel)
    if (myPort != null) myPort.write("BLUE\n"); // LED azul por fusión
    
    // Calcular nueva forma fusionada
    float newX = (activa.x + formaEstatica.x) / 2;
    float newY = (activa.y + formaEstatica.y) / 2;
    int newTipo = min(activa.tipoForma + 1, maxFormaTipo); // Siguiente nivel
    
    // Eliminar las dos formas viejas
    formas.remove(indexEstatica); // Eliminar la estática
    
    // Crear la nueva forma fusionada y agregarla al array de estáticas
    Forma nuevaForma = new Forma(newX, newY, activa.col, newTipo);
    formas.add(nuevaForma);
    
    // Actualizar puntaje y contador
    score += (newTipo + 1) * 10;
    fusionCount++;
    
    // Crear una nueva forma activa y detener la caída
    activa = crearNuevaForma(GAME_X + GAME_WIDTH/2, SPAWN_Y);
    cayendo = false;
    if (myPort != null) myPort.write("GREEN_OFF\n");
    verificarGameOver();
    
  } else {
    // Colisión de formas diferentes (Se establece la forma activa en el punto de contacto)
    establecerForma(activa, finalY);
    activa = crearNuevaForma(GAME_X + GAME_WIDTH/2, SPAWN_Y);
  }
}

void verificarGameOver() {
  // Game Over si alguna forma establecida toca o cruza la línea de límite
  for (Forma f : formas) {
    if (f.y - f.tam/2 < MAX_ACTIVE_HEIGHT) {
      println("GAME OVER: Colapso del Canvas!");
      if (myPort != null) myPort.write("RED\n"); // LED rojo para colapso
      // El juego se detiene hasta que el usuario presione el botón de reinicio
      return; 
    }
  }
}


// --- Lectura de datos desde Arduino ---
void serialEvent(Serial p) {
  String data = trim(p.readStringUntil('\n'));
  if (data == null || data.isEmpty()) return;

  String[] v = split(data, ',');
  // Formato esperado: X, Y, Color, Size, Drop, Shape, Reset (7 valores)
  if (v.length != 7) return; 

  int joyX = int(v[0]);
  // int joyY = int(v[1]); // No usado en este juego
  int bColor = int(v[2]);
  int bSize  = int(v[3]);
  int bDrop  = int(v[4]);
  int bShape = int(v[5]); 
  int bReset = int(v[6]);

  // Movimiento horizontal con el joystick
  activa.x += joyX * 0.1; 
  // La función draw() se encarga de constreñir la posición

  // --- Controles de botones (usamos flancos de subida con un simple flag estático) ---
  static boolean prevBColor = false;
  static boolean prevBSize = false;
  static boolean prevBShape = false;
  
  if (bColor == 1 && !prevBColor) {
    activa.cambiarColor();
  }
  prevBColor = (bColor == 1);

  if (bSize == 1 && !prevBSize) {
    // Permite al usuario tener cierta variación de tamaño dentro del nivel de forma actual
    activa.cambiarTam(); 
  }
  prevBSize = (bSize == 1);

  if (bShape == 1 && !prevBShape) {
    activa.cambiarForma(); // Botón 3: Cambia el tipo de forma de la activa (nivel)
  }
  prevBShape = (bShape == 1);
  
  if (bDrop == 1 && !cayendo) {
    cayendo = true;
    // GREEN_ON se envía en draw() para mantener el brillo constante
  }

  if (bReset == 1) {
    reiniciarJuego();
    if (myPort != null) myPort.write("GREEN_OFF\n"); 
  }
}

// --- Reiniciar el juego ---
void reiniciarJuego() {
  formas.clear();
  activa = crearNuevaForma(GAME_X + GAME_WIDTH/2, SPAWN_Y);
  cayendo = false;
  score = 0;
  fusionCount = 0;
  println("Juego reiniciado");
}


// --- Clase de Forma orgánica ---
class Forma {
  float x, y, tam;
  color col;
  int tipoForma; // Nivel de la forma (0 al 5)
  float noiseOffset;

  Forma(float x, float y, color c, int tipo) {
    this.x = x;
    this.y = y;
    this.col = c;
    this.tipoForma = tipo;
    this.tam = mapaTamano(tipo); // Tamaño dependiente del tipo
    this.noiseOffset = random(100); // Para variedad estética
  }
  
  // Mapea el tipo de forma a un tamaño base (como en Suika)
  float mapaTamano(int tipo) {
    switch (tipo) {
      case 0: return 40;
      case 1: return 55;
      case 2: return 70;
      case 3: return 85;
      case 4: return 100;
      case 5: return 120;
      default: return 40;
    }
  }


  void display(boolean esActiva) {
    // La forma activa es un poco más transparente y vibrante
    if (esActiva && !cayendo) {
       fill(hue(col), saturation(col), brightness(col), 85); 
    } else {
       fill(col);
    }
    
    // Dibujo de la forma basado en el tipo (Nivel Suika)
    switch (tipoForma) {
      case 0: // Círculo
        ellipse(x, y, tam, tam);
        break;
      case 1: // Elipse
        ellipse(x, y, tam * 0.9, tam * 1.2);
        break;
      case 2: // Arp Suave (Nivel 3)
        dibujarFormaOrganica(x, y, tam, 10, noiseOffset);
        break;
      case 3: // Arp Riñón (Nivel 4)
        dibujarFormaOrganica(x, y, tam, 15, noiseOffset + 100);
        break;
      case 4: // Arp Complejo (Nivel 5)
        dibujarFormaOrganica(x, y, tam, 20, noiseOffset + 200);
        break;
      case 5: // Forma Final (Nivel 6)
        dibujarFormaOrganica(x, y, tam, 25, noiseOffset + 300);
        break;
      default:
        ellipse(x, y, tam, tam); 
        break;
    }
  }
  
  // Función para dibujar formas orgánicas usando ruido Perlin (estilo Jean Arp)
  void dibujarFormaOrganica(float cx, float cy, float t, float factorNoise, float offset) {
    beginShape();
    float maxRadius = t / 2;
    for (float a = 0; a < TWO_PI; a += 0.1) {
      float r = maxRadius + factorNoise * noise(offset + cos(a), offset + sin(a));
      vertex(cx + cos(a) * r, cy + sin(a) * r);
    }
    endShape(CLOSE);
  }


  void caer() {
    y += 3; // Simula la gravedad
  }

  // Botón 1: Permite cambiar el color
  void cambiarColor() {
    col = paleta[int(random(paleta.length))];
  }

  // Botón 2: Permite cambiar el tamaño (variación dentro del nivel actual)
  void cambiarTam() {
    // Una pequeña variación, no un cambio completo de nivel
    tam = mapaTamano(tipoForma) + random(-5, 5); 
  }
  
  // Botón 3: Permite cambiar el tipo de forma (Nivel)
  void cambiarForma() {
    tipoForma = (tipoForma + 1) % (maxFormaTipo + 1);
    tam = mapaTamano(tipoForma); // Actualizar el tamaño al cambiar el tipo
  }
  
  // Método de colisión simple (distancia entre centros)
  boolean colisionaCon(Forma otra) {
    float d = dist(x, y, otra.x, otra.y);
    return d < (tam/2 + otra.tam/2);
  }
}
```

<img src="https://raw.githubusercontent.com/DannyCatalan/Interfaz-II/refs/heads/main/img/fondo%20suika-arp.jpg"/>

