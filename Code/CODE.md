const int RPWM = 5;     // PWM chiều thuận
const int LPWM = 6;     // PWM chiều nghịch
const int R_EN = 7;     // Enable chiều thuận
const int L_EN = 8;     // Enable chiều nghịch
const int POT_SPEED = A0; // Biến trở tốc độ
const int BTN_FWD = 3;  // Nút tiến 
const int BTN_REV = 11; // Nút lùi
const int MODE_SW = 12; // Công tắc chế độ
const int BTN_STOP = 10; // Nút dừng

// Biến toàn cục
int motorSpeed = 0;
const int ANGLE_X = 90; // Góc quay cố định 90°
enum {MANUAL, AUTO, STOPPED} mode = MANUAL;
enum {IDLE, LEFT_OUT, BACK_CENTER, RIGHT_OUT, RETURN_CENTER} state = IDLE;
unsigned long actionStartTime = 0;
unsigned long rotationDuration = 0;
const unsigned long PAUSE_TIME = 500; // 500ms dừng
const float DEGREES_PER_SECOND = 90.0; // 90 độ/giây

void setup() {
  Serial.begin(9600);
  
  // Thiết lập chân điều khiển motor (GIỮ NGUYÊN)
  pinMode(RPWM, OUTPUT);
  pinMode(LPWM, OUTPUT);
  pinMode(R_EN, OUTPUT);
  pinMode(L_EN, OUTPUT);
  digitalWrite(R_EN, HIGH);
  digitalWrite(L_EN, HIGH);
  
  // Thiết lập nút nhấn (GIỮ NGUYÊN)
  pinMode(BTN_FWD, INPUT_PULLUP);
  pinMode(BTN_REV, INPUT_PULLUP);
  pinMode(MODE_SW, INPUT_PULLUP);
  pinMode(BTN_STOP, INPUT_PULLUP);
  pinMode(POT_SPEED, INPUT);
  
  Serial.println("Hệ thống điều khiển động cơ");
  Serial.println("Chế độ tự động: Chu trình quay ±90°");
}

void loop() {
  // Đọc giá trị biến trở tốc độ (GIỮ NGUYÊN)
  motorSpeed = map(analogRead(POT_SPEED), 0, 1023, 30, 255);
  
  // Tính thời gian quay cho 90°
  rotationDuration = (ANGLE_X * 1000) / (DEGREES_PER_SECOND * (motorSpeed/255.0));

  // Xử lý nút dừng (GIỮ NGUYÊN)
  if (digitalRead(BTN_STOP) == LOW) {
    motorStop();
    mode = STOPPED;
    state = IDLE;
    Serial.println("Đã DỪNG CHU TRÌNH");
    delay(100);
    return;
  }

  // Kiểm tra chế độ hoạt động (GIỮ NGUYÊN)
  if (digitalRead(MODE_SW) == LOW && mode != AUTO) {
    mode = AUTO;
    state = LEFT_OUT;
    actionStartTime = millis();
    Serial.println("Bắt đầu chế độ tự động (góc quay 90°)");
  } else if (digitalRead(MODE_SW) == HIGH && mode != MANUAL) {
    mode = MANUAL;
    motorStop();
    Serial.println("Chuyển sang chế độ thủ công");
  }

  // Xử lý theo chế độ (GIỮ NGUYÊN)
  if (mode == AUTO) {
    handleAutoMode();
  } else if (mode == MANUAL) {
    handleManualMode();
  }
  
  delay(10);
}

void handleAutoMode() {
  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - actionStartTime;

  switch (state) {
    case LEFT_OUT:
      motorReverse();
      if (elapsed >= rotationDuration) {
        motorStop();
        state = BACK_CENTER;
        actionStartTime = currentTime;
        Serial.println("Đã quay trái 90°, đang dừng...");
      }
      break;
      
    case BACK_CENTER:
      if (elapsed >= PAUSE_TIME) {
        motorForward();
        state = RIGHT_OUT;
        actionStartTime = currentTime;
        Serial.println("Bắt đầu quay về trung tâm");
      }
      break;
      
    case RIGHT_OUT:
      motorForward();
      if (elapsed >= rotationDuration) {
        motorStop();
        state = RETURN_CENTER;
        actionStartTime = currentTime;
        Serial.println("Đã quay phải 90°, đang dừng...");
      }
      break;
      
    case RETURN_CENTER:
      if (elapsed >= PAUSE_TIME) {
        motorReverse();
        state = LEFT_OUT;
        actionStartTime = currentTime;
        Serial.println("Bắt đầu quay về trung tâm (lặp chu trình)");
      }
      break;
      
    case IDLE:
      motorStop();
      break;
  }
}

// GIỮ NGUYÊN hàm điều khiển thủ công
void handleManualMode() {
  if (digitalRead(BTN_FWD) == LOW) {
    motorForward();
  } else if (digitalRead(BTN_REV) == LOW) {
    motorReverse();
  } else {
    motorStop();
  }
}

// GIỮ NGUYÊN hàm điều khiển motor
void motorForward() {
  analogWrite(RPWM, motorSpeed);
  analogWrite(LPWM, 0);
}

void motorReverse() {
  analogWrite(RPWM, 0);
  analogWrite(LPWM, motorSpeed);
}

void motorStop() {
  analogWrite(RPWM, 0);
  analogWrite(LPWM, 0);
}
 
