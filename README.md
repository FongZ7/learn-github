#include "SPI.h"  // รวมไลบรารี SPI สำหรับการสื่อสาร
#include <TFT_eSPI.h>  // รวมไลบรารีสำหรับการจัดการหน้าจอ

#include <Arduino.h>  // รวมไลบรารี Arduino
#include "driver/gpio.h"  // รวมไลบรารีสำหรับการจัดการ GPIO
#include "driver/twai.h"  // รวมไลบรารีสำหรับการจัดการ TWAI (CAN)

// การตั้งค่าปุ่ม
const int button1Pin = 4;  // ขา GPIO สำหรับปุ่ม 1
const int button2Pin = 5;  // ขา GPIO สำหรับปุ่ม 2
const int button3Pin = 6;  // ขา GPIO สำหรับปุ่ม 3
const int buttonIncPin = 7;  // ขา GPIO สำหรับปุ่มเพิ่มค่า
const int buttonDecPin = 15;  // ขา GPIO สำหรับปุ่มลดค่า

#define CAN_TX_PIN GPIO_NUM_1  // กำหนดขา GPIO สำหรับส่งข้อมูล CAN
#define CAN_RX_PIN GPIO_NUM_2  // กำหนดขา GPIO สำหรับรับข้อมูล CAN

int currentValue = 0;  // ตัวแปรสำหรับเก็บค่าปัจจุบัน
int value = 0;  // ตัวแปรสำหรับเก็บค่าเพื่อเปรียบเทียบการเปลี่ยนแปลง
int Data = 0;  // ตัวแปรสำหรับเก็บค่าที่จะส่งผ่าน CAN
TFT_eSPI tft = TFT_eSPI();  // สร้างอ็อบเจ็กต์ "tft" สำหรับหน้าจอ

void setup() {
  // การตั้งค่าปุ่ม
  pinMode(button1Pin, INPUT_PULLDOWN);  // ตั้งค่าปุ่ม 1 เป็น INPUT พร้อม PULLDOWN
  pinMode(button2Pin, INPUT_PULLDOWN);  // ตั้งค่าปุ่ม 2 เป็น INPUT พร้อม PULLDOWN
  pinMode(button3Pin, INPUT_PULLDOWN);  // ตั้งค่าปุ่ม 3 เป็น INPUT พร้อม PULLDOWN
  pinMode(buttonIncPin, INPUT_PULLDOWN);  // ตั้งค่าปุ่มเพิ่มค่าเป็น INPUT พร้อม PULLDOWN
  pinMode(buttonDecPin, INPUT_PULLDOWN);  // ตั้งค่าปุ่มลดค่าเป็น INPUT พร้อม PULLDOWN

  // เริ่มต้นหน้าจอ
  pinMode(45, OUTPUT);  // ตั้งค่าขา GPIO 45 เป็น OUTPUT
  digitalWrite(45, HIGH);  // ส่งสัญญาณ HIGH ไปที่ขา GPIO 45
  Serial.begin(115200);  // เริ่มต้นการสื่อสารผ่าน Serial ที่ความเร็ว 115200
  tft.init();  // เริ่มต้นหน้าจอ
  tft.setRotation(3);  // ตั้งค่าการหมุนหน้าจอเป็นแนวนอน

  // ล้างหน้าจอด้วยสีขาว
  tft.fillScreen(TFT_WHITE);  // ล้างหน้าจอด้วยสีขาว

  // ตั้งค่าสีและขนาดของข้อความ
  tft.setTextColor(TFT_BLACK, TFT_WHITE);  // กำหนดสีข้อความเป็นสีดำและสีพื้นหลังเป็นสีขาว

  // ตั้งค่าทั่วไปของ TWAI (CAN)
  twai_general_config_t g_config = TWAI_GENERAL_CONFIG_DEFAULT(CAN_TX_PIN, CAN_RX_PIN, TWAI_MODE_NORMAL);  // ตั้งค่าทั่วไปสำหรับ CAN
  twai_timing_config_t t_config = TWAI_TIMING_CONFIG_125KBITS();  // ตั้งค่าความเร็ว CAN ที่ 125 kbps
  twai_filter_config_t f_config = TWAI_FILTER_CONFIG_ACCEPT_ALL();  // ตั้งค่าตัวกรองให้รับทุกข้อความ

  // ติดตั้งไดรเวอร์ TWAI (CAN)
  if (twai_driver_install(&g_config, &t_config, &f_config) != ESP_OK) {  // ติดตั้งไดรเวอร์ CAN
    Serial.println("Failed to install driver");  // หากติดตั้งไม่สำเร็จ พิมพ์ข้อความนี้
    return;  // ออกจากฟังก์ชัน
  }

  // เริ่มการทำงานของไดรเวอร์ TWAI (CAN)
  if (twai_start() != ESP_OK) {  // เริ่มต้นไดรเวอร์ CAN
    Serial.println("Failed to start driver");  // หากเริ่มต้นไม่สำเร็จ พิมพ์ข้อความนี้
    return;  // ออกจากฟังก์ชัน
  }
  Foam(0);  // เรียกใช้ฟังก์ชัน Foam เพื่อตั้งค่าเริ่มต้น
  sendtwaiMessage(Data);  // ส่งข้อความผ่าน CAN
}

void loop() {
  // ตรวจสอบการกดปุ่ม
  if (digitalRead(button1Pin) == HIGH) {  // ถ้าปุ่ม 1 ถูกกด
    currentValue = 1;  // ตั้งค่าปัจจุบันเป็น 1
    if (currentValue != Data) {  // ถ้าค่าปัจจุบันไม่เท่ากับค่าใน Data
      Data = currentValue;  // อัปเดต Data
      sendtwaiMessage(Data);  // ส่งข้อความผ่าน CAN
    }
  }
  if (digitalRead(button2Pin) == HIGH) {  // ถ้าปุ่ม 2 ถูกกด
    currentValue = 3;  // ตั้งค่าปัจจุบันเป็น 3
    if (currentValue != Data) {  // ถ้าค่าปัจจุบันไม่เท่ากับค่าใน Data
      Data = currentValue;  // อัปเดต Data
      sendtwaiMessage(Data);  // ส่งข้อความผ่าน CAN
    }
  }
  if (digitalRead(button3Pin) == HIGH) {  // ถ้าปุ่ม 3 ถูกกด
    currentValue = 6;  // ตั้งค่าปัจจุบันเป็น 6
    if (currentValue != Data) {  // ถ้าค่าปัจจุบันไม่เท่ากับค่าใน Data
      Data = currentValue;  // อัปเดต Data
      sendtwaiMessage(Data);  // ส่งข้อความผ่าน CAN
    }                                                                                                       
  } 
  if (digitalRead(buttonIncPin) == HIGH) {  // ถ้าปุ่มเพิ่มค่าถูกกด
    currentValue = min(currentValue + 1, 100);  // เพิ่มค่าปัจจุบันทีละ 1% และจำกัดค่าสูงสุดที่ 100
    delay(200);  // หน่วงเวลา 200 มิลลิวินาที
    if (currentValue != Data) {  // ถ้าค่าปัจจุบันไม่เท่ากับค่าใน Data
      Data = currentValue;  // อัปเดต Data
      sendtwaiMessage(Data);  // ส่งข้อความผ่าน CAN
    }                              
  }
  if (digitalRead(buttonDecPin) == HIGH) {  // ถ้าปุ่มลดค่าถูกกด
    currentValue = max(currentValue - 1, 0);  // ลดค่าปัจจุบันทีละ 1% และจำกัดค่าต่ำสุดที่ 0
    delay(200);  // หน่วงเวลา 200 มิลลิวินาที
    if (currentValue != Data) {  // ถ้าค่าปัจจุบันไม่เท่ากับค่าใน Data
      Data = currentValue;  // อัปเดต Data
      sendtwaiMessage(Data);  // ส่งข้อความผ่าน CAN
    }                              
  }
  delay(50);  // หน่วงเวลา 50 มิลลิวินาที

  // รับข้อมูลจาก CAN bus
  twai_message_t message;  // สร้างตัวแปรสำหรับเก็บข้อความ CAN
  if (twai_receive(&message, pdMS_TO_TICKS(10)) == ESP_OK) {  // รับข้อความจาก CAN ภายในเวลา 10 มิลลิวินาที
    if (message.data_length_code == 1) {  // ถ้าขนาดข้อมูลของข้อความเท่ากับ 1 ไบต์
      int receivedValue = message.data[0];  // รับค่าจากข้อมูล
      if (receivedValue != currentValue) {  // อัปเดตค่าหากค่าแตกต่าง
        currentValue = receivedValue;  // อัปเดตค่าปัจจุบัน
      }
    }
  }
  if (currentValue != value) {  // ถ้าค่าปัจจุบันไม่เท่ากับค่าใน value
    value = currentValue;  // อัปเดต value
    Foam(value);  // เรียกใช้ฟังก์ชัน Foam เพื่ออัปเดตหน้าจอ
  }
}

void Foam(int V) {
  if (V >= 0) {  // ถ้าค่าปัจจุบันไม่เป็นลบ
    int W = 100 - V;  // คำนวณค่าน้ำ
    tft.setCursor(0, 0);  // ตั้งตำแหน่งเคอร์เซอร์
    tft.fillRect(15, 40, 165, 250, TFT_WHITE);  // ล้างพื้นที่ข้อความด้วยสีขาว
    
    tft.setTextSize(1);  // กำหนดขนาดข้อความ
    tft.setCursor(0, 0);  // ตั้งตำแหน่งเคอร์เซอร์
    tft.setTextDatum(MC_DATUM);  // กำหนดตำแหน่งศูนย์กลางข้อความเพื่อการจัดตำแหน่งที่แม่นยำ
    tft.drawString(String(V), 90, 80, 8);  // แสดงข้อความเปอร์เซ็นต์โฟมที่ตำแหน่ง (90, 80) ขนาดตัวอักษร 8
    tft.setTextSize(1);  // กำหนดขนาดข้อความ
    tft.drawString("% Foam", 230, 100, 4);  // แสดงข้อความ "% Foam" ที่ตำแหน่ง (230, 100) ขนาดตัวอักษร 4

    tft.setTextSize(1);  // กำหนดขนาดข้อความ
    tft.setCursor(0, 100);  // ตั้งตำแหน่งเคอร์เซอร์
    tft.setTextDatum(MC_DATUM);  // กำหนดตำแหน่งศูนย์กลางข้อความเพื่อการจัดตำแหน่งที่แม่นยำ
    tft.drawString(String(W), 90, 180, 8);  // แสดงข้อความเปอร์เซ็นต์น้ำที่ตำแหน่ง (90, 180) ขนาดตัวอักษร 8
    tft.setTextSize(1);  // กำหนดขนาดข้อความ
    tft.drawString("% Water", 230, 200, 4);  // แสดงข้อความ "% Water" ที่ตำแหน่ง (230, 200) ขนาดตัวอักษร 4
  }
  delay(50);  // หน่วงเวลา 50 มิลลิวินาที
}

void sendtwaiMessage(int value) {
  twai_message_t tx_message;  // สร้างตัวแปรสำหรับเก็บข้อความที่ส่งผ่าน CAN
  tx_message.identifier = 0x1;  // กำหนดไอดีของข้อความ
  tx_message.flags = TWAI_MSG_FLAG_NONE;  // ตั้งค่าธงของข้อความ
  tx_message.data_length_code = 1;  // กำหนดขนาดข้อมูลของข้อความเป็น 1 ไบต์
  tx_message.data[0] = value;  // กำหนดค่าข้อมูลเป็นค่าที่ส่งผ่านเข้ามา

  if (twai_transmit(&tx_message, pdMS_TO_TICKS(10)) == ESP_OK) {  // ส่งข้อความผ่าน CAN ภายในเวลา 10 มิลลิวินาที
    Serial.println("Message sent successfully");  // ถ้าส่งข้อความสำเร็จ แสดงข้อความนี้
  } else {
    Serial.println("Failed to send message");  // ถ้าส่งข้อความไม่สำเร็จ แสดงข้อความนี้
  }
}
