# Sun-Chazer Project

Sun-Chazer is a solar tracker system using the ESP8266 to adjust the solar panel's orientation for maximum efficiency. The system tracks the sun by adjusting the horizontal and vertical position of the solar panel using servo motors. It operates through a web interface, allowing users to control the system remotely.

## Components Used
- **ESP8266** - Microcontroller for Wi-Fi and control.
- **Servos** - For adjusting the solar panel's horizontal and vertical positioning.
- **Relay** - To control the motor direction.
- **Solar Panel** - To read voltage and track the sun's position.
  
## Features
- Automated solar panel tracking based on solar voltage.
- Web-based control through the ESP8266 Access Point.
- Horizontal and vertical movement of the solar panel via servo motors.
- Adjustable servo speed for smooth operation.
- Reversible motor control using relays.

## Code

```cpp
#include <Servo.h>
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>

// Wi-Fi credentials (for the AP)
const char* ssid = "Sun_Chazer_AP";       // Name of the access point
const char* password = "123456789";       // Password for the access point

// Pin definitions
const int BOTTOM_SERVO_PIN = D3;  // Horizontal movement
const int TOP_SERVO_PIN = D4;     // Vertical movement
const int SOLAR_PANEL_PIN = A0;   // Analog pin for solar panel voltage
// Pin definitions for motor control (Relay pins)
const int RELAY1_PIN = D5;  // Controls Motor Terminal 1
const int RELAY2_PIN = D6;  // Controls Motor Terminal 2

// Servo objects
Servo bottomServo;
Servo topServo;

// Create an instance of the server
ESP8266WebServer server(80); // Create a web server on port 80

// Variables to store best positions
int bestHorizontalAngle = 90;
int bestVerticalAngle = 90;
float maxVoltage = 0;

// Timing variables for non-blocking servo movement
unsigned long lastUpdateTime = 0;
int servoSpeed = 5; // Time between each servo step in milliseconds
bool horizontalServoMoving = false;
bool verticalServoMoving = false;
int currentHorizontalAngle = 90;
int currentVerticalAngle = 90;
int targetHorizontalAngle = 90;
int targetVerticalAngle = 90;

// Variables to track timing for reverse motor and tracking
unsigned long lastTrackingTime = 0;  // To trigger tracking every 5 seconds
bool motorReversing = false;          // Flag to check if motor is reversing
bool MotorStat = false;               // Motor state for tracking actions

// Function declaration for energising()
void energising();
void updateServoPosition(Servo& servo, int& currentAngle, int targetAngle, bool& moving);
void initialisehor();
void initialisever();

// Function to read voltage from the solar panel
float readVoltage() {
  int sensorValue = analogRead(SOLAR_PANEL_PIN);  // Read analog pin A0
  float voltage = sensorValue * (3.3 / 1023.0);   // Convert the value to voltage
  return voltage;
}

void setup() {
  Serial.begin(9600);
  
  bottomServo.attach(BOTTOM_SERVO_PIN);
  topServo.attach(TOP_SERVO_PIN);

  // Initialize relays
  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, OUTPUT);
  stopMotor();
  
  // Initialize servos to middle position
  bottomServo.write(90);
  topServo.write(90);
  
  delay(1000);  // Wait for servos to reach initial position
  
  // Set up the ESP8266 as an Access Point
  WiFi.softAP(ssid, password);
  Serial.println("Access Point Started");
  Serial.print("IP address: ");
  Serial.println(WiFi.softAPIP());  // Print the IP address of the ESP8266
  
  // Define routes for the web server
  server.on("/", HTTP_GET, handleRoot);
  server.on("/start_tracking", HTTP_GET, energising);
  
  // Start the server
  server.begin();
}

void loop() {
  server.handleClient();  // Handle client requests

  energising();  // Perform energising function periodically
  delay(100);  // Small delay for smoother execution
}

// Function to handle web interface root
void handleRoot() {
  String html = "<!DOCTYPE html>"
                "<html lang=\"en\">"
                "<head>"
                "<meta charset=\"UTF-8\">"
                "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">"
                "<title>Sun Chazer Control</title>"
                "<style>"
                "body { font-family: Arial, sans-serif; background-color: #add8e6; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; color: #333; background-image: url('https://wallpapers.com/images/featured/8k-ultra-hd-nature-prdfm720u6780jl5.jpg'); background-repeat: no-repeat; }"
                ".container { text-align: center;backdrop-filter: blur(10px); -webkit-backdrop-filter: blur(10px); background-color: rgba(256,256,256, 0.5); padding: 2rem; border-radius: 15px; box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1); max-width: 400px; width: 100%; }"
                "h1 { color: #388e3c; margin-bottom: 20px; }"
                ".button { display: inline-block; padding: 12px 24px; margin: 10px; font-size: 18px; cursor: pointer; text-align: center; text-decoration: none; outline: none; color: #fff; background-color: #4CAF50; border: none; border-radius: 25px; box-shadow: 0 5px #999; transition: all 0.3s ease; }"
                ".button:hover { background-color: #45a049; }"
                ".button:active { background-color: #3e8e41; box-shadow: 0 3px #666; transform: translateY(2px); }"
                "#status { margin-top: 20px; font-weight: bold; font-size: 18px; background-color: #ffcccb; padding: 10px; border-radius: 5px; }"
                "input { padding: 10px; font-size: 16px; border-radius: 5px; border: 1px solid #ccc; margin-bottom: 10px; width: 80%; }"
                "</style>"
                "</head>"
                "<body>"
                "<div class=\"container\">"
                "<h1>Sun Chazer Control</h1>"
                "<input type=\"text\" id=\"ipAddress\" placeholder=\"Enter IP Address\">"
                "<button class=\"button\" onclick=\"startTracking()\">Start Tracking</button>"
                "<div id=\"status\">Status: Ready</div>"
                "</div>"
                "<script>"
                "function startTracking() {"
                "    fetch('/start_tracking')"
                "        .then(response => response.text())"
                "        .then(data => {"
                "            document.getElementById('status').textContent = 'Status: ' + data; "
                "        });"
                "} "
                "</script>"
                "</body>"
                "</html>";
  server.send(200, "text/html", html);
}

void energising() {
  float voltage = readVoltage();  // Get the current voltage from the solar panel
  if (voltage > 0.00) {
    if (!MotorStat) {
      Serial.println("Motor Reverse");
      reverseMotor();
      delay(3000);  // Let the motor run for 3 seconds
      stopMotor();
      MotorStat = true;
    }
    handleStartTracking();
  } else {
    Serial.println("Motor Forward");
    forwardMotor();
    delay(3000);  // Let the motor run for 3 seconds
    stopMotor();
    MotorStat = false;
  }
}

// Handle start tracking request
void handleStartTracking() {
  // Reset for a new tracking cycle
  maxVoltage = 0;
  bestHorizontalAngle = 90;
  bestVerticalAngle = 90;
  
  // Start tracking
  Serial.println("Tracking started...");
  initialisehor(); // Find best horizontal position
  
  // Set the target horizontal angle
  targetHorizontalAngle = bestHorizontalAngle;
  horizontalServoMoving = true;  // Start moving horizontal servo

  initialisever(); // Find best vertical position

  // Set the target vertical angle
  targetVerticalAngle = bestVerticalAngle;
  verticalServoMoving = true;  // Start moving vertical servo

  // Report final position via Serial
  Serial.println("Final Position:");
  Serial.print("Horizontal Angle: ");
  Serial.print(bestHorizontalAngle);
  Serial.print(" (");
  Serial.print(bestHorizontalAngle <= 90 ? "Left" : "Right");
  Serial.print("), Vertical Angle: ");
  Serial.print(bestVerticalAngle);
  Serial.print(" (");
  Serial.print(bestVerticalAngle <= 90 ? "Down" : "Up");
  Serial.println(")");
  Serial.print("Max Voltage: ");
  Serial.print(maxVoltage);
  Serial.println("V");

  // Now move to the best position
  horizontalServoMoving = true;
  verticalServoMoving = true;

  // Start servo movements to the best position
  moveToBestPosition(); 

  // Add a 3-second delay after tracking process is complete
  delay(10000);  // Delay for 3 seconds before the next cycle or action

  server.send(200, "text/plain", "Tracking Done!");
}

void moveToBestPosition() {
  // Move horizontal servo to the best horizontal angle
  while (currentHorizontalAngle != targetHorizontalAngle) {
    updateServoPosition(bottomServo, currentHorizontalAngle, targetHorizontalAngle, horizontalServoMoving);
    delay(20); // Delay to give the servo time to move
  }

  // Move vertical servo to the best vertical angle
  while (currentVerticalAngle != targetVerticalAngle) {
    updateServoPosition(topServo, currentVerticalAngle, targetVerticalAngle, verticalServoMoving);
    delay(20); // Delay to give the servo time to move
  }
}

// Horizontal tracking function
void initialisehor() {
  // Horizontal sweep with even smaller steps for more precise tracking
  for (int angle = 0; angle <= 180; angle++) {  // Step size of 1 degree for finer movement
    bottomServo.write(angle);
    delay(20);  // Allow time for servo to move and reading to stabilize

    float voltage = readVoltage();

    // Averaging over multiple readings for more accurate results
    float avgVoltage = 0;
    int readings = 5;  // Number of readings to average
    for (int i = 0; i < readings; i++) {
      avgVoltage += readVoltage();
      delay(10);  // Small delay between readings
    }
    avgVoltage /= readings;  // Average voltage

    if (avgVoltage > maxVoltage) {
      maxVoltage = avgVoltage;
      bestHorizontalAngle = angle;
    }

    // Output voltage and angle for monitoring
    Serial.print("Horizontal Angle: ");
    Serial.print(angle);
    Serial.print(", Avg Voltage: ");
    Serial.println(avgVoltage, 4);  // Display voltage with 4 decimal places for precision
  }
}

// Vertical tracking function (Down to Up scan)
void initialisever() {
  // Vertical sweep with smaller steps for smoother tracking
  for (int angle = 20; angle <= 170; angle++) {  // Step size of 1 degree for vertical movement
    topServo.write(angle);  // Move the vertical servo to the current angle
    delay(20);  // Allow time for servo to move and reading to stabilize

    float voltage = readVoltage();

    // Averaging over multiple readings for more accurate results
    float avgVoltage = 0;
    int readings = 5;  // Number of readings to average
    for (int i = 0; i < readings; i++) {
      avgVoltage += readVoltage();
      delay(10);  // Small delay between readings
    }
    avgVoltage /= readings;  // Average voltage

    if (avgVoltage > maxVoltage) {
      maxVoltage = avgVoltage;
      bestVerticalAngle = angle;
    }

    // Output voltage and angle for monitoring
    Serial.print("Vertical Angle: ");
    Serial.print(angle);
    Serial.print(", Avg Voltage: ");
    Serial.println(avgVoltage, 4);  // Display voltage with 4 decimal places for precision
  }
}

// Function to smoothly update the servo position (adjusted for precision)
void updateServoPosition(Servo &servo, int &currentAngle, int targetAngle, bool &movingFlag) {
  unsigned long currentTime = millis();

  // Check if it's time to move the servo
  if (currentTime - lastUpdateTime >= servoSpeed) {
    lastUpdateTime = currentTime;

    if (currentAngle < targetAngle) {
      currentAngle++;
      servo.write(currentAngle);
    } else if (currentAngle > targetAngle) {
      currentAngle--;
      servo.write(currentAngle);
    } else {
      movingFlag = false;  // Stop moving if we reached the target
    }
  }
}

// Motor control functions
void forwardMotor() {
  digitalWrite(RELAY1_PIN, HIGH);
  digitalWrite(RELAY2_PIN, LOW);
  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, INPUT);
  Serial.println("Relay 1 ON, Relay 2 OFF");
}

void reverseMotor() {
  digitalWrite(RELAY1_PIN, LOW);
  digitalWrite(RELAY2_PIN, HIGH);
  pinMode(RELAY1_PIN, INPUT);
  pinMode(RELAY2_PIN, OUTPUT);
  Serial.println("Relay 1 OFF, Relay 2 ON");
}

void stopMotor() {
  digitalWrite(RELAY1_PIN, LOW);
  digitalWrite(RELAY2_PIN, LOW);
  pinMode(RELAY1_PIN, INPUT);
  pinMode(RELAY2_PIN, INPUT);
  Serial.println("Both Relays OFF");
}
