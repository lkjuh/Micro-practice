/*
 ==============================================================================
                          SƠ ĐỒ HƯỚNG DẪN NỐI CHÂN
 ==============================================================================
 1. MÀN HÌNH LCD 1602 (Chế độ 4-bit):
    - LCD VSS  --> GND (Đất)
    - LCD VDD  --> 5V  (Nguồn)
    - LCD V0   --> Chân giữa Biến trở 10K (2 chân bên nối 5V và GND)
    - LCD RS   --> Pin 12 (Arduino)
    - LCD RW   --> GND (Bắt buộc nối đất)
    - LCD E    --> Pin 11 (Arduino)
    - LCD D4   --> Pin 4  (Arduino)
    - LCD D5   --> Pin 5  (Arduino)
    - LCD D6   --> Pin 6  (Arduino)
    - LCD D7   --> Pin 7  (Arduino)
    - LCD A    --> 5V  (Anode LED nền)
    - LCD K    --> GND (Cathode LED nền)

 2. NÚT BẤM (Cấu hình INPUT_PULLUP - Nối 1 chân vào Pin Arduino, 1 chân vào GND):
    - BTN_UP    (Lên)     --> Pin A0
    - BTN_DOWN  (Xuống)   --> Pin A1
    - BTN_LEFT  (Trái)    --> Pin A2
    - BTN_RIGHT (Phải)    --> Pin A3
    - BTN_OK    (Chọn/Lưu)--> Pin A4
    - BTN_BACK  (Quay lại)--> Pin A5

 3. CÒI BÁO (BUZZER):
    - Buzzer (+) --> Pin D3 (Arduino)
    - Buzzer (-) --> GND
 ==============================================================================
*/

#include <Arduino.h>
#include <LiquidCrystal.h>
#include <EEPROM.h>

// ==========================================================
// CẤU HÌNH KIỂU HIỂN THỊ MENU (THAY ĐỔI SỐ Ở ĐÂY)
// 1: Mũi tên đơn       ( > Registering )
// 2: Khối vuông đen    ( ■ Registering )
// 3: Bao ngoặc vuông   ( [Registering] )
// 4: Mũi tên 2 bên     ( > Registering < )
// 5: Mũi tên dài       ( -> Registering )
// ==========================================================
#define MENU_STYLE 1  // <--- THAY ĐỔI TỪ 1 ĐẾN 5 TẠI ĐÂY

// Cấu hình chân LCD: RS=12, E=11, D4=4, D5=5, D6=6, D7=7
LiquidCrystal lcd(12, 11, 4, 5, 6, 7);

// Cấu hình chân nút bấm
#define BTN_UP    A0
#define BTN_DOWN  A1
#define BTN_LEFT  A2
#define BTN_RIGHT A3
#define BTN_OK    A4
#define BTN_BACK  A5

// Cấu hình chân Còi
#define BUZZER_PIN 3

// Tạo ký tự Khối vuông màu đen (Solid Block)
byte solidBlock[8] = {
  B11111,
  B11111,
  B11111,
  B11111,
  B11111,
  B11111,
  B11111,
  B11111
};

// Các trạng thái của hệ thống
enum State {
  STATE_MENU,
  STATE_REG_ID,
  STATE_REG_CODE,
  STATE_ENTER_ID,
  STATE_SHOW_RESULT
};

State currentState = STATE_MENU;

int menuOption = 0;               // 0: Registering, 1: Enter ID
int idDigits[2] = {0, 0};         // ID gồm 2 chữ số (00 - 99)
int codeDigits[4] = {0, 0, 0, 0}; // Code gồm 4 chữ số (0000 - 9999)
int currentDigitPos = 0;          // Vị trí chữ số đang chỉnh sửa

unsigned long lastActivityTime = 0; // Đếm thời gian cho đếm ngược 5s

// ==========================================================
// KHAI BÁO NGUYÊN MẪU HÀM
// ==========================================================
bool isPressed(int pin);
void beep();
void checkTimeout();
void resetToMenu();
void printMenuItem(int row, const char* text, bool isSelected);
void showMenu();
void showIDInput(const char* title);
void showCodeInput();
void handleUp();
void handleDown();
void handleLeft();
void handleRight();
void handleOK();
void handleBack();

// ==========================================================
// CHƯƠNG TRÌNH CHÍNH
// ==========================================================
void setup() {
  lcd.begin(16, 2);
  
  // Nạp ký tự khối vuông đen vào CGRAM vị trí 0
  lcd.createChar(0, solidBlock);

  // Cấu hình chân Còi
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  // Cấu hình các chân nút bấm
  pinMode(BTN_UP, INPUT_PULLUP);
  pinMode(BTN_DOWN, INPUT_PULLUP);
  pinMode(BTN_LEFT, INPUT_PULLUP);
  pinMode(BTN_RIGHT, INPUT_PULLUP);
  pinMode(BTN_OK, INPUT_PULLUP);
  pinMode(BTN_BACK, INPUT_PULLUP);

  showMenu();
}

void loop() {
  checkTimeout(); // Kiểm tra quá 5s không bấm nút

  if (isPressed(BTN_UP)) {
    beep();
    lastActivityTime = millis();
    handleUp();
  }
  if (isPressed(BTN_DOWN)) {
    beep();
    lastActivityTime = millis();
    handleDown();
  }
  if (isPressed(BTN_LEFT)) {
    beep();
    lastActivityTime = millis();
    handleLeft();
  }
  if (isPressed(BTN_RIGHT)) {
    beep();
    lastActivityTime = millis();
    handleRight();
  }
  if (isPressed(BTN_OK)) {
    beep();
    lastActivityTime = millis();
    handleOK();
  }
  if (isPressed(BTN_BACK)) {
    beep();
    lastActivityTime = millis();
    handleBack();
  }
}

// Hàm phát tiếng tít ngắn khi nhấn nút
void beep() {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(50); // Độ dài tiếng kêu (50ms)
  digitalWrite(BUZZER_PIN, LOW);
}

// ==========================================================
// HÀM XỬ LÝ IN MỤC MENU DỰA TRÊN MENU_STYLE
// ==========================================================
void printMenuItem(int row, const char* text, bool isSelected) {
  lcd.setCursor(0, row);

  if (isSelected) {
    #if MENU_STYLE == 1
      lcd.print("> ");
      lcd.print(text);
      lcd.print(" ");
    #elif MENU_STYLE == 2
      lcd.write(byte(0)); 
      lcd.print(" ");
      lcd.print(text);
      lcd.print(" ");
    #elif MENU_STYLE == 3
      lcd.print("[");
      lcd.print(text);
      lcd.print("]");
    #elif MENU_STYLE == 4
      lcd.print("> ");
      lcd.print(text);
      lcd.print(" <");
    #elif MENU_STYLE == 5
      lcd.print("-> ");
      lcd.print(text);
    #endif
  } else {
    #if MENU_STYLE == 1 || MENU_STYLE == 2
      lcd.print("  ");
      lcd.print(text);
      lcd.print(" ");
    #elif MENU_STYLE == 3
      lcd.print(" ");
      lcd.print(text);
      lcd.print(" ");
    #elif MENU_STYLE == 4
      lcd.print("  ");
      lcd.print(text);
      lcd.print("  ");
    #elif MENU_STYLE == 5
      lcd.print("   ");
      lcd.print(text);
    #endif
  }
}

// Hiển thị Menu chính
void showMenu() {
  lcd.clear();
  printMenuItem(0, "Registering", menuOption == 0);
  printMenuItem(1, "Enter ID",    menuOption == 1);
}

// ==========================================================
// CÁC HÀM XỬ LÝ HỆ THỐNG
// ==========================================================
bool isPressed(int pin) {
  if (digitalRead(pin) == LOW) {
    delay(50); // Debounce
    if (digitalRead(pin) == LOW) {
      while (digitalRead(pin) == LOW); // Chờ nhả phím
      return true;
    }
  }
  return false;
}

void checkTimeout() {
  if (currentState == STATE_SHOW_RESULT || currentState == STATE_ENTER_ID || currentState == STATE_REG_ID) {
    if (millis() - lastActivityTime > 5000) {
      resetToMenu();
    }
  }
}

void resetToMenu() {
  currentState = STATE_MENU;
  showMenu();
}

void showIDInput(const char* title) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(title);
  lcd.setCursor(0, 1);
  lcd.print("ID: ");
  for (int i = 0; i < 2; i++) {
    lcd.print(idDigits[i]);
  }
  
  lcd.setCursor(4 + currentDigitPos, 1);
  lcd.cursor();
}

void showCodeInput() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Set Code (4-dig):");
  lcd.setCursor(0, 1);
  lcd.print("Code: ");
  for (int i = 0; i < 4; i++) {
    lcd.print(codeDigits[i]);
  }
  lcd.setCursor(6 + currentDigitPos, 1);
  lcd.cursor();
}

void handleUp() {
  if (currentState == STATE_MENU) {
    menuOption = 0;
    showMenu();
  } else if (currentState == STATE_REG_ID || currentState == STATE_ENTER_ID) {
    idDigits[currentDigitPos] = (idDigits[currentDigitPos] + 1) % 10;
    showIDInput(currentState == STATE_REG_ID ? "Reg - Enter ID:" : "Check - Enter ID:");
  } else if (currentState == STATE_REG_CODE) {
    codeDigits[currentDigitPos] = (codeDigits[currentDigitPos] + 1) % 10;
    showCodeInput();
  }
}

void handleDown() {
  if (currentState == STATE_MENU) {
    menuOption = 1;
    showMenu();
  } else if (currentState == STATE_REG_ID || currentState == STATE_ENTER_ID) {
    idDigits[currentDigitPos] = (idDigits[currentDigitPos] + 9) % 10;
    showIDInput(currentState == STATE_REG_ID ? "Reg - Enter ID:" : "Check - Enter ID:");
  } else if (currentState == STATE_REG_CODE) {
    codeDigits[currentDigitPos] = (codeDigits[currentDigitPos] + 9) % 10;
    showCodeInput();
  }
}

void handleLeft() {
  if (currentState == STATE_REG_ID || currentState == STATE_ENTER_ID) {
    if (currentDigitPos > 0) currentDigitPos--;
    showIDInput(currentState == STATE_REG_ID ? "Reg - Enter ID:" : "Check - Enter ID:");
  } else if (currentState == STATE_REG_CODE) {
    if (currentDigitPos > 0) currentDigitPos--;
    showCodeInput();
  }
}

void handleRight() {
  if (currentState == STATE_REG_ID || currentState == STATE_ENTER_ID) {
    if (currentDigitPos < 1) currentDigitPos++; // Giới hạn 2 số ID (vị trí 0 và 1)
    showIDInput(currentState == STATE_REG_ID ? "Reg - Enter ID:" : "Check - Enter ID:");
  } else if (currentState == STATE_REG_CODE) {
    if (currentDigitPos < 3) currentDigitPos++; // Giới hạn 4 số Code (vị trí 0, 1, 2, 3)
    showCodeInput();
  }
}

void handleOK() {
  lcd.noCursor();
  
  if (currentState == STATE_MENU) {
    for (int i = 0; i < 2; i++) idDigits[i] = 0;
    currentDigitPos = 0;
    
    if (menuOption == 0) {
      currentState = STATE_REG_ID;
      showIDInput("Reg - Enter ID:");
    } else {
      currentState = STATE_ENTER_ID;
      showIDInput("Check - Enter ID:");
    }
  } 
  else if (currentState == STATE_REG_ID) {
    currentState = STATE_REG_CODE;
    for (int i = 0; i < 4; i++) codeDigits[i] = 0;
    currentDigitPos = 0;
    showCodeInput();
  } 
  else if (currentState == STATE_REG_CODE) {
    int idVal = idDigits[0]*10 + idDigits[1];
    int codeVal = codeDigits[0]*1000 + codeDigits[1]*100 + codeDigits[2]*10 + codeDigits[3];
    
    int eepromAddr = idVal * sizeof(int);
    EEPROM.put(eepromAddr, codeVal);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Saved!");
    delay(1500);
    resetToMenu();
  } 
  else if (currentState == STATE_ENTER_ID) {
    int idVal = idDigits[0]*10 + idDigits[1]; // Đã sửa: Tính ID 2 chữ số
    int eepromAddr = idVal * sizeof(int);
    int storedCode = -1;
    EEPROM.get(eepromAddr, storedCode);

    currentState = STATE_SHOW_RESULT;
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("ID: ");
    for (int i = 0; i < 2; i++) lcd.print(idDigits[i]); // Đã sửa: In đúng 2 số ID
    
    lcd.setCursor(0, 1);
    if (storedCode < 0 || storedCode > 9999) {
      lcd.print("Not Found!");
    } else {
      lcd.print("Code: ");
      char formatBuf[5];
      sprintf(formatBuf, "%04d", storedCode); // Định dạng in 4 chữ số Code
      lcd.print(formatBuf);
    }
  }
}

void handleBack() {
  lcd.noCursor();
  resetToMenu();
}