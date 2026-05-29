# SpiderBot_Controller

**Arduino framework to control SpiderBot (which uses XN297 protocol) via nRF24L01.**

Fully abstracts the XN297 protocol (scramble, CRC, frequency hopping, bind) and lets you connect any control source with just a few lines of code.

---

##  Features

- ✅ Full XN297 protocol (scramble, bind, frequency hopping, CRC)  
- ✅ nRF24L01 support 
- ✅ Adapters prepared for Bluetooth controllers, own controllers with joystick and serial monitor
- ✅ Interface to create custom **radio** and **input** adapters  
- ✅ Callbacks for events: pairing, reception, transmission  

---

##  Installation

### Arduino IDE (recommended)
1. Download the repository as a `.zip`
2. **Sketch → Include Library → Add .ZIP Library**


### Dependencies
| Library | Required | Notes |
|---|---|---|
| [RF24](https://github.com/nRF24/RF24) | ✅ | To use `NRF24Radio` |
| [Bluepad32](https://github.com/ricardoquesada/bluepad32) | ⚪ Optional | To use `Bluepad32InputAdapter` |

---

##  Quick Start

Use any of the provided examples!

---

##  nRF24L01 → ESP32 Wiring used

| nRF24L01 | ESP32 |
|---|---|
| VCC | 3.3V |
| GND | GND |
| CE  | GPIO 4 |
| CSN | GPIO 5 |
| SCK | GPIO 18 |
| MOSI| GPIO 23 |
| MISO| GPIO 19 |

---

### Available commands

| Action | payload[0] | payload[1] |
|---|---|---|
| IDLE | 0x00 | 0xFF |
| Up | 0x01 | 0xFE |
| Double fire | 0x02 | 0xFD |
| Down | 0x04 | 0xFB |
| Right | 0x08 | 0xF7 |
| Single fire | 0x10 | 0xEF |
| Left | 0x20 | 0xDF |
| Crouch | 0x40 | 0xBF |
| Shield | 0x80 | 0x7F |
| Eject | 0x00 | 0xFF | (payload[2]=0x88) |

---

##  Radio adapter

Implement `XN297RadioBase`:

```cpp
class MyRadio : public XN297RadioBase {
public:
    bool    begin()                              override { /* init */ return true; }
    void    setChannel(uint8_t ch)               override { /* change frequency */ }
    void    writeRaw(const uint8_t* buf, uint8_t len) override { /* transmit */ }
    bool    available()                          override { /* any RX data? */ return false; }
    uint8_t readRaw(uint8_t* buf, uint8_t maxLen) override { /* read RX */ return 0; }
    void    startListening()                     override { /* RX mode */ }
    void    stopListening()                      override { /* TX mode */ }
};
```

---

##  Creating your own input adapter

Implement `SpiderBot_InputAdapter`:

```cpp
class MyInput : public SpiderBot_InputAdapter {
public:
    void begin()  override { /* setup */ }
    void update() override { /* read hardware */ }

    bool getPayload(uint8_t payload[6]) override {
        // Fill payload with the desired command
        // Return true if there is active input, false for IDLE
        payload[0] = 0x01; payload[1] = 0xFE; // UP
        return true;
    }
};
```

##  Repository structure

```
SpiderBot_Controller/
├── src/
│   ├── SpiderBot_Controller.h      # Main class
│   ├── SpiderBot_Controller.cpp
│   ├── XN297Radio.h                # Radio abstraction + NRF24Radio
│   └── SpiderBot_InputAdapter.h    # Input abstraction + Serial + Bluepad32
├── examples/
│   ├── BasicSerial/                # Keyboard/Serial control
│   ├── BluetoothPS4/               # Bluetooth controller with Bluepad32
│   └── CustomHardware/             # Template for custom hardware
├── library.properties
└── README.md
```
---

