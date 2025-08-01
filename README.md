# 🌞 SunChazer - Arduino Based Solar Tracking Rover

SunChazer is a smart solar panel rover that **tracks sunlight**, and when it doesn’t find enough light, it **moves forward or rotates** to find a better sunlight spot. It also has a **flap system** (6 solar panels connected) that **opens when there is sunlight** and **closes when it’s dark**, controlled using an N20 motor with relays.

---

## 🧠 Features

- Tracks sunlight using LDRs.
- Moves forward, left, or right based on best sunlight angle.
- Stops when solar voltage is high.
- Opens/closes flaps automatically based on solar voltage.
- Fully controlled by one Arduino Uno.
- Uses L298N motor driver and two servo motors.

---

## 📦 Hardware Required

| Component         | Quantity |
|------------------|----------|
| Arduino Uno       | 1        |
| L298N Motor Driver | 1        |
| Servo Motor (SG90) | 2        |
| N20 Motor         | 1        |
| 2-Channel Relay Module | 1    |
| LDR Sensors       | 4        |
| 10kΩ and 6.8kΩ resistors (Voltage Divider) | 1 each |
| BO Motors (for wheels) | 6 (3 left, 3 right) |
| 15V battery (for motor driver) | 1 |
| 3.7V Li-ion battery (for relay boost to 5V) | 1 |
| Solar Panel (2–6V) | 1        |
| Wires, breadboard, jumper wires | As needed |

---

## 🛠️ Wiring Summary

### Servo Motors:
- Horizontal Servo: Pin D3
- Vertical Servo: Pin D4

### L298N Motor Driver:
- IN1 → D1  
- IN2 → D2  
- IN3 → D6  
- IN4 → D7  
- ENA (speed left) → D5  
- ENB (speed right) → D8  
- VCC → 12V–15V battery  
- GND → Arduino GND  
- 5V (from motor driver) NOT connected to Arduino 5V

### LDR Sensors:
- A1, A2, A3, A4 (for LDRs facing different directions)

### Solar Voltage Reading:
- A0 (through voltage divider)

**Voltage Divider Circuit** (for 0–6V to safe range for A0):
- Solar +ve → R1 (10kΩ) → A0  
- A0 → R2 (6.8kΩ) → GND

### Relay Control (for Flaps):
- Relay1 IN → D9  
- Relay2 IN → D10  
- Relay VCC → 5V (via boost converter if needed)  
- Relay GND → Arduino GND

---

## 🧾 Arduino Code

```cpp
#include <Servo.h>

Servo horizontal;
Servo vertical;

const int ldrTopLeft = A1;
const int ldrTopRight = A2;
const int ldrBottomLeft = A3;
const int ldrBottomRight = A4;

const int voltagePin = A0;

const int motorIn1 = 1;
const int motorIn2 = 2;
const int motorIn3 = 6;
const int motorIn4 = 7;
const int motorENA = 5;
const int motorENB = 8;

const int flapRelay1 = 9;
const int flapRelay2 = 10;

int posH = 90;
int posV = 90;

bool flapsOpen = true;
unsigned long lastFlapAction = 0;

void setup() {
  Serial.begin(9600);
  horizontal.attach(3);
  vertical.attach(4);

  pinMode(motorIn1, OUTPUT);
  pinMode(motorIn2, OUTPUT);
  pinMode(motorIn3, OUTPUT);
  pinMode(motorIn4, OUTPUT);
  pinMode(motorENA, OUTPUT);
  pinMode(motorENB, OUTPUT);
  
  pinMode(flapRelay1, OUTPUT);
  pinMode(flapRelay2, OUTPUT);

  horizontal.write(posH);
  vertical.write(posV);
}

void loop() {
  float voltage = readVoltage();
  Serial.print("Solar Voltage: ");
  Serial.println(voltage);

  if (voltage <= 0.6 && flapsOpen) {
    closeFlaps();
    flapsOpen = false;
    stopMotors();
    return;
  }

  if (voltage > 0.6 && !flapsOpen) {
    openFlaps();
    flapsOpen = true;
  }

  scanSun();
  delay(3000);

  if (voltage <= 3.0) {
    if (posH < 80) {
      turnLeft();
    } else if (posH > 100) {
      turnRight();
    } else {
      moveForward();
    }
    delay(3000);
    stopMotors();
  }
}

float readVoltage() {
  int adcValue = analogRead(voltagePin);
  float voltage = (adcValue * 5.0 / 1023.0) * ((10.0 + 6.8) / 6.8);
  return voltage;
}

void scanSun() {
  int tl = analogRead(ldrTopLeft);
  int tr = analogRead(ldrTopRight);
  int bl = analogRead(ldrBottomLeft);
  int br = analogRead(ldrBottomRight);

  int avgTop = (tl + tr) / 2;
  int avgBottom = (bl + br) / 2;
  int avgLeft = (tl + bl) / 2;
  int avgRight = (tr + br) / 2;

  if (avgTop < avgBottom && posV < 170) posV += 2;
  else if (avgBottom < avgTop && posV > 10) posV -= 2;

  if (avgLeft < avgRight && posH < 170) posH += 2;
  else if (avgRight < avgLeft && posH > 10) posH -= 2;

  horizontal.write(posH);
  vertical.write(posV);
}

void moveForward() {
  digitalWrite(motorIn1, HIGH);
  digitalWrite(motorIn2, LOW);
  digitalWrite(motorIn3, HIGH);
  digitalWrite(motorIn4, LOW);
  analogWrite(motorENA, 255);
  analogWrite(motorENB, 255);
}

void turnLeft() {
  digitalWrite(motorIn1, LOW);
  digitalWrite(motorIn2, HIGH);
  digitalWrite(motorIn3, HIGH);
  digitalWrite(motorIn4, LOW);
  analogWrite(motorENA, 255);
  analogWrite(motorENB, 255);
}

void turnRight() {
  digitalWrite(motorIn1, HIGH);
  digitalWrite(motorIn2, LOW);
  digitalWrite(motorIn3, LOW);
  digitalWrite(motorIn4, HIGH);
  analogWrite(motorENA, 255);
  analogWrite(motorENB, 255);
}

void stopMotors() {
  digitalWrite(motorIn1, LOW);
  digitalWrite(motorIn2, LOW);
  digitalWrite(motorIn3, LOW);
  digitalWrite(motorIn4, LOW);
  analogWrite(motorENA, 0);
  analogWrite(motorENB, 0);
}

void openFlaps() {
  digitalWrite(flapRelay1, HIGH);
  digitalWrite(flapRelay2, LOW);
  delay(2000);
  digitalWrite(flapRelay1, LOW);
  digitalWrite(flapRelay2, LOW);
}

void closeFlaps() {
  digitalWrite(flapRelay1, LOW);
  digitalWrite(flapRelay2, HIGH);
  delay(2000);
  digitalWrite(flapRelay1, LOW);
  digitalWrite(flapRelay2, LOW);
}
