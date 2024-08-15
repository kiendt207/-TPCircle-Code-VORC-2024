#include "Adafruit_TCS34725.h"       //Thư viện TCS34725
#include <Adafruit_PWMServoDriver.h> // thư viện PCA9685
#include <ESP32Servo.h>              // Thư viện ESP32Servo
#include <PS2X_lib.h>                // Thư viện PS2X
#include <Wire.h>

Adafruit_TCS34725 tcs =
    Adafruit_TCS34725(TCS34725_INTERGRATIONTIME_50MS, TCS34725_GAIN_1X);

#define PS2_DAT 12
#define PS2_CMD 13
#define PS2_SEL 15
#define PS2_CLK 14

#define MOTOR1_PIN 4  // Motor điều khiển bánh xe 1
#define MOTOR2_PIN 5  // Motor điều khiển bánh xe 2
#define MOTOR3_PIN 14 // Motor thu bóng
#define MOTOR4_PIN 13 // Motor linear slide

#define SERVO1_PIN 12 // Servo 180 điều khiển cửa xả bóng đen
#define SERVO2_PIN 15 // Servo 180 điều khiển hệ thống phân loại
#define SERVO3_PIN 11 // Servo 360 điều khiển linear slide bóng đen
#define SERVO4_PIN 10 // Servo 360 điều khiển cửa xả bóng trắng

#define MOTOR1_POWER 75  // Công suất Motor 1 (0-100%)
#define MOTOR2_POWER 75  // Công suất Motor 2 (0-100%)
#define MOTOR3_POWER 25  // Công suất Motor 3 (0-100%)
#define MOTOR4_[POWER 75 // Công suất Motor 4 (0-100%)

PS2X ps2x;
Servo motor1; // Đối tượng Motor DC 300 RPM
Servo motor2; // Đối tượng Motor DC 300 RPM
Servo motor3; // Đối tượng Motor DC 180 RPM
Servo motor4; // Đối tượng Motor DC 1500 RPM
Servo servo_1; // Đối tượng servo 1 dùng cho linear slide của chất thải
Servo servo_2; // Đối tượng servo 2 dùng cho cửa xả của chất thải
Servo servo_3; // Đối tượng servo 3 dùng cho cửa xả khoang nước
Servo servo_4; // Đối tượng servo 4 dùng cho phân loại bóng đen và trắng

int vt = 0; // vị trí của servo từ 0-180 độ

bool button1Pressed = false; // Cờ báo hiệu nút 1 được nhấn
bool button2Pressed = false; // Cờ báo hiệu nút 2 được nhấn
bool button3Pressed = false; // Cờ báo hiệu nút 3 được nhấn
bool button4Pressed = false; // Cờ báo hiệu nút 4 được nhấn
bool button5Pressed = false; // Cờ báo hiệu nút 5 được nhấn
bool button6Pressed = false; // Cờ báo hiệu nút 6 được nhấn
bool isMovingFoward_1 = false; // Cờ theo dõi trạng thái di chuyển của servo 1
bool isMovingFoward_2 = false; // Cờ theo dõi trạng thái di chuyển của servo 2
bool isMovingFoward_3 = false; // Cờ theo dõi trạng thái di chuyển của servo 3
bool isRunning_4 = false; // Cờ theo dõi trạng thái di chuyển của servo 4
bool motor3Running = false; // Cờ báo hiệu motor 3 đang chạy
bool motor4Running = false; // Cờ báo hiệu motor 4 đang chạy

void setup() {
  Serial.begin(9600);

  // Khởi tạo PS2X
  ps2x.config_gamepad(PS2_CLK, PS2_CMD, PS2_SEL, PS2_DAT, false, false);

  // Khởi tạo Motor DC
  motor1.attach(MOTOR1_PIN);
  motor2.attach(MOTOR2_PIN);
  motor3.attach(MOTOR3_PIN);
  motor4.attach(MOTOR4_PIN);

  // Khởi tạo động cơ Servo
  servo_1.attach(SERVO1_PIN);
  servo_2.attach(SERVO2_PIN);
  servo_3.attach(SERVO3_PIN);
  servo_4.attach(SERVO4_PIN);

  stopMotors(); // Dừng tất cả các motor
}

void loop() {
  // Đọc dữ liệu từ tay cầm PS2
  ps2x.read_gamepad(false, false);

  // Điều khiển robot
  controlMotors();

  // khởi tạo các giá trị cảm biến màu
  uint16_t r, g, b, c;
  tcs.getRawData(&r, &g, &b, &c);

  // Xử lý nút nhấn
  if (ps2x.ButtonPressed(PSB_PAD_LEFT) && !button1Pressed) {
    button1Pressed = true;
    if (!motor3Running) {
      motor3Running = true;
      motor3.write(180); // Quay motor 3 ngược chiều kim đồng hồ
    } else {
      motor3Running = false;
      motor3.write(90); // Dừng motor 3
    }
  } else if (ps2x.ButtonReleased(PSB_PAD_LEFT)) {
    button1Pressed = false;
  }

  if (ps2x.ButtonPressed(PSB_PAD_UP) && !button2Pressed) {
    button2Pressed = true;
    if (!motor4Running) {
      motor4Running = true;
      rotateMotor4Clockwise(); // Quay motor 4 theo chiều kim đồng hồ 4 vòng
    } else {
      motor4Running = false;
      stopMotor4(); // Dừng motor 4
    }
  } else if (ps2x.ButtonReleased(PSB_PAD_UP)) {
    button2Pressed = false;
  }

  if (button3Pressed) {
    // kiểm tra nút đã được ấn hay chưa
    if (ps2x.ButtonPressed(PSB_R1)) {
      button3Pressed = true;
      isMovingFoward_1 = !isMovingFoward_1; // đảo ngược trạng thái di chuyển
      if (isMovingFoward_1) {
        for (vt = 0; vt <= 90; vt++)
          // điều khiển servo đến vị trí vt theo chiều kim đồng hồ
          // servo được đưa đến góc 90 độ để mở cửa xả bóng đen
          servo_1.write(vt);
        delay(12); // đợi 12ms để servo di chuyển đến vị trí
      }
    } else {
      for (vt = 90; vt >= 0; vt--) {
        // điều khiển servo đến vị trí vt
        servo_1.write(vt);
        delay(12); // đợi 12ms để servo di chuyển đến vị trí
      }
    }
  }

  servo_2.write(90);
  if (button4Pressed) {
    // kiểm tra nút đã được ấn hay chưa
    if (ps2x.ButtonPressed(PSB_CROSS)) {
      button4Pressed = true;
      if (r < 90 && g < 90 && b < 90) {
        isMovingFoward_2 = !isMovingFoward_2; // đảo ngược trạng thái di chuyển
        if (isMovingFoward_2) {
          for (vt = 90; vt >= 0; vt--)
            servo_2.write(vt);
          delay(12); // đợi 12ms để servo di chuyển đến vị trí
        }
      } else if (r > 180 && g > 180 && b > 180) {
        isMovingFoward_2 = !isMovingFoward_2; // đảo ngược trạng thái di chuyển
        if (isMovingFoward_2) {
          for (vt = 90; vt >= 0; vt--)
            servo_2.write(vt);
          delay(12); // đợi 12ms để servo di chuyển đến vị trí
        }
      }
    }
  }

  // Điều khiển linear slide
  if (button5Pressed) {
    // kiểm tra nút đã được ấn hay chưa
    if (ps2x.ButtonPressed(PSB_R1)) {
      button5Pressed = true;
      servo_3.write(0); // cho servo chạy theo chiều kim đh
    } else if (ps2x.ButtonReleased(PSB_R1)) {
      button5Pressed = false;
      servo_3.write(90); // cho servo dừng
    }
  }

  if (button6Pressed) {
    // kiểm tra nút đã được ấn hay chưa
    //  điều khiển cửa xả bóng trắng
    if (ps2x.ButtonPressed(PSB_L1)) {
      button6Pressed = true servo_4.write(180);
    } else if (ps2x.ButtonReleased(PSB_L1)) {
      button6Pressed = false;
      servo_4.write(90); // Dừng servo
    }
  }

  void controlMotors() {
    int stickX = ps2x.Analog(PSS_RX);
    int stickY = ps2x.Analog(PSS_RY);

    // Điều khiển Motor 1 và Motor 2
    if (stickY > 150) {
      motor1.writeMicroseconds(map(stickY, 150, 255, 1000, 2000));
      motor2.writeMicroseconds(map(stickY, 150, 255, 1000, 2000));
    } else if (stickY < 105) {
      motor1.writeMicroseconds(map(stickY, 105, 0, 2000, 1000));
      motor2.writeMicroseconds(map(stickY, 105, 0, 2000, 1000));
    } else {
      motor1.writeMicroseconds(1500); // Dừng motor 1
      motor2.writeMicroseconds(1500); // Dừng motor 2
    }
  }

  void rotateMotor4Clockwise() {
    unsigned long startTime = millis();
    while ((millis() - startTime) <= 8000) {
      motor4.write(0); // Quay motor 4 theo chiều kim đồng hồ
      delay(20);
    }
    stopMotor4(); // Dừng motor 4
  }

  void stopMotor4() {
    motor4.write(90); // Dừng motor 4
  }

  void stopMotors() {
    motor1.writeMicroseconds(1500); // Dừng motor 1
    motor2.writeMicroseconds(1500); // Dừng motor 2
    motor3.write(90);               // Dừng motor 3
    motor4.write(90);               // Dừng motor 4
  }

