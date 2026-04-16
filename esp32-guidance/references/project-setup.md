# Project Setup and Configuration

## Project Structure

Standard PlatformIO project layout:

```
my-esp32-project/
├── platformio.ini          # Build configuration
├── src/
│   └── main.cpp            # Main sketch (must be named main.cpp)
├── include/
│   ├── config.h            # Configuration constants
│   └── sensors.h           # Custom headers
├── lib/                    # Local libraries
│   └── MyLibrary/
│       ├── src/
│       └── library.properties
├── test/                   # Unit tests (optional)
└── README.md              # Project documentation
```

**Key rule:** `src/main.cpp` is required; PlatformIO will not find `sketch.ino` or other names.

## Minimal platformio.ini

This works for most ESP32-C3 projects:

```ini
[env:esp32-c3]
platform = espressif32
board = esp32-c3-devkitm-1
framework = arduino

; USB serial support
build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1

; Monitor settings
monitor_speed = 115200
upload_speed = 460800
```

## Complete platformio.ini with Common Options

```ini
[platformio]
default_envs = esp32-c3

[env:esp32-c3]
platform = espressif32
board = esp32-c3-devkitm-1
framework = arduino

; ===== USB Serial (Required) =====
build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1

; ===== ESP-IDF Logging =====
; Uncomment to enable ESP-IDF logs
; build_flags = ${env:esp32-c3.build_flags}
;     -DLOG_LOCAL_LEVEL=ESP_LOG_DEBUG

; ===== Monitor and Upload =====
monitor_speed = 115200
upload_speed = 460800
monitor_filters = esp32_exception_decoder

; ===== Memory Layout =====
; Useful for flash-heavy projects
; board_build.flash_size = 2MB
; board_build.psram_mode = disabled
```

## Board Selection Reference

Select the correct board in `platformio.ini`:

| Hardware | Board ID | Notes |
|----------|----------|-------|
| ESP32-C3 DevKit | `esp32-c3-devkitm-1` | Most common, auto-reset |
| ESP32-C2 Super Mini | `esp32-c2-devkitm-1` | Manual BOOT+RESET needed |
| ESP32-S3 DevKit | `esp32-s3-devkitc-1` | Large board, more features |
| ESP32-S2 | `esp32-s2-saola-1` | Medium board |
| ESP32 (Original) | `esp32doit-devkit-v1` | Legacy, external serial |

## Minimal main.cpp

Use this as a starting point:

```cpp
#include <Arduino.h>

void setup() {
  // Initialize serial
  Serial.begin(115200);
  delay(500); // Wait for serial ready
  
  Serial.println("\n\n=== Setup Complete ===");
}

void loop() {
  Serial.println("Loop running...");
  delay(1000);
}
```

## main.cpp with EVERY_N_MILLIS and Dual Logging

Template for projects with timing, WiFi, and logging:

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <esp_log.h>
#include <esp_wifi.h>

// ===== Configuration =====
const char* ssid = "your_ssid";
const char* password = "your_password";
static const char* TAG = "MainApp";

// ===== Macros =====
#define EVERY_N_MILLIS_ID(ID, N) \
  static uint32_t _timer_##ID = 0; \
  if (millis() - _timer_##ID >= (N)) { \
    _timer_##ID = millis(); \
    true; \
  } else { \
    false; \
  }

// ===== Setup Function =====
void setup() {
  // Serial setup
  Serial.begin(115200);
  delay(500);
  
  // Logging setup
  esp_log_level_set("*", ESP_LOG_INFO);
  esp_log_level_set(TAG, ESP_LOG_DEBUG);
  
  Serial.println("\n=== Setup Start ===");
  ESP_LOGI(TAG, "Application starting");
  
  // WiFi setup
  WiFi.mode(WIFI_STA);
  WiFi.setTxPower(WIFI_POWER_8_5dBm);
  uint8_t protocol = WIFI_PROTOCOL_11B | WIFI_PROTOCOL_11G;
  esp_wifi_set_protocol(WIFI_IF_STA, protocol);
  WiFi.begin(ssid, password);
  
  ESP_LOGI(TAG, "Setup complete, waiting for WiFi...");
}

// ===== Loop Function =====
bool wifiConnected = false;

void loop() {
  // WiFi polling every 1 second
  if (EVERY_N_MILLIS_ID(wifi_poll, 1000)) {
    if (WiFi.status() == WL_CONNECTED && !wifiConnected) {
      wifiConnected = true;
      ESP_LOGI(TAG, "WiFi connected: %s", WiFi.localIP().toString().c_str());
    }
  }
  
  // Main work every 500ms
  if (EVERY_N_MILLIS_ID(work, 500)) {
    Serial.println("Doing work...");
  }
  
  // Log status every 5 seconds
  if (EVERY_N_MILLIS_ID(status, 5000)) {
    Serial.print("Uptime: ");
    Serial.print(millis() / 1000);
    Serial.println(" seconds");
  }
  
  // Feed watchdog
  yield();
  esp_task_wdt_reset();
}
```

## Building and Uploading

### Build Only
```bash
platformio run --environment esp32-c3
```

### Upload to Board
```bash
platformio run --environment esp32-c3 --target upload
```

### Build and Monitor Serial Output
```bash
platformio run --environment esp32-c3 --target upload --monitor
```

### Monitor Without Uploading
```bash
platformio device monitor --port /dev/ttyUSB0 --baud 115200
```

## Common Build Issues

### Flash Size Exceeded

**Error:** `Image too large for partition`

**Solutions:**
1. **Remove unused code** — Delete debug serial.print() calls in loop
2. **Optimize logging** — Use ESP-IDF logs instead of Serial.printf
3. **Use partition table** — Larger flash layout
4. **Build with optimizations:** 
   ```ini
   build_flags = -O2
   ```

### Serial Monitor Won't Connect

**Solutions:**
1. Close other serial monitors (only one can connect)
2. Check port: `platformio device list`
3. Reset board: Press RESET button
4. Try different USB cable/port

### Upload Fails: Device Not Found

**Solutions:**
1. Check USB connection
2. Try manual BOOT + RESET sequence
3. Check correct board selected in platformio.ini
4. Update drivers (especially Windows)

## IDE Integration

### VS Code

Install extensions:
1. **PlatformIO IDE** - Provides build/upload buttons in UI

Then:
1. Open project folder
2. platformio.ini will be auto-detected
3. Build/upload buttons appear in bottom bar

### CLion / IntelliJ

1. Install **PlatformIO for CLion** plugin
2. Open project folder
3. PlatformIO toolbar appears

### Arduino IDE (Not Recommended)

PlatformIO is better than Arduino IDE for ESP32 projects because:
- Better build system
- Easier library management
- Build flags easier to set
- Upload process more reliable

But if you must use Arduino IDE:
1. Install "ESP32 by Espressif Systems" board package
2. Select Tools > Board > ESP32 > ESP32C3
3. Set Tools > Upload Speed to 460800

## Getting Help

### PlatformIO Docs
- https://docs.platformio.org/

### ESP32 Arduino Framework
- https://docs.espressif.com/projects/arduino-esp32/

### ESP-IDF Documentation
- https://docs.espressif.com/projects/esp-idf/

### Community
- PlatformIO Forum: https://community.platformio.org/
- ESP32 Forum: https://www.esp32.com/

## Key Rules

1. **Name main file `src/main.cpp`** — PlatformIO requires this
2. **Set USB build flags in platformio.ini** — Required for serial on C3
3. **Select correct board in platformio.ini** — Affects build and upload
4. **Use 115200 baud for serial** — Standard for ESP32
5. **Monitor serial in same environment** — Only one connection allowed
6. **Always call delay(500) after Serial.begin()** — Gives USB time to initialize
7. **Close serial monitor before uploading** — Blocks upload if connected
