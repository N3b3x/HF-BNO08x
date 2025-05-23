# HF - BNO08x
Hardware Agnostic BNO08x library - as used in the HardFOC-V1 controller

# BNO085 C++ Sensor Library 🚀

> **Full-stack, hardware-agnostic, zero-thread driver for Hillcrest / CEVA BNO08x**  

![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)
![Interface: I²C / SPI](https://img.shields.io/badge/Interface-I²C%20%7C%20SPI-9cf)
![Language: C++11](https://img.shields.io/badge/C%2B%2B-11%2B-blue)
![Made with ❤️](https://img.shields.io/badge/Made%20with-%E2%9D%A4-red)

---

## 📜 Table of Contents
1. [Features ✨](#features-✨)
2. [Architecture 📐](#architecture-📐)
3. [Getting Started 🏁](#getting-started-🏁)
4. [Hardware Wiring 🔌](#hardware-wiring-🔌)
5. [Porting Guide 🧳](#porting-guide-🧳)
   * [ESP32 🚀](#esp32-🚀)
   * [STM32 ⚙️](#stm32-⚙️)
   * [Arduino 🎯](#arduino-🎯)
6. [Usage Examples 💻](#usage-examples-💻)
7. [Advanced Notes 🔬](#advanced-notes-🔬)
8. [Contributing 🤝](#contributing-🤝)
9. [License 📄](#license-📄)
10. [Acknowledgements 🙏](#acknowledgements-🙏)

---

## Features ✨
|   | Capability |
|---|-------------|
| 🎯 **Complete Coverage** | Access every BNO085 SH-2 report: raw & calibrated IMU, rotation vectors, activity, tap/shake, step counter & more. |
| 🛠️ **Hardware-Agnostic** | Abstract transport interface (`IBNO085Transport`) lets you plug in *any* I²C, SPI or UART implementation. |
| 💤 **No Internal Threads** | You control timing – call `update()` in your loop, ISR or RTOS task. |
| 🔁 **Auto Re-Sync** | Detects sensor resets & seamlessly re-enables all configured features. |
| 🧮 **Float-Friendly API** | Returns handy structs (`Vector3`, `Quaternion`, `SensorEvent`) with SI units. |
| 📚 **GPLv3 & Apache-2.0** | C++ wrapper under GPLv3; CEVA SH-2 backend under Apache 2.0 – both included. |

---

## Architecture 📐
```mermaid
classDiagram
    direction LR
    class BNO085 {
  +begin() + enableSensor() + update() + setCallback() + getLatest()
    }
    class IBNO085Transport <|.. I2CTransport
    class IBNO085Transport <|.. SPITransport
    class SensorEvent
    BNO085 --> IBNO085Transport : uses
    BNO085 --> SensorEvent : "produces ➡️"
```
The BNO085 class shields your app from the gritty SH-2/SHTP details, while IBNO085Transport shields it from your hardware.

## Library Structure 🗂️

```
src/
 ├── BNO085.hpp / BNO085.cpp  - high level driver
 ├── BNO085_Transport.hpp     - transport interface to implement
 ├── sh2/                      - vendor SH-2 library (git submodule)
 ├── app/, rvc/, dfu/          - reference HAL and DFU utilities
```

Only the `BNO085.*` files and your chosen transport implementation are required
for normal use. The other folders provide optional examples and helper code.

Getting Started 🏁
bash
Always show details

Copy
#Clone wherever you keep libs 📂
git clone --depth=1 https://github.com/yourOrg/bno085-cpp.git libs/bno085

#Add the.cpp /.h files plus sh2/* to your project build.
# CMake example ⤵️
add_subdirectory(libs/bno085)
target_link_libraries(myApp PRIVATE bno085)
Dependencies:
Only a C/C++ compiler (C++11) and your MCU’s I/O driver – no STL, no RTOS.

Hardware Wiring 🔌
txt
Always show details

Copy
MCU 3V3  ──── VIN   BNO085
MCU GND  ──── GND
MCU SCL  ──── SCL   (w/ 4.7 kΩ pull-up)
MCU SDA  ──── SDA   (w/ 4.7 kΩ pull-up)
MCU GPIO ──── INT   (optional, active-low IRQ)
MCU GPIO ──── NRST  (optional reset)
PS0 + PS1 → GND ➡️ selects I²C (tie high for SPI) 
ADR/SA0  → GND ➡️ address 0x4A (0x4B if high)
Tip: Use the INT line to wake your code only when data is ready – saves power & cycles! ⚡️

Porting Guide 🧳
ESP32 🚀 (ESP-IDF v5.x)
cpp
Always show details

Copy
#include "driver/i2c.h"
class Esp32I2CTransport : public IBNO085Transport {
  bool open() override {
      i2c_config_t c = { .mode = I2C_MODE_MASTER,
                         .sda_io_num = 21, .scl_io_num = 22,
                         .master.clk_speed = 400000 };
      i2c_param_config(I2C_NUM_0, &c);
      i2c_driver_install(I2C_NUM_0, c.mode, 0, 0, 0);
      return true;
  }
  int read(uint8_t* d,size_t n)  override { 
      return i2c_master_read_from_device(I2C_NUM_0, 0x4A, d, n, 50)==ESP_OK? n:-1; }
  int write(const uint8_t* d,size_t n) override {
      return i2c_master_write_to_device(I2C_NUM_0, 0x4A, d, n, 50)==ESP_OK? n:-1; }
  uint32_t getTimeUs() override { return esp_timer_get_time(); }
};
💡 See examples/esp32_idf for a full project, including ISR for the INT pin.

STM32 ⚙️ (HAL / CubeIDE)
cpp
Always show details

Copy
extern I2C_HandleTypeDef hi2c1;
class STM32I2CTransport : public IBNO085Transport {
  bool open() override { return HAL_I2C_IsDeviceReady(&hi2c1, 0x4A<<1, 3, 100)==HAL_OK; }
  int  read(uint8_t* b,size_t n) override { return HAL_I2C_Master_Receive(&hi2c1, 0x4A<<1, b,n,100)==HAL_OK?n:-1;}
  int  write(const uint8_t* b,size_t n) override { return HAL_I2C_Master_Transmit(&hi2c1,0x4A<<1,(uint8_t*)b,n,100)==HAL_OK?n:-1;}
  uint32_t getTimeUs() override { return HAL_GetTick()*1000; }
};
For SPI: use HAL_SPI_TransmitReceive & toggle CS manually; tie PS0+PS1 high.

Arduino 🎯
cpp
Always show details

Copy
#include <Wire.h>
class ArduinoTransport : public IBNO085Transport {
  bool open() override { Wire.begin(); Wire.setClock(400000); delay(50); return true; }
  int  read(uint8_t* b,size_t n) override { Wire.requestFrom(0x4A,n); size_t c=0; while(Wire.available()) b[c++]=Wire.read(); return c;}
  int  write(const uint8_t* b,size_t n) override { Wire.beginTransmission(0x4A); Wire.write(b,n); return Wire.endTransmission()==0?n:-1; }
  uint32_t getTimeUs() override { return micros(); }
};
Memory ⛔ note: AVR (<2 KB RAM) is tight – stick to a few low-rate sensors.

Usage Examples 💻
Quick Start
cpp
Always show details

Copy
BNO085 imu(new ArduinoTransport());
if (!imu.begin()) { Serial.println("🚫 IMU not found!"); while(1); }

imu.enableSensor(BNO085Sensor::RotationVector, 10);   // 100 Hz
imu.enableSensor(BNO085Sensor::StepCounter, 0);       // on-change

imu.setCallback([](const SensorEvent& e){
  if (e.sensor == BNO085Sensor::RotationVector) {
      Serial.printf("🧭 Yaw %.1f°\n", e.toEuler().yaw);
  } else if (e.sensor == BNO085Sensor::StepCounter) {
      Serial.printf("👣 Steps: %u\n", e.stepCount);
  }
});

void loop() {
    imu.update();      // call as often as possible (or when INT fires)
}
Polling Loop
cpp
Always show details

Copy
while (true) {
    imu.update();                           // pump packets
    if (imu.hasNewData(BNO085Sensor::TapDetector)) {
        auto tap = imu.getLatestData(BNO085Sensor::TapDetector);
        Serial.println(tap.doubleTap ? "👆 Double Tap!" : "👉 Tap!");
    }
    delay(5);
}
Advanced Notes 🔬
🌟 Tare NOW: imu.tareNow() to zero orientation.

🎯 Accuracy: event.accuracy (0–3) shows calib quality – wait for 3 before trusting heading.

💾 DFU: hold BOOTN low on reset → sensor enters bootloader for firmware update.

🔋 Power: disable unused reports to save ~20 mA.

Contributing 🤝
Pull requests & issues welcome!
Please run clang-format -style=file before committing and sign off your work (-s flag).

License 📄
C++ wrapper code: GNU GPL v3.0

CEVA SH-2 backend: Apache 2.0 (included).

By contributing, you agree code is released under the same GPLv3 license.

Acknowledgements 🙏
CEVA Inc. for open-sourcing the SH-2 driver.

SparkFun & Adafruit for inspiring wiring diagrams.

Everyone in the open-source IMU community 💖

Made with a cup of ☕ and a dash of 🚀
