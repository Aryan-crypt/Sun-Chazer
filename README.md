# 🌞 SunChazer - Smart Solar Panel & Rover System

SunChazer is a smart Arduino-based system that:
- Automatically scans sunlight using two servos (horizontal + vertical).
- Tracks the sun's position and aligns 6 solar panels to the best angle.
- If solar voltage drops below 1.5V (like in shade or night), the rover moves in search of better light.
- The rover moves forward or turns left/right based on sun position.
- 6 solar panels are opened or closed automatically using an N20 motor controlled by 2 relays.
- Flaps close at night (solar voltage < 0.6V), and open in the morning (voltage > 0.6V).

🔗 **Live Project Website**: [Click here to visit](https://graceful-chebakia-c9f383.netlify.app/)

---

## 🛠 Components Used

- Arduino Uno
- 2x Servo motors (for sun tracking)
- 6x BO motors (3 left, 3 right for rover)
- L298N Motor Driver
- 2x Relays for N20 motor control
- N20 Gear Motor (for flap control)
- 15V Battery (for motor power)
- 3.7V Battery (for relay power)
- Voltage divider resistors (to scale 6V panel output to 3.3V)
- Solar Panel (6V max)
- Wires, Breadboard, etc.

---

## ⚡ Connections

### 🔋 Power Supply
- Arduino powered via USB or 5V regulator from 15V battery.
- Relays powered by 3.7V Li-ion battery.
- N20 motor powered by 15V via relays.

### 🧠 Arduino Pins

| Component            | Arduino Pin |
|---------------------|-------------|
| Servo Horizontal     | D3          |
| Servo Vertical       | D4          |
| Motor Driver IN1     | D1          |
| Motor Driver IN2     | D2          |
| Motor Driver IN3     | D6          |
| Motor Driver IN4     | D7          |
| Relay 1 (N20 Dir1)   | D5          |
| Relay 2 (N20 Dir2)   | D8          |
| Voltage Sensor (Solar)| A0         |

> Note: We removed motor speed (ENA/ENB) to free pins for N20 logic.

---

## 📈 Voltage Divider

Solar Panel outputs up to 6V. A voltage divider scales it down to < 3.3V for Arduino's A0:

**Use:**
- R1 = 10kΩ (from panel + to A0)
- R2 = 10kΩ (from A0 to GND)

Formula used:  
`Vout = Vin × (R2 / (R1 + R2))`  
6V → 3V (safe for Arduino)

---

## 🧠 Working Logic

1. **Sun Scanning:**
   - Every 3 sec, horizontal + vertical servos scan the sun.
   - Panel aligns to the best light direction.

2. **Rover Movement:**
   - If max solar voltage ≤ 1.5V:
     - Rover moves forward or rotates to better angle.
     - Motor direction chosen based on best horizontal servo angle.

3. **Flap Control via N20 Motor:**
   - If voltage < 0.6V → N20 closes flaps (2 seconds).
   - Waits until voltage rises above 0.6V.
   - Then → N20 opens flaps (2 seconds).
   - Logic uses relays to reverse motor direction.

---

## ✅ Full Arduino Code

```cpp
#include <Servo.h>

Servo horizontal;
Servo vertical;

const int servoH = 3;
const int servoV = 4;

// Motor pins
#define IN1 1
#define IN2 2
#define IN3 6
#define IN4 7

// Relay pins for N20 motor
#define RELAY1 5
#define RELAY2 8

// Sensor pin
#define SENSOR A0

int hMin = 0, hMax = 180;
int vMin = 0, vMax = 90;
int bestH = 90, bestV = 45;

unsigned long lastScanTime = 0;
const unsigned long scanInterval = 3000;

bool flapsOpen = false;

void setup() {
  Serial.begin(9600);
  horizontal.attach(servoH);
  vertical.attach(servoV);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  pinMode(RELAY1, OUTPUT);
  pinMode(RELAY2, OUTPUT);

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);

  digitalWrite(RELAY1, LOW);
  digitalWrite(RELAY2, LOW);

  horizontal.write(bestH);
  vertical.write(bestV);
}

void loop() {
  float voltage = analogRead(SENSOR) * (5.0 / 1023.0);

  if (voltage < 0.6) {
    if (flapsOpen) {
      closeFlaps();
      flapsOpen = false;
    }
    delay(500);
    return;
  } else {
    if (!flapsOpen) {
      openFlaps();
      flapsOpen = true;
    }
  }

  if (millis() - lastScanTime >= scanInterval) {
    scanSun();
    lastScanTime = millis();
    if (getVoltage(bestH, bestV) <= 1.5) {
      moveRover();
    }
  }
}

void scanSun() {
  float maxVoltage = 0;
  for (int h = hMin; h <= hMax; h += 30) {
    horizontal.write(h);
    delay(300);
    for (int v = vMin; v <= vMax; v += 30) {
      vertical.write(v);
      delay(300);
      float vVal = getVoltage(h, v);
      if (vVal > maxVoltage) {
        maxVoltage = vVal;
        bestH = h;
        bestV = v;
      }
    }
  }
  horizontal.write(bestH);
  vertical.write(bestV);
}

float getVoltage(int h, int v) {
  return analogRead(SENSOR) * (5.0 / 1023.0);
}

void moveRover() {
  if (bestH < 60) {
    turnLeft();
  } else if (bestH > 120) {
    turnRight();
  } else {
    moveForward();
  }
}

void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  delay(3000);
  stopMotors();
}

void turnLeft() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  delay(3000);
  stopMotors();
}

void turnRight() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  delay(3000);
  stopMotors();
}

void stopMotors() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

void openFlaps() {
  digitalWrite(RELAY1, HIGH);
  digitalWrite(RELAY2, LOW);
  delay(2000);
  digitalWrite(RELAY1, LOW);
  digitalWrite(RELAY2, LOW);
}

void closeFlaps() {
  digitalWrite(RELAY1, LOW);
  digitalWrite(RELAY2, HIGH);
  delay(2000);
  digitalWrite(RELAY1, LOW);
  digitalWrite(RELAY2, LOW);
}
