# My-first-IOT-project
Automated Fire Extenguiher system using iot, that helps in detecting the fire at an early stages and extenguishing it without any human intervention. 




source code ( we run this in Arduino IDE software)
 #include <Servo.h>

const int flameSensor1Pin = A0;   // Flame sensor 1 (left)
const int flameSensor2Pin = A1;   // Flame sensor 2 (right)
const int relayPin        = 7;    // Relay pin (active-low)
const int servoPin        = 9;    // Servo motor pin

// Tuning
const int SAMPLE_COUNT    = 5;    // number of analog samples to average             
const int FLAME_THRESH    = 300;  // analog threshold (0-1023). lower -> flame
const unsigned long PUMP_ON_MS    = 3000UL; // pump ON duration in ms
const unsigned long BETWEEN_MS    = 2000UL; // pause between sweeps in ms

Servo myServo;

// State machine
enum SystemState { IDLE, LEFT_ACTION, RIGHT_ACTION, SWEEP_LEFT, SWEEP_RIGHT };
SystemState state = IDLE;

unsigned long stateStartTime = 0; // when we entered the current state
unsigned long actionEndTime = 0;  // when to finish current pump action

// helper: read averaged analog value
int readAvg(int pin) {
  long sum = 0;
  for (int i = 0; i < SAMPLE_COUNT; ++i) {
    sum += analogRead(pin);
    delay(5); // small spacing between samples
  }
  return (int)(sum / SAMPLE_COUNT);
}

void setup() {
  Serial.begin(9600);

  pinMode(flameSensor1Pin, INPUT);
  pinMode(flameSensor2Pin, INPUT);
  pinMode(relayPin, OUTPUT);  

  myServo.attach(servoPin);
  myServo.write(90);               // center initially

  digitalWrite(relayPin, HIGH);    // Relay OFF initially (active-low)
  state = IDLE;
  stateStartTime = millis();
}

void loop() {
  // Read and smooth sensors
  int rawLeft  = readAvg(flameSensor1Pin);
  int rawRight = readAvg(flameSensor2Pin);

  bool flameLeft  = (rawLeft  < FLAME_THRESH);
  bool flameRight = (rawRight < FLAME_THRESH);

  // Debug
  Serial.print("L:"); Serial.print(rawLeft);
  Serial.print(flameLeft ? " [F]" : " [ ]");
  Serial.print("  R:"); Serial.print(rawRight);
  Serial.print(flameRight ? " [F]" : " [ ]");
  Serial.print("  State:"); Serial.println((int)state);

  unsigned long now = millis();

  // Decide next state only when in IDLE (or maintain current action until finished)
  switch (state) {
    case IDLE:
      if (flameLeft && !flameRight) {
        state = LEFT_ACTION;
        stateStartTime = now;
        actionEndTime = now + PUMP_ON_MS;
        myServo.write(0);            // point left
        digitalWrite(relayPin, LOW); // pump ON (active-low)
      }
      else if (flameRight && !flameLeft) {
        state = RIGHT_ACTION;
        stateStartTime = now;
        actionEndTime = now + PUMP_ON_MS;
        myServo.write(180);          // point right
        digitalWrite(relayPin, LOW); // pump ON
      }
      else if (flameLeft && flameRight) {
        // start sweep: left first
        state = SWEEP_LEFT;
        stateStartTime = now;
        actionEndTime = now + PUMP_ON_MS;
        myServo.write(0);
        digitalWrite(relayPin, LOW);
      }
      else {
        // no flame -> ensure center and pump off
        myServo.write(90);
        digitalWrite(relayPin, HIGH);
      }
      break;

    case LEFT_ACTION:
      // keep pump on until time elapses
      if (now >= actionEndTime) {
        digitalWrite(relayPin, HIGH); // pump OFF
        state = IDLE;
        stateStartTime = now;
      }
      break;

    case RIGHT_ACTION:
      if (now >= actionEndTime) {
        digitalWrite(relayPin, HIGH);
        state = IDLE;
        stateStartTime = now;
      }
      break;

    case SWEEP_LEFT:
      if (now >= actionEndTime) {
        // finished left spray -> turn pump off, wait BETWEEN_MS then go right
        digitalWrite(relayPin, HIGH);
        state = SWEEP_RIGHT;
        stateStartTime = now + BETWEEN_MS;       // schedule right action start after pause
        actionEndTime = stateStartTime + PUMP_ON_MS;
      }
      break;

    case SWEEP_RIGHT:
      // If we set stateStartTime in SWEEP_LEFT as now + BETWEEN_MS, ensure we start right action at that time:
      if (now >= stateStartTime && digitalRead(relayPin) == HIGH) {
        // start right action (if not already started)
        myServo.write(180);
        digitalWrite(relayPin, LOW);
      }
      if (now >= actionEndTime) {
        digitalWrite(relayPin, HIGH);
        state = IDLE;
        stateStartTime = now;
      }
      break;
  }

  // small loop delay to avoid spamming serial too fast
  delay(100);
}
