#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <SPI.h>
#include <MFRC522.h>
#include <Adafruit_Fingerprint.h>
#include <ESP32Servo.h>
#include <Keypad.h>
#include <EEPROM.h>

// -------------------------------------------------------------------------
// CẤU HÌNH HỆ THỐNG & BỘ NHỚ FLASH
// -------------------------------------------------------------------------
#define EEPROM_SIZE 512
#define MAX_USERS 10

const int ADDR_ADMIN_PIN = 0;   
const int ADDR_USER_PIN  = 4;   
const int ADDR_RFID_COUNT = 8;  
const int ADDR_RFID_DATA  = 10; 

// -------------------------------------------------------------------------
// CẤU HÌNH NGOẠI VI & CHÂN CẮM
// -------------------------------------------------------------------------
LiquidCrystal_I2C lcd(0x27, 16, 2);

#define RST_PIN 4  
#define SS_PIN 5   
MFRC522 mfrc522(SS_PIN, RST_PIN);

HardwareSerial mySerial(2); 
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);

#define SERVO_PIN 2 
Servo doorServo;

const byte ROWS = 4; 
const byte COLS = 4; 
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {32, 33, 25, 26}; 
byte colPins[COLS] = {27, 14, 12, 13}; 
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// -------------------------------------------------------------------------
// BIẾN TOÀN CỤC & TRẠNG THÁI
// -------------------------------------------------------------------------
String adminPin = "1234"; 
String userPin  = "0000";
int wrongAttempts = 0;
unsigned long lockUntil = 0;
unsigned long penaltyTime = 30000; 

struct RfidCard {
  byte uid[4];
};
RfidCard registeredCards[MAX_USERS];
int rfidCount = 0;

// Khai báo cấu trúc hàm mới (Trả về int để xử lý Quay lại / Thoát)
void mainMenu();
void handleAdminMenu(char menuType);
int enrollFingerprint();
int deleteFingerprint();
int enrollRFID();
int deleteRFID();
int changePins(char mode);
String inputSecret(String prompt);
bool checkAdminAuth();
void openDoor();
void verifySuccess();
void verifyFail();
void saveSystemData();
void loadSystemData();

// -------------------------------------------------------------------------
// SETUP
// -------------------------------------------------------------------------
void setup() {
  Serial.begin(115200);
  EEPROM.begin(EEPROM_SIZE);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("He thong dang");
  lcd.setCursor(0, 1);
  lcd.print("khoi dong...");

  ESP32PWM::allocateTimer(0);
  ESP32PWM::allocateTimer(1);
  ESP32PWM::allocateTimer(2);
  ESP32PWM::allocateTimer(3);
  doorServo.setPeriodHertz(50);
  doorServo.attach(SERVO_PIN, 500, 2400);
  doorServo.write(0); 

  SPI.begin(18, 19, 23, 5); 
  mfrc522.PCD_Init();

  mySerial.begin(57600, SERIAL_8N1, 16, 17); 
  finger.begin(57600);
  
  if (finger.verifyPassword()) {
    Serial.println("AS608: OK");
  } else {
    Serial.println("AS608: LOI KET NOI!");
    lcd.clear();
    lcd.print("Loi: Cam bien VT");
    delay(2000);
  }

  loadSystemData();

  keypad.setDebounceTime(80); 
  keypad.setHoldTime(500);    

  lcd.clear();
  mainMenu();
}

// -------------------------------------------------------------------------
// VÒNG LẶP CHÍNH (LOOP)
// -------------------------------------------------------------------------
void loop() {
  if (lockUntil > 0) {
    if (millis() < lockUntil) {
      long timeLeft = (lockUntil - millis()) / 1000;
      lcd.setCursor(0, 0);
      lcd.print("He thong bi KHOA");
      lcd.setCursor(0, 1);
      lcd.print("Con lai: " + String(timeLeft) + "s   ");
      delay(500);
      return;
    } else {
      lockUntil = 0;
      mainMenu();
    }
  }

  char key = keypad.getKey();
  if (key && keypad.getState() == PRESSED) {
    if (key == 'A' || key == 'B' || key == 'C') {
      handleAdminMenu(key);
    } 
    else if (key >= '0' && key <= '9') {
      String entered = "";
      entered += key;
      lcd.clear();
      lcd.print("Nhap Ma PIN:");
      lcd.setCursor(0, 1);
      lcd.print("*");
      
      bool inputDone = false;
      while (!inputDone) {
        char nextKey = keypad.waitForKey();
        if (nextKey >= '0' && nextKey <= '9') {
          if (entered.length() < 4) {
            entered += nextKey;
            lcd.setCursor(entered.length() - 1, 1);
            lcd.print("*");
          }
        } 
        else if (nextKey == 'D') { 
          if (entered.length() > 0) {
            entered.remove(entered.length() - 1);
            lcd.setCursor(entered.length(), 1);
            lcd.print(" ");
          }
          if (entered.length() == 0) { 
            mainMenu();
            return;
          }
        } 
        else if (nextKey == '*') { 
          mainMenu();
          return;
        }
        else if (nextKey == '#') {
          if (entered.length() == 4) {
            if (entered == adminPin || entered == userPin) {
              verifySuccess();
              openDoor();
            } else {
              verifyFail();
            }
            inputDone = true;
          }
        }
      }
    } 
    else if (key == '*') {
      mainMenu();
    }
  }

  if (finger.getImage() == FINGERPRINT_OK) {
    if (finger.image2Tz() == FINGERPRINT_OK) {
      if (finger.fingerSearch() == FINGERPRINT_OK) {
        verifySuccess();
        openDoor();
      } else {
        verifyFail();
      }
    }
    delay(1000); 
  }

  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    bool accessGranted = false;
    for (int i = 0; i < rfidCount; i++) {
      if (memcmp(mfrc522.uid.uidByte, registeredCards[i].uid, 4) == 0) {
        accessGranted = true;
        break;
      }
    }
    mfrc522.PICC_HaltA();
    
    if (accessGranted) {
      verifySuccess();
      openDoor();
    } else {
      verifyFail();
    }
  }
}

// -------------------------------------------------------------------------
// CÁC HÀM CHỨC NĂNG HỆ THỐNG
// -------------------------------------------------------------------------
void mainMenu() {
  lcd.clear();
  lcd.print("San sang!");
  lcd.setCursor(0, 1);
  lcd.print("Quet VT/The/PIN");
}

void verifySuccess() {
  wrongAttempts = 0;
  penaltyTime = 30000; 
  lcd.clear();
  lcd.print("XAC THUC DUNG!");
  delay(1000);
}

void verifyFail() {
  wrongAttempts++;
  lcd.clear();
  lcd.print("SAI XAC THUC!");
  lcd.setCursor(0, 1);
  lcd.print("Sai lan: " + String(wrongAttempts));
  delay(2000);

  if (wrongAttempts >= 3) { 
    lockUntil = millis() + penaltyTime;
    penaltyTime *= 2; 
  }
  mainMenu();
}

void openDoor() {
  lcd.clear();
  lcd.print("CUA DANG MO...");
  doorServo.write(90); 
  delay(5000);       
  doorServo.write(0);  
  mainMenu();
}

// -------------------------------------------------------------------------
// MENU QUẢN TRỊ VIÊN ĐA TẦNG (Cấu trúc hỗ trợ nút Quay lại D)
// -------------------------------------------------------------------------
void handleAdminMenu(char menuType) {
  if (!checkAdminAuth()) {
    mainMenu(); 
    return;
  }

  bool stayInMenu = true;
  while (stayInMenu) { 
    lcd.clear();
    if (menuType == 'A') {
      lcd.print("QL VT: 1-Them");
      lcd.setCursor(0, 1);
      lcd.print("       2-Xoa");
    } 
    else if (menuType == 'B') {
      lcd.print("QL RFID: 1-Them");
      lcd.setCursor(0, 1);
      lcd.print("         2-Xoa");
    } 
    else if (menuType == 'C') {
      lcd.print("1-Doi PIN Admin");
      lcd.setCursor(0, 1);
      lcd.print("2-Doi PIN User");
    }

    char choice = keypad.waitForKey();

    if (choice == '*' || choice == 'D') { 
      stayInMenu = false; 
    } 
    else if (menuType == 'A') {
      if (choice == '1') {
        if (enrollFingerprint() == -1) stayInMenu = false; 
      } else if (choice == '2') {
        if (deleteFingerprint() == -1) stayInMenu = false;
      }
    } 
    else if (menuType == 'B') {
      if (choice == '1') {
        if (enrollRFID() == -1) stayInMenu = false;
      } else if (choice == '2') {
        if (deleteRFID() == -1) stayInMenu = false;
      }
    } 
    else if (menuType == 'C') {
      if (choice == '1' || choice == '2') {
        if (changePins(choice) == -1) stayInMenu = false;
      }
    }
  }
  mainMenu(); 
}

String inputSecret(String prompt) {
  lcd.clear();
  lcd.print(prompt);
  String result = "";
  lcd.setCursor(0, 1);
  
  while (true) {
    char key = keypad.waitForKey();
    if (key >= '0' && key <= '9') {
      if (result.length() < 4) {
        result += key;
        lcd.setCursor(result.length() - 1, 1);
        lcd.print("*");
      }
    } else if (key == 'D') {
      if (result.length() > 0) { 
        result.remove(result.length() - 1);
        lcd.setCursor(result.length(), 1);
        lcd.print(" ");
      } else {
        return "BACK"; 
      }
    } else if (key == '*') {
      return "CANCEL"; 
    } else if (key == '#') {
      if (result.length() > 0) return result; 
    }
  }
}

bool checkAdminAuth() {
  String entered = inputSecret("Nhap PIN Admin:");
  if (entered == "CANCEL" || entered == "BACK") return false; 
  
  if (entered == adminPin) return true;
  lcd.clear();
  lcd.print("Sai PIN Admin!");
  delay(2000);
  return false;
}

// -------------------------------------------------------------------------
// CHỨC NĂNG VÂN TAY 
// -------------------------------------------------------------------------
int enrollFingerprint() {
  lcd.clear();
  lcd.print("Nhap ID (1-299):");
  String idStr = "";
  lcd.setCursor(0, 1);
  
  while (true) {
    char key = keypad.waitForKey();
    
    if (key >= '0' && key <= '9') {
      if (idStr.length() < 3) { 
        idStr += key;
        lcd.setCursor(0, 1);
        lcd.print(idStr + " "); 
      }
    } 
    else if (key == 'D') {
      if (idStr.length() > 0) {
        idStr.remove(idStr.length() - 1); 
        lcd.clear();
        lcd.print("Nhap ID (1-299):");
        lcd.setCursor(0, 1);
        lcd.print(idStr); 
      } else {
        return 1; 
      }
    } 
    else if (key == '#') {
      if (idStr.length() > 0) break;
    } 
    else if (key == '*') {
      return -1; 
    }
  }
  
  int id = idStr.toInt();
  if (id < 1 || id > 299) {
    lcd.clear();
    lcd.print("ID khong hop le!");
    delay(2000);
    return 1; 
  }

  lcd.clear();
  lcd.print("Dat van tay len");
  while (finger.getImage() != FINGERPRINT_OK) {
    char key = keypad.getKey(); 
    if (key == '*') return -1;
    if (key == 'D') return 1;
  }
  if (finger.image2Tz(1) != FINGERPRINT_OK) return 1;

  lcd.clear();
  lcd.print("Nhac tay ra...");
  delay(2000);
  
  lcd.clear();
  lcd.print("Dat lai van tay");
  while (finger.getImage() != FINGERPRINT_OK) {
    char key = keypad.getKey();
    if (key == '*') return -1;
    if (key == 'D') return 1;
  }
  if (finger.image2Tz(2) != FINGERPRINT_OK) return 1;

  if (finger.createModel() == FINGERPRINT_OK) {
    if (finger.storeModel(id) == FINGERPRINT_OK) {
      lcd.clear();
      lcd.print("Luu thanh cong!");
    } else {
      lcd.clear();
      lcd.print("Loi luu tru!");
    }
  } else {
    lcd.clear();
    lcd.print("Khong trung khop!");
  }
  delay(2000);
  return 0; 
}

int deleteFingerprint() {
  lcd.clear();
  lcd.print("1-Xoa 1 ID VT");
  lcd.setCursor(0, 1);
  lcd.print("2-Xoa tat ca");
  
  char choice;
  while (true) {
    choice = keypad.waitForKey();
    if (choice == '*') return -1;
    if (choice == 'D') return 1; 
    if (choice == '1' || choice == '2') break;
  }

  if (choice == '1') {
    lcd.clear();
    lcd.print("Nhap ID can xoa:");
    String idStr = "";
    lcd.setCursor(0, 1);
    
    while (true) {
      char key = keypad.waitForKey();
      if (key >= '0' && key <= '9') {
        if (idStr.length() < 3) {
          idStr += key;
          lcd.setCursor(0, 1);
          lcd.print(idStr + " ");
        }
      } 
      else if (key == 'D') {
        if (idStr.length() > 0) {
          idStr.remove(idStr.length() - 1);
          lcd.clear();
          lcd.print("Nhap ID can xoa:");
          lcd.setCursor(0, 1);
          lcd.print(idStr);
        } else {
          return 1;
        }
      } 
      else if (key == '#') {
        if (idStr.length() > 0) break;
      } 
      else if (key == '*') {
        return -1;
      }
    }
    
    int id = idStr.toInt();
    if (finger.deleteModel(id) == FINGERPRINT_OK) {
      lcd.clear();
      lcd.print("Da xoa ID VT!");
    } else {
      lcd.clear();
      lcd.print("Loi hoac sai ID!");
    }
  } 
  else if (choice == '2') {
    lcd.clear();
    lcd.print("Xoa TAT CA VT?");
    lcd.setCursor(0, 1);
    lcd.print("Bam # xac nhan");

    while (true) {
      char key = keypad.waitForKey();
      if (key == '#') {
        if (finger.emptyDatabase() == FINGERPRINT_OK) {
          lcd.clear();
          lcd.print("Da xoa het VT!");
        } else {
          lcd.clear();
          lcd.print("Loi xoa VT!");
        }
        break; 
      }
      else if (key == 'D') return 1;  
      else if (key == '*') return -1; 
    }
  }
  delay(2000);
  return 0;
}

// -------------------------------------------------------------------------
// CHỨC NĂNG THẺ TỪ RFID
// -------------------------------------------------------------------------
int enrollRFID() {
  lcd.clear();
  lcd.print("Quet the moi...");
  unsigned long startTime = millis();
  
  while (!mfrc522.PICC_IsNewCardPresent() || !mfrc522.PICC_ReadCardSerial()) {
    char key = keypad.getKey();
    if (key == '*') {
      lcd.clear();
      lcd.print("Da huy thao tac!");
      delay(1000);
      return -1; 
    }
    if (key == 'D') return 1; 
    
    if (millis() - startTime > 10000) { 
      lcd.clear();
      lcd.print("Qua thoi gian!");
      delay(2000);
      return 1;
    }
  }

  if (rfidCount >= MAX_USERS) {
    lcd.clear();
    lcd.print("Bo nho the day!");
    mfrc522.PICC_HaltA();
    delay(2000);
    return 1;
  }

  memcpy(registeredCards[rfidCount].uid, mfrc522.uid.uidByte, 4);
  rfidCount++;
  saveSystemData();

  lcd.clear();
  lcd.print("Them the OK!");
  mfrc522.PICC_HaltA();
  delay(2000);
  return 0;
}

int deleteRFID() {
  lcd.clear();
  lcd.print("1-Xoa 1 the");
  lcd.setCursor(0, 1);
  lcd.print("2-Xoa tat ca");
  
  char choice;
  while(true){
    choice = keypad.waitForKey();
    if (choice == '*') return -1;
    if (choice == 'D') return 1;
    if (choice == '1' || choice == '2') break;
  }

  if (choice == '1') {
    lcd.clear();
    lcd.print("Quet the can xoa");
    
    while (!mfrc522.PICC_IsNewCardPresent() || !mfrc522.PICC_ReadCardSerial()) {
      char key = keypad.getKey();
      if (key == '*') {
        lcd.clear();
        lcd.print("Da huy thao tac!");
        delay(1000);
        return -1;
      }
      if (key == 'D') return 1;
    }
    
    int indexToDelete = -1;
    for (int i = 0; i < rfidCount; i++) {
      if (memcmp(mfrc522.uid.uidByte, registeredCards[i].uid, 4) == 0) {
        indexToDelete = i;
        break;
      }
    }

    if (indexToDelete != -1) {
      for (int i = indexToDelete; i < rfidCount - 1; i++) {
        registeredCards[i] = registeredCards[i + 1];
      }
      rfidCount--;
      saveSystemData();
      lcd.clear();
      lcd.print("Da xoa the!");
    } else {
      lcd.clear();
      lcd.print("The chua dang ky");
    }
    mfrc522.PICC_HaltA();
  } 
  else if (choice == '2') {
    lcd.clear();
    lcd.print("Xoa TAT CA THE?");
    lcd.setCursor(0, 1);
    lcd.print("Bam # xac nhan");

    while (true) {
      char key = keypad.waitForKey();
      if (key == '#') {
        rfidCount = 0;
        saveSystemData();
        lcd.clear();
        lcd.print("Da xoa het the!");
        break; 
      }
      else if (key == 'D') return 1;  
      else if (key == '*') return -1; 
    }
  }
  delay(2000);
  return 0;
}

// -------------------------------------------------------------------------
// ĐỔI MÃ PIN & LƯU TRỮ EEPROM
// -------------------------------------------------------------------------
int changePins(char mode) {
  String currentPin = (mode == '1') ? adminPin : userPin;
  String oldPinInput = inputSecret("Nhap PIN cu:");
  if (oldPinInput == "CANCEL") return -1; 
  if (oldPinInput == "BACK") return 1;
  
  if (oldPinInput != currentPin) {
    lcd.clear();
    lcd.print("Sai ma PIN cu!");
    delay(2000);
    return 1;
  }

  String newPin = inputSecret("Nhap PIN moi:");
  if (newPin == "CANCEL") return -1; 
  if (newPin == "BACK") return 1;
  
  if (newPin.length() == 4) {
    String confirmPin = inputSecret("Xac nhan PIN moi:");
    if (confirmPin == "CANCEL") return -1; 
    if (confirmPin == "BACK") return 1;
    
    if (newPin == confirmPin) {
      if (mode == '1') adminPin = newPin;
      else userPin = newPin;
      
      saveSystemData();
      lcd.clear();
      lcd.print("Doi PIN OK!");
    } else {
      lcd.clear();
      lcd.print("Khong trung khop!");
    }
  }
  delay(2000);
  return 0;
}

void saveSystemData() {
  for (int i = 0; i < 4; i++) EEPROM.write(ADDR_ADMIN_PIN + i, adminPin[i]);
  for (int i = 0; i < 4; i++) EEPROM.write(ADDR_USER_PIN + i, userPin[i]);
  EEPROM.write(ADDR_RFID_COUNT, rfidCount);
  int currentAddr = ADDR_RFID_DATA;
  for (int i = 0; i < rfidCount; i++) {
    for (int j = 0; j < 4; j++) {
      EEPROM.write(currentAddr, registeredCards[i].uid[j]);
      currentAddr++;
    }
  }
  EEPROM.commit();
}

void loadSystemData() {
  if (EEPROM.read(ADDR_ADMIN_PIN) == 0xFF || EEPROM.read(ADDR_ADMIN_PIN) == 0) {
    saveSystemData(); 
    return;
  }
  adminPin = "";
  for (int i = 0; i < 4; i++) adminPin += (char)EEPROM.read(ADDR_ADMIN_PIN + i);
  userPin = "";
  for (int i = 0; i < 4; i++) userPin += (char)EEPROM.read(ADDR_USER_PIN + i);
  
  rfidCount = EEPROM.read(ADDR_RFID_COUNT);
  if(rfidCount > MAX_USERS) rfidCount = 0;
  
  int currentAddr = ADDR_RFID_DATA;
  for (int i = 0; i < rfidCount; i++) {
    for (int j = 0; j < 4; j++) {
      registeredCards[i].uid[j] = EEPROM.read(currentAddr);
      currentAddr++;
    }
  }
}
