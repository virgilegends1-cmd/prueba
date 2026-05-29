# SpiderBot Controller Framework

**An Arduino framework to control the SpiderBot toy (which uses a proprietary XN297 protocol) using a standard nRF24L01 radio module.**

This project uses software abstracts the XN297 protocol. It emulates the preamble, scrambler, software CRC, and frequency hopping sequence, allowing you to control the SpiderBot and receive telemetry (health/shots) using an ESP32 or Arduino.

---

## ⚙️ Framework Architecture

The framework is built around a central hub and modular adapters. You don't need to touch the radio protocol; you just plug adapters into the controller:

1. **`SpiderBot_Controller` (The Hub):** Manages the XN297 state machine (Searching -> Waiting ACK -> Bound), frequency hopping, and packet encoding/decoding.
2. **`RadioAdapter`:** The hardware layer communicating with the nRF24L01.
3. **`InputAdapters`:** Read user input (Serial, Analog Joysticks, Bluetooth Gamepads...) and translate them into SpiderBot commands. 
4. **`OutputAdapters`:** Receive telemetry from the SpiderBot (Health, Shots left) and display it (Serial monitor, OLED screens...). 

---

## 🚀 Installation & Dependencies

### Arduino IDE
1. Download this repository as a `.zip` file.
2. Go to **Sketch → Include Library → Add .ZIP Library...**

### Dependencies
| Library | Required | Notes |
|---|---|---|
| [RF24](https://github.com/nRF24/RF24)  | Required for the `NRF24Radio` adapter. |
| [Bluepad32](https://github.com/ricardoquesada/bluepad32) | Only needed if you use the `Bluepad32InputAdapter`. |
| [Adafruit SSD1306](https://github.com/adafruit/Adafruit_SSD1306) | Only needed if you use the OLED display adapter. |

---

## 🔌 Used Wiring (ESP32-WROOM-32 -> nRF24L01)

| nRF24L01 | ESP32-WROOM-32 (Default SPI) |
|---|---|
| VCC | 3.3V |
| GND | GND |
| CE  | GPIO 4 |
| CSN | GPIO 5 |
| SCK | GPIO 18 | (Default)
| MOSI| GPIO 23 | (Default)
| MISO| GPIO 19 | (Default)

---

## 🛠️ Extending the Framework

The true potential of this framework is the ability to create your own adapters.

### 1. Creating Custom Inputs (Controllers)
To create a new way to control the SpiderBot (e.g., a web server, a custom remote, sensors), create a class that inherits from `SpiderBot_InputAdapter`.

**The Payload Rule:** The SpiderBot requires `Byte 1` to be the logical inverse of `Byte 0` (Bitwise NOT). For example, if UP is `0x01` (`00000001`), its inverse is `0xFE` (`11111110`). The helper method `_cmd()` handles array creation for you.

```cpp
#include "SpiderBot_InputAdapter.h"

class MyCustomInput : public SpiderBot_InputAdapter {
public:
    void begin() override {
        // Initialize your hardware (e.g., pinMode)
    }

    void update() override {
        // Read sensors/buttons here if needed continuously
    }

    bool getPayload(uint8_t payload[6]) override {
        // Return TRUE if a button is pressed, FALSE if nothing is pressed (IDLE)
        if (digitalRead(PIN_UP) == LOW) {
            return _cmd(payload, 0x01, 0xFE); // UP command
        }
        if (digitalRead(PIN_FIRE) == LOW) {
            return _cmd(payload, 0x10, 0xEF); // FIRE command
        }
        return false; 
    }

    bool checkReset() override {
        // Return TRUE if the user wants to force the framework to search for a new toy
        return (digitalRead(PIN_RESET) == LOW);
    }
};
```

### 2. Creating Custom Outputs (Displays/Telemetry)
When the SìderBot is paired, it constantly sends back its current state. You can read this to light up LEDs, play sounds, or update screens by inheriting from `SpiderBot_OutputAdapter`.

The `SpiderBotState` struct provides:
*   `health` (0 to 10)
*   `shots` (0 to 3)

```cpp
#include "SpiderBot_Controller.h"

class MyLEDOutput : public SpiderBot_OutputAdapter {
public:
    void begin() override {
        pinMode(LED_PIN, OUTPUT);
    }

    void update(const SpiderBotState& state) override {
        // This is called automatically when new telemetry arrives
        if (state.health <= 3) {
            digitalWrite(LED_PIN, HIGH); // Low health warning!
        } else {
            digitalWrite(LED_PIN, LOW);
        }
    }
};
```

---

## 🎮 Basic Usage Example

Here is how you put all the pieces together in your main `.ino` sketch:

```cpp
#include <SpiderBot_Controller.h>

// 1. Instantiate the radio hardware (CE, CSN)
NRF24Radio radio(4, 5);

// 2. Instantiate your inputs and outputs
SerialInputAdapter input;
SerialDisplayAdapter output;

// 3. Create the central controller
SpiderBot_Controller controller(&radio);

void setup() {
    Serial.begin(115200);
    Serial.println("Starting SpiderBot Serial Controller");

    // 4. Attach modules to the controller
    controller.addInput(&Input);
    controller.addOutput(&Output);

    // 5. Start everything
     controller.begin();
}

void loop() {
    // 6. Keep the framework running
    controller.update();
}
```

---

## 🕹️ Pairing a Bluetooth Controller (Bluepad32)

If you are using the `Bluepad32InputAdapter` to control the SpiderBot with a wireless gamepad, the pairing process is handled automatically by the ESP32.

By default, the framework is set to **forget previous controllers** every time it reboots to ensure a clean connection.

**To pair your controller:**
1. Turn on your ESP32. Check the Serial Monitor; it should say `"Press your remote controller pairing button."`
2. Put your gamepad into **Pairing Mode**:
   * **PlayStation 4 / 5:** Hold the `Share` + `PS` buttons simultaneously until the light bar flashes rapidly.
   * **Xbox Wireless:** Turn it on and hold the small `Sync` button on top until the Xbox logo blinks quickly.
   * **Nintendo Switch Pro:** Hold the small `Sync` button near the USB port.
3. The ESP32 will automatically find it, pair with it, and print `"[BT] Remote control connected."`

---

### Command Reference Table

| Action | `B0` (Action) | `B1` (Inverse `~B0`) | `B2` |
|---|---|---|---|
| IDLE | `0x00` | `0xFF` | `0x00` |
| Up | `0x01` | `0xFE` | `0x00` |
| Double fire | `0x02` | `0xFD` | `0x00` |
| Down | `0x04` | `0xFB` | `0x00` |
| Right | `0x08` | `0xF7` | `0x00` |
| Single fire | `0x10` | `0xEF` | `0x00` |
| Left | `0x20` | `0xDF` | `0x00` |
| Crouch | `0x40` | `0xBF` | `0x00` |
| Shield | `0x80` | `0x7F` | `0x00` |
| Eject Header | `0x00` | `0xFF` | `0x88` |

---

## 📁 Repository structure

```text
Spider-Bot_Remote/
├── src/
│   ├── SpiderBot_Controller.h      # Main class
│   ├── SpiderBot_Controller.cpp
│   ├── XN297Radio.h                # Radio abstraction + NRF24Radio
│   └── SpiderBot_InputAdapter.h    # Input abstraction + Serial + Bluepad32
├── examples/
│   ├── All_In_One/                 # Combines multiple inputs and outputs
│   ├── BasicSerial/                # Keyboard/Serial monitor control
│   ├── BluetoothController/        # Bluetooth gamepad control via Bluepad32
│   └── PhysicalController/         # Custom hardware (Analog joysticks & buttons)
├── library.properties
└── README.md
```