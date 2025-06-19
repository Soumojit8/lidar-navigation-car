# lidar-navigation-car
Autonomous self-driving car using ESP32 and TF-Luna LiDAR with real-time web-based distance monitoring and manual control dashboard via Chart.js.
🚗 Self-Driving Car using ESP32 and LiDAR
This project demonstrates a smart autonomous car built using an ESP32 microcontroller and a TF-Luna LiDAR sensor. The car is capable of detecting obstacles, making navigation decisions, and can also be controlled manually via a Wi-Fi-based web interface. A real-time distance graph is plotted using Chart.js for live monitoring.
🔧 Features
•	📡 TF-Luna LiDAR for real-time obstacle detection (up to 30 meters)
•	🧠 Autonomous Mode with dynamic path planning using servo-based directional scanning
•	🔹 Manual Mode via onboard web server (hosted by ESP32)
•	📊 Live Distance Chart on the web dashboard using Chart.js
•	📶 Wi-Fi AP Mode – No external router or internet required
•	⚙️ Controls 4 DC motors with L298N motor driver
🧰 Technologies Used
•	ESP32 Dev Board
•	TF-Luna LiDAR Sensor
•	L298N Motor Driver
•	Servo Motor (for scanning)
•	Chart.js (for live plotting)
•	Embedded C (Arduino) + HTML + JS
🌐 Web Interface
•	Access the ESP32 hotspot at 192.168.4.1
•	View real-time distance and control car movements
•	Responsive dashboard with live data graph and control buttons
code-
#include <ESP32Servo.h>
#include <WiFi.h>
#include <WebServer.h>

// ==== Pin Definitions ====
#define IN1 25
#define IN2 26
#define IN3 27
#define IN4 14
#define LED_PIN 2
#define SERVO_PIN 13

// ==== Global Variables ====
HardwareSerial tfLuna(2); // RX = 16, TX = 17
Servo servo;
WebServer server(80);

int distance = 0;
bool manualControl = false;
bool initialized = false;

// ==== Function Declarations ====
void handleRoot();
void handleDistance();
void moveForward();
void moveBackward();
void stopMotors();
void turnLeft();
void turnRight();
void rotateAndSearch();
int readDistanceCM();
int scanDirection(int angle);
void searchFreePathAndMove();
void searchFreeStartPath();

// ==== Setup ====
void setup() {
  Serial.begin(115200);
  tfLuna.begin(115200, SERIAL_8N1, 16, 17); // TF-Luna UART

  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  pinMode(LED_PIN, OUTPUT);

  servo.attach(SERVO_PIN);
  servo.write(90);

  WiFi.softAP("SelfCar_AP", "12345678");
  Serial.println("WiFi started at 192.168.4.1");

  server.on("/", handleRoot);
  server.on("/distance", handleDistance);
  server.on("/forward", []() { manualControl = true; moveForward(); server.send(200, "text/plain", "Forward"); });
  server.on("/backward", []() { manualControl = true; moveBackward(); server.send(200, "text/plain", "Backward"); });
  server.on("/left", []() { manualControl = true; turnLeft(); server.send(200, "text/plain", "Left"); });
  server.on("/right", []() { manualControl = true; turnRight(); server.send(200, "text/plain", "Right"); });
  server.on("/stop", []() { manualControl = true; stopMotors(); server.send(200, "text/plain", "Stopped"); });
  server.on("/auto", []() { manualControl = false; initialized = false; server.send(200, "text/plain", "Auto Mode"); });

  server.begin();
  Serial.println("Web server started");
}

// ==== Loop ====
void loop() {
  server.handleClient();

  if (manualControl) return;

  if (!initialized) {
    searchFreeStartPath();
    initialized = true;
  }

  distance = readDistanceCM();
  Serial.println("Distance: " + String(distance) + " cm");

  if (distance > 15) {
    moveForward();
    digitalWrite(LED_PIN, LOW);
  } else {
    stopMotors();
    digitalWrite(LED_PIN, HIGH);
    searchFreePathAndMove();
  }

  delay(20);
}

// ==== Web Server Handlers ====
void handleRoot() {
  String html = R"rawliteral(
  <!DOCTYPE html>
  <html>
  <head>
    <title>Self Driving Car</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  </head>
  <body>
    <h2>Self Driving Car</h2>
    <p><b>Distance:</b> <span id="dist">--</span> cm</p>

    <canvas id="chart" width="400" height="200"></canvas><br>

    <button onclick="fetch('/forward')">Forward</button>
    <button onclick="fetch('/left')">Left</button>
    <button onclick="fetch('/stop')">Stop</button>
    <button onclick="fetch('/right')">Right</button><br><br>
    <button onclick="fetch('/backward')">Backward</button><br><br>
    <button onclick="fetch('/auto')">Auto Mode</button>

    <script>
      const distLabel = document.getElementById('dist');
      const ctx = document.getElementById('chart').getContext('2d');
      let labels = [], dataPoints = [];

      const chart = new Chart(ctx, {
        type: 'line',
        data: {
          labels: labels,
          datasets: [{
            label: 'Distance (cm)',
            borderColor: 'blue',
            data: dataPoints,
            fill: false,
            tension: 0.2
          }]
        },
        options: {
          animation: false,
          scales: {
            x: { display: false },
            y: { min: 0, max: 300 }
          }
        }
      });

      setInterval(() => {
        fetch('/distance').then(res => res.text()).then(data => {
          const dist = parseInt(data);
          distLabel.innerText = dist;
          const now = new Date().toLocaleTimeString();
          labels.push(now); dataPoints.push(dist);
          if (labels.length > 30) {
            labels.shift(); dataPoints.shift();
          }
          chart.update();
        });
      }, 1000);
    </script>
  </body>
  </html>
  )rawliteral";

  server.send(200, "text/html", html);
}

void handleDistance() {
  server.send(200, "text/plain", String(distance));
}

// ==== Motor Control ====
void moveForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);
}

void stopMotors() {
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);
}

void turnLeft() {
  digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void turnRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);
}

void rotateAndSearch() {
  for (int i = 0; i < 8; i++) {
    turnRight(); delay(150);
    stopMotors(); delay(50);
    int d = readDistanceCM();
    if (d > 15) {
      moveForward();
      return;
    }
  }
  stopMotors();
}

// ==== TF-Luna Sensor ====
int readDistanceCM() {
  static uint8_t buffer[9];
  while (tfLuna.available() >= 9) {
    if (tfLuna.read() == 0x59 && tfLuna.peek() == 0x59) {
      tfLuna.read(); // consume second 0x59
      for (int i = 0; i < 7; i++) buffer[i] = tfLuna.read();
      int dist = buffer[0] + buffer[1] * 256;
      return dist;
    }
  }
  return distance;  // fallback to last known
}

int scanDirection(int angle) {
  servo.write(angle);
  delay(250);
  return readDistanceCM();
}

void searchFreePathAndMove() {
  int left = scanDirection(140);
  int right = scanDirection(40);
  int forward = scanDirection(90);

  if (left > 15) {
    turnLeft(); delay(400);
    moveForward();
  } else if (right > 15) {
    turnRight(); delay(400);
    moveForward();
  } else if (forward > 15) {
    moveForward();
  } else {
    rotateAndSearch();
  }
}

void searchFreeStartPath() {
  Serial.println("Scanning startup path...");
  searchFreePathAndMove();
}
📸 Photos
![image](https://github.com/user-attachments/assets/ae2d9ed3-fcee-46b1-aa16-167c276ac737)
![image](https://github.com/user-attachments/assets/c9ba6b05-0e07-4298-ae6c-ad22c38023a6)
🎥 Demo Video
Link 1-https://drive.google.com/file/d/1B4WZ4mDAuPPzikeV_zhRCTpZVjjvfkCv/view?usp=drivesdk
Link 2- https://drive.google.com/file/d/1AxgJNsy4dZGnNkRFMW-gRuzX6j4pDZ5j/view?usp=drivesdk
Link 3- https://drive.google.com/file/d/1Aug0IEuh4NxHzd3xdzPbduqNbRDdD0HM/view?usp=drivesdk
👤 Author
Soumojit Bakshi
Final Year B.Tech, Electronics & Communication Engineering
Email: soumojitbakshi8@gmal.com

