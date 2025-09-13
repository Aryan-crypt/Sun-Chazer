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

// --- Pin Definitions ---
// Servo Pins
const int BOTTOM_SERVO_PIN = D3;   // Horizontal movement
const int TOP_SERVO_PIN = D4;      // Vertical movement

// Solar Panel Analog Pin
const int SOLAR_PANEL_PIN = A0;    // Reads panel voltage

// Motor Driver (L298N) Pins (ESP8266 Mapping)
const int IN1 = D1;  // Left Forward
const int IN2 = D2;  // Left Backward
const int IN3 = D6;  // Right Forward
const int IN4 = D7;  // Right Backward
const int ENA = D5;  // PWM Left motors
const int ENB = D8;  // PWM Right motors

// --- Constants ---
#define LEFT_THRESHOLD 50
#define RIGHT_THRESHOLD 130
int speedValue = 255; // PWM speed (0-255)

// --- Servo Setup ---
Servo bottomServo;
Servo topServo;

// --- Tracking Variables ---
int bestHorizontalAngle = 90;
int bestVerticalAngle = 90;
float maxVoltage = 0;

int currentHorizontalAngle = 90;
int currentVerticalAngle = 90;

void setup() {
  Serial.begin(9600);

  // Attach servos
  bottomServo.attach(BOTTOM_SERVO_PIN);
  topServo.attach(TOP_SERVO_PIN);

  // Initialize motor pins
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);

  stopMotors();
  bottomServo.write(90);
  topServo.write(90);
  delay(1000);
}

void loop() {
  energising();
  delay(100);
}

// --- MAIN LOGIC ---
void energising() {
  float voltage = readVoltage();
  Serial.print("Initial voltage: "); Serial.println(voltage);

  handleStartTracking();

  if (maxVoltage <= 3.0) {
    Serial.println("[LOW VOLTAGE] Initiating rover movement...");
    moveRoverBasedOnAngle(bestHorizontalAngle);
  }

  delay(3000); // 3 second delay like original ESP logic
}

// --- Solar Tracking ---
void handleStartTracking() {
  maxVoltage = 0;
  bestHorizontalAngle = 90;
  bestVerticalAngle = 90;

  Serial.println("Starting Sun Tracking...");
  initialisehor();
  initialisever();

  moveToBestPosition();

  Serial.print("Best H Angle: "); Serial.println(bestHorizontalAngle);
  Serial.print("Best V Angle: "); Serial.println(bestVerticalAngle);
  Serial.print("Max Voltage: "); Serial.println(maxVoltage);
}

void initialisehor() {
  for (int angle = 0; angle <= 180; angle++) {
    bottomServo.write(angle);
    delay(20);

    float avgVoltage = 0;
    for (int i = 0; i < 5; i++) {
      avgVoltage += readVoltage();
      delay(10);
    }
    avgVoltage /= 5.0;

    if (avgVoltage > maxVoltage) {
      maxVoltage = avgVoltage;
      bestHorizontalAngle = angle;
    }
  }
}

void initialisever() {
  for (int angle = 20; angle <= 90; angle++) {
    topServo.write(angle);
    delay(20);

    float avgVoltage = 0;
    for (int i = 0; i < 5; i++) {
      avgVoltage += readVoltage();
      delay(10);
    }
    avgVoltage /= 5.0;

    if (avgVoltage > maxVoltage) {
      maxVoltage = avgVoltage;
      bestVerticalAngle = angle;
    }
  }
}

void moveToBestPosition() {
  bottomServo.write(bestHorizontalAngle);
  topServo.write(bestVerticalAngle);
  delay(500);
}

float readVoltage() {
  int sensorValue = analogRead(SOLAR_PANEL_PIN);
  float voltage = sensorValue * (3.3 / 1023.0);  // ESP8266 analog range is 0–3.3V
  return voltage;
}

// --- Rover Movement Based on Angle ---
void moveRoverBasedOnAngle(int angle) {
  if (angle < LEFT_THRESHOLD) {
    turnLeft();
  } else if (angle > RIGHT_THRESHOLD) {
    turnRight();
  } else {
    moveForward();
  }
  delay(3000);
  stopMotors();
}

// --- Motor Movement Functions ---
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  analogWrite(ENA, speedValue);
  analogWrite(ENB, speedValue);
  Serial.println("Moving Forward");
}

void turnLeft() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  analogWrite(ENA, speedValue);
  analogWrite(ENB, speedValue);
  Serial.println("Turning Left");
}

void turnRight() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  analogWrite(ENA, speedValue);
  analogWrite(ENB, speedValue);
  Serial.println("Turning Right");
}

void stopMotors() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  analogWrite(ENA, 0);
  analogWrite(ENB, 0);
  Serial.println("Motors Stopped");
}
