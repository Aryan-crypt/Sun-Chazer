```markdown
# Solar Tracking Rover

An intelligent rover that autonomously tracks the sun's position to maximize solar energy collection while navigating its environment.

![Solar Tracking Rover](https://img.shields.io/badge/Status-Working-brightgreen) ![License](https://img.shields.io/badge/License-MIT-blue) ![Platform](https://img.shields.io/badge/Platform-ESP8266-orange)

## Features

- 🔄 **Dual-Axis Solar Tracking**: Horizontal and vertical servo motors precisely position the solar panel to capture maximum sunlight throughout the day.
- 🚗 **Intelligent Navigation**: When light is insufficient, the rover autonomously moves to find better positioning based on optimal sun angle calculations.
- ⚡ **Energy Efficient**: Optimized movement patterns and smart scanning algorithms minimize power consumption while maximizing energy collection.
- 📊 **Real-time Monitoring**: Continuous voltage monitoring and angle tracking ensure the rover always operates at peak efficiency.
- 🔧 **Adaptive Movement**: Turns for 1 second when changing direction, then moves forward for 2 seconds to efficiently navigate toward light sources.
- 🌐 **ESP8266 Powered**: Built on the versatile ESP8266 platform with Wi-Fi capabilities for future IoT integrations.

## Hardware Components

### Required Components

1. **ESP8266 Development Board** (NodeMCU or similar)
2. **Servo Motors** (x2):
   - 1x for horizontal movement (bottom servo)
   - 1x for vertical movement (top servo)
3. **Solar Panel** (5V-6V output recommended)
4. **L298N Motor Driver** (for controlling DC motors)
5. **DC Motors** (x2) with wheels
6. **Chassis** for the rover
7. **Battery Pack** (7.4V-12V recommended)
8. **Jumper Wires**
9. **Breadboard or PCB for connections

### Pin Connections

#### ESP8266 to Servo Motors
- **D3** → Signal pin of Bottom Servo (horizontal movement)
- **D4** → Signal pin of Top Servo (vertical movement)
- **5V** → VCC of both servos
- **GND** → GND of both servos

#### ESP8266 to Solar Panel
- **A0** → Output pin of Solar Panel

#### ESP8266 to L298N Motor Driver
- **D1** → IN1 (Left motor forward)
- **D2** → IN2 (Left motor backward)
- **D6** → IN3 (Right motor forward)
- **D7** → IN4 (Right motor backward)
- **D5** → ENA (PWM speed control for left motors)
- **D8** → ENB (PWM speed control for right motors)
- **5V** → VCC of L298N
- **GND** → GND of L298N

#### L298N to DC Motors
- **OUT1, OUT2** → Left motor terminals
- **OUT3, OUT4** → Right motor terminals
- **12V** → Motor power input (from battery pack)

#### Power Connections
- **Battery Pack** → VIN of ESP8266 and motor power input of L298N
- **Common Ground** between all components

> **Note**: For a visual representation of the connections, visit the interactive wiring diagram on our [project website](https://your-website-url.com#connections).

## Installation

### Prerequisites

1. Arduino IDE (version 1.8.10 or higher)
2. ESP8266 Board Manager
3. Required Libraries:
   - `Servo.h` (included with Arduino IDE)

### Board Manager Setup

1. Open Arduino IDE
2. Go to **File > Preferences**
3. Add the following URL to "Additional Boards Manager URLs":
   ```
   http://arduino.esp8266.com/stable/package_esp8266com_index.json
   ```
4. Go to **Tools > Board > Boards Manager...**
5. Search for "esp8266" and install the latest version
6. Select your board from **Tools > Board > ESP8266 Boards**

### Library Installation

1. Go to **Tools > Manage Libraries...**
2. Search for "Servo" and install the Servo library by Michael Margolis

### Uploading the Code

1. Connect your ESP8266 to your computer
2. Select the correct port in **Tools > Port**
3. Upload the code using the upload button in Arduino IDE

## Usage

1. **Power On**: Connect the battery pack to power the rover
2. **Initialization**: The rover will initialize with servos centered at 90 degrees
3. **Solar Tracking**: The rover will automatically start scanning for the optimal sun position
4. **Navigation**: If the solar panel voltage is below 3.0V, the rover will move to find a better position:
   - If the optimal angle is < 50°, the rover turns left for 1 second
   - If the optimal angle is > 130°, the rover turns right for 1 second
   - Otherwise, the rover moves forward for 2 seconds
5. **Monitoring**: Connect to the ESP8266 via serial monitor (9600 baud) to see real-time data

## Code

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
```

## Project Website

For an interactive experience with 3D visualizations, connection diagrams, and detailed explanations, visit our project website: [Solar Tracking Rover Website](https://your-website-url.com)

## Troubleshooting

### Common Issues

1. **Servos Not Moving**
   - Check power connections to servos
   - Ensure servo signal wires are connected to correct pins (D3, D4)
   - Verify that the servos are getting adequate power (external power may be needed)

2. **Motors Not Responding**
   - Check L298N power connections
   - Verify motor driver pin connections
   - Ensure battery pack is charged

3. **Inaccurate Solar Tracking**
   - Check solar panel connection to A0
   - Ensure the solar panel is receiving adequate sunlight
   - Calibrate voltage thresholds if needed

4. **Rover Moving Erratically**
   - Check for loose connections
   - Ensure all components share a common ground
   - Verify that the battery can supply enough current

## Future Enhancements

- [ ] Implement Wi-Fi connectivity for remote monitoring
- [ ] Add obstacle avoidance using ultrasonic sensors
- [ ] Implement machine learning for improved sun tracking
- [ ] Add a mobile app for control and monitoring
- [ ] Implement power management and sleep modes
- [ ] Add GPS for location-based sun position calculation

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request. For major changes, please open an issue first to discuss what you would like to change.

### Steps to Contribute

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Arduino community for libraries and support
- ESP8266 community for the excellent platform
- Open source hardware suppliers

## Contact

For questions or suggestions, please open an issue or contact:
- Email: info@rover.com
- Project Website: https://your-website-url.com
- GitHub: https://github.com/yourusername/solar-tracking-rover
```

This README.md file provides a comprehensive overview of your Solar Tracking Rover project, including:

1. Clear project description and features
2. Detailed hardware requirements and pin connections
3. Step-by-step installation and setup instructions
4. Usage guidelines
5. The complete code from the website
6. Troubleshooting section for common issues
7. Future enhancement ideas
8. Contribution guidelines
9. License and contact information

The README is formatted with Markdown for easy rendering on GitHub and includes badges for project status, license, and platform. It also references the interactive website for users who want a more visual understanding of the project.
