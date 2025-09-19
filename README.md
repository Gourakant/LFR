# LFR Basic Code
//Basic LFR code:

// ==== IR Sensor Pins ====
#define IR1 A0  // Far Left
#define IR2 A1  // Mid Left
#define IR3 A2  // Center
#define IR4 A3  // Mid Right
#define IR5 A4  // Far Right

// ==== Motor Driver Pins ====
#define IN1 4
#define IN2 5
#define IN3 6
#define IN4 7
#define EN1 10
#define EN2 11

// ==== Speed Settings ====
const int baseSpeed = 90;
const int sharpTurnSpeed = 80;
const int curveSpeed = 70;
const int reverseSpeed = 80;

// ==== State Machine ====
enum TrackState { NORMAL, L_TURN_LEFT, L_TURN_RIGHT, T_SECTION, CURVE, LOST };
TrackState currentState = NORMAL;
unsigned long lastIntersectionTime = 0;
bool isTurning = false;

void setup() {
  pinMode(IR1, INPUT); pinMode(IR2, INPUT); pinMode(IR3, INPUT);
  pinMode(IR4, INPUT); pinMode(IR5, INPUT);

  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  pinMode(EN1, OUTPUT); pinMode(EN2, OUTPUT);

  Serial.begin(9600);
}

void loop() {
  int s[5] = {
    !digitalRead(IR1),  // Leftmost
    !digitalRead(IR2),  // Left mid
    !digitalRead(IR3),  // Center
    !digitalRead(IR4),  // Right mid
    !digitalRead(IR5)   // Rightmost
  };

  // === Determine Track State ===
  if (s[0] && s[4] && millis() - lastIntersectionTime > 500) {
    currentState = T_SECTION;
    lastIntersectionTime = millis();
  } 
  else if ((s[0] || s[1]) && !s[2] && !s[3] && !s[4]) {
    currentState = L_TURN_LEFT;
  } 
  else if ((s[3] || s[4]) && !s[0] && !s[1] && !s[2]) {
    currentState = L_TURN_RIGHT;
  } 
  else if (s[1] || s[3]) {
    currentState = CURVE;
  } 
  else if (s[2]) {
    currentState = NORMAL;
  } 
  else {
    currentState = LOST;
  }

  // === Execute State Action ===
  switch (currentState) {
    case T_SECTION:
      handleTSection(s);
      break;
    case L_TURN_LEFT:
      followLeftL(s);
      break;
    case L_TURN_RIGHT:
      followRightL(s);
      break;
    case CURVE:
      followCurve(s[1], s[3]);
      break;
    case NORMAL:
      forward();
      break;
    case LOST:
      searchLine();
      break;
  }
}

void forward() {
  analogWrite(EN1, baseSpeed);
  analogWrite(EN2, baseSpeed);
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void followLeftL(int s[5]) {
  if (!isTurning) {
    isTurning = true;
    
    // Turn LEFT (counterclockwise)
    analogWrite(EN1, sharpTurnSpeed);
    analogWrite(EN2, sharpTurnSpeed);
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  // Left reverse
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);  // Right forward

    // Wait until center sensor realigns
    unsigned long start = millis();
    while (millis() - start < 1000) {
      if (!digitalRead(IR3)) break;
    }
    
    forward();
    delay(100);
    isTurning = false;
  }
}

void followRightL(int s[5]) {
  if (!isTurning) {
    isTurning = true;
    
    // Turn RIGHT (clockwise)
    analogWrite(EN1, sharpTurnSpeed);
    analogWrite(EN2, sharpTurnSpeed);
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);  // Left forward
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  // Right reverse

    // Wait until center sensor realigns
    unsigned long start = millis();
    while (millis() - start < 1000) {
      if (!digitalRead(IR3)) break;
    }
    
    forward();
    delay(100);
    isTurning = false;
  }
}

void handleTSection(int s[5]) {
  // Default right turn
  followRightL(s);
  delay(300);
}

void followCurve(bool leftActive, bool rightActive) {
  int speedLeft = curveSpeed;
  int speedRight = curveSpeed;

  if (leftActive)  speedRight -= 30;
  if (rightActive) speedLeft -= 30;

  analogWrite(EN1, speedLeft);
  analogWrite(EN2, speedRight);
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void searchLine() {
  analogWrite(EN1, 100);
  analogWrite(EN2, 100);
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);
}

# LFR Modified Code
// ==== IR Sensor Pins ====
#define IR1 A0  // Far Left
#define IR2 A1  // Mid Left
#define IR3 A2  // Center
#define IR4 A3  // Mid Right
#define IR5 A4  // Far Right

// ==== Motor Driver Pins ====
#define IN1 4
#define IN2 5
#define IN3 6
#define IN4 7
#define EN1 10
#define EN2 11

// ==== Speed Settings ====
const int baseSpeed = 90+40;
const int sharpTurnSpeed = 80+40;
const int curveSpeed = 70+40;
const int reverseSpeed = 80+40;

// ==== State Machine ====
enum TrackState { NORMAL, L_TURN_LEFT, L_TURN_RIGHT, T_SECTION, CURVE, LOST, FINISH };
TrackState currentState = NORMAL;

unsigned long lastIntersectionTime = 0;
unsigned long finishLineStartTime = 0;
bool finishLineDetected = false;
bool isTurning = false;

void setup() {
  pinMode(IR1, INPUT); pinMode(IR2, INPUT); pinMode(IR3, INPUT);
  pinMode(IR4, INPUT); pinMode(IR5, INPUT);

  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  pinMode(EN1, OUTPUT); pinMode(EN2, OUTPUT);

  Serial.begin(9600);
}

void loop() {
  int s[5] = {
    !digitalRead(IR1),  // Leftmost
    !digitalRead(IR2),  // Left mid
    !digitalRead(IR3),  // Center
    !digitalRead(IR4),  // Right mid
    !digitalRead(IR5)   // Rightmost
  };

  // === Finish Line Detection ===
  if (s[0] && s[1] && s[2] && s[3] && s[4]) {
    if (finishLineStartTime == 0) {
      finishLineStartTime = millis();  // Start timing
    } else if (millis() - finishLineStartTime >= 2000 && !finishLineDetected) {
      finishLineDetected = true;
      currentState = FINISH;
    }
  } else {
    finishLineStartTime = 0;  // Reset timer if condition breaks
  }

  // === Determine Track State ===
  if (!finishLineDetected) {
    if (s[0] && s[4] && millis() - lastIntersectionTime > 500) {
      currentState = T_SECTION;
      lastIntersectionTime = millis();
    } 
    else if ((s[0] || s[1]) && !s[2] && !s[3] && !s[4]) {
      currentState = L_TURN_LEFT;
    } 
    else if ((s[3] || s[4]) && !s[0] && !s[1] && !s[2]) {
      currentState = L_TURN_RIGHT;
    } 
    else if (s[1] || s[3]) {
      currentState = CURVE;
    } 
    else if (s[2]) {
      currentState = NORMAL;
    } 
    else {
      currentState = LOST;
    }
  }

  // === Execute State Action ===
  switch (currentState) {
    case T_SECTION:
      handleTSection(s);
      break;
    case L_TURN_LEFT:
      followLeftL(s);
      break;
    case L_TURN_RIGHT:
      followRightL(s);
      break;
    case CURVE:
      followCurve(s[1], s[3]);
      break;
    case NORMAL:
      forward();
      break;
    case LOST:
      searchLine();
      break;
    case FINISH:
      stopAtFinish();
      break;
  }
}

void forward() {
  analogWrite(EN1, baseSpeed);
  analogWrite(EN2, baseSpeed);
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void followLeftL(int s[5]) {
  if (!isTurning) {
    isTurning = true;
    
    analogWrite(EN1, sharpTurnSpeed);
    analogWrite(EN2, sharpTurnSpeed);
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  // Left reverse
    digitalWrite(IN3, LOW); digitalWrite(IN4, HIGH);  // Right forward

    unsigned long start = millis();
    while (millis() - start < 1000) {
      if (!digitalRead(IR3)) break;
    }
    
    forward();
    delay(100);
    isTurning = false;
  }
}

void followRightL(int s[5]) {
  if (!isTurning) {
    isTurning = true;
    
    analogWrite(EN1, sharpTurnSpeed);
    analogWrite(EN2, sharpTurnSpeed);
    digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);  // Left forward
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  // Right reverse

    unsigned long start = millis();
    while (millis() - start < 1000) {
      if (!digitalRead(IR3)) break;
    }
    
    forward();
    delay(100);
    isTurning = false;
  }
}

void handleTSection(int s[5]) {
  // Default right turn
  followRightL(s);
  delay(300);
}

void followCurve(bool leftActive, bool rightActive) {
  int speedLeft = curveSpeed;
  int speedRight = curveSpeed;

  if (leftActive)  speedRight -= 30;
  if (rightActive) speedLeft -= 30;

  analogWrite(EN1, speedLeft);
  analogWrite(EN2, speedRight);
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

// ✅ Improved Search Line function
void searchLine() {
  static unsigned long spinStartTime = 0;
  static bool spinLeft = true;

  int s[5] = {
    !digitalRead(IR1),
    !digitalRead(IR2),
    !digitalRead(IR3),
    !digitalRead(IR4),
    !digitalRead(IR5)
  };

  // ✅ If line is found → exit search mode
  if (s[0] || s[1] || s[2] || s[3] || s[4]) {
    forward();
    spinStartTime = 0;   // reset timer
    return;
  }

  // === Line not found → continue spinning ===
  if (spinStartTime == 0) {
    spinStartTime = millis();  // start timing
  }

  if (spinLeft) {
    // spin left
    analogWrite(EN1, 100);
    analogWrite(EN2, 80);
    digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH);  // Left backward
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);   // Right forward
  } else {
    // spin right
    analogWrite(EN1, 100);
    analogWrite(EN2, 80);
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);   // Left forward
    digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);  // Right backward
  }

  // After 1.5s → change direction
  if (millis() - spinStartTime >= 1500) {
    spinLeft = !spinLeft;       // flip direction
    spinStartTime = millis();   // reset timer
  }
}

void stopAtFinish() {
  analogWrite(EN1, 0);
  analogWrite(EN2, 0);
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);

  Serial.println("Finish line detected. Robot stopped.");

  while (true);  // Stop permanently
}
