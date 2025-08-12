# Code-For-RC-with-Arduino-Uno-or-compatible-
This Arduino-based RC car is controlled via Bluetooth using an HC-05/HC-06 module and a smartphone. Two DC motors, driven by an L298N motor driver, provide movement, while a servo motor handles steering. Commands like F (forward), B (backward), S (stop), L (left), R (right), and numbers 0–9 (speed levels) are sent from a Bluetooth terminal app. 


// RC Car - Bluetooth + L298N + Servo
// Upload to Arduino Uno
#include <Servo.h>
#include <SoftwareSerial.h>

SoftwareSerial bt(10, 11); // RX, TX to HC-05 (BT TX -> 10, BT RX <- 11)
Servo steer;

const int IN1 = 2;
const int IN2 = 3;
const int ENA = 5; // PWM left
const int IN3 = 4;
const int IN4 = 7;
const int ENB = 6; // PWM right
const int SERVO_PIN = 9;

int speedLevel = 6; // default 0..9
const int speedMap[10] = {0, 28, 56, 85, 113, 141, 170, 198, 226, 255}; // map 0..9 -> 0..255

const int SERVO_CENTER = 90;
const int SERVO_LEFT = 45;
const int SERVO_RIGHT = 135;

String inputBuffer = "";

void setup() {
  Serial.begin(9600);
  bt.begin(9600); // HC-05 default
  steer.attach(SERVO_PIN);
  steer.write(SERVO_CENTER);

  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  pinMode(ENA, OUTPUT); pinMode(ENB, OUTPUT);

  stopMotors();
  Serial.println("RC Car ready. Pair bluetooth and send commands.");
}

void loop() {
  // read from bluetooth if available
  while (bt.available()) {
    char c = bt.read();
    if (c == '\r' || c == '\n') {
      if (inputBuffer.length() > 0) {
        processCommand(inputBuffer);
        inputBuffer = "";
      }
    } else {
      inputBuffer += c;
      // if single-char command, process immediately
      if (inputBuffer.length() == 1 && isSingleCharCmd(inputBuffer.charAt(0))) {
        processCommand(inputBuffer);
        inputBuffer = "";
      }
      // allow two-char like 'F5'
      if (inputBuffer.length() >= 2) {
        processCommand(inputBuffer);
        inputBuffer = "";
      }
    }
  }

  // also allow USB Serial control for debugging
  while (Serial.available()) {
    char c = Serial.read();
    if (c == '\r' || c == '\n') continue;
    if (isSingleCharCmd(c)) {
      String s = String(c);
      processCommand(s);
    } else {
      // read possible digit
      String s = "";
      s += c;
      if (Serial.available()) {
        char d = Serial.read();
        s += d;
      }
      processCommand(s);
    }
  }
}

bool isSingleCharCmd(char c) {
  c = toupper(c);
  if (c=='F' || c=='B' || c=='S' || c=='L' || c=='R' || c=='C') return true;
  if (c >= '0' && c <= '9') return true;
  return false;
}

void processCommand(String cmd) {
  cmd.trim();
  if (cmd.length() == 0) return;
  cmd.toUpperCase();

  // Example: "F" or "B" or "F5" or "3"
  char lead = cmd.charAt(0);

  if (lead >= '0' && lead <= '9') {
    speedLevel = lead - '0';
    applySpeed();
    sendStatus();
    return;
  }

  int givenSpeed = -1;
  if (cmd.length() >= 2 && isDigit(cmd.charAt(1))) {
    givenSpeed = cmd.charAt(1) - '0';
  }

  switch (lead) {
    case 'F':
      if (givenSpeed >= 0) speedLevel = givenSpeed;
      forward();
      break;
    case 'B':
      if (givenSpeed >= 0) speedLevel = givenSpeed;
      backward();
      break;
    case 'S':
      stopMotors();
      break;
    case 'L':
      steer.write(SERVO_LEFT);
      // minor differential turning: slow right motor slightly
      // (optional) keep direction same
      break;
    case 'R':
      steer.write(SERVO_RIGHT);
      // optional differential
      break;
    case 'C':
      steer.write(SERVO_CENTER);
      break;
    default:
      // unknown
      break;
  }
  sendStatus();
}

void applySpeed() {
  int pwm = speedMap[ constrain(speedLevel, 0, 9) ];
  analogWrite(ENA, pwm);
  analogWrite(ENB, pwm);
}

void forward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  applySpeed();
}

void backward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  applySpeed();
}

void stopMotors() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  analogWrite(ENA, 0);
  analogWrite(ENB, 0);
}

void sendStatus() {
  String s = "SPD:" + String(speedLevel) + " SERVO:" + String(steer.read());
  bt.println(s);
  Serial.println(s);
}
