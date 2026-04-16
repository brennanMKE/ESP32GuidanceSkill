# Serial and ESP-IDF Logging on ESP32-C3

## The Critical Problem: Serial.println Doesn't Work by Default

On ESP32-C3, **Serial.println is silent by default**. The USB interface defaults to JTAG debugging mode, not serial communication. This causes a dangerous silent failure:

```cpp
void setup() {
  Serial.begin(115200);
  Serial.println("Hello World"); // ❌ No output appears, NO error message
}
```

**Result:** Code runs, no crashes, but serial output never appears. This wastes debugging time.

## The Solution: Enable USB CDC with Build Flags

Add these flags to `platformio.ini` to switch USB from JTAG to CDC serial mode:

```ini
[env:esp32-c3]
platform = espressif32
board = esp32-c3-devkitm-1
framework = arduino

build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1
```

**What these flags do:**
- `ARDUINO_USB_MODE=1`: Switch USB from JTAG debugging to CDC serial communication
- `ARDUINO_USB_CDC_ON_BOOT=1`: Initialize USB CDC at boot (Serial works immediately, no driver delay)

**With these flags, Serial.println works:**
```cpp
void setup() {
  Serial.begin(115200);
  delay(500); // Wait for USB CDC to initialize
  Serial.println("Hello World"); // ✓ Now appears on serial monitor
}
```

## Important Trade-Off: USB CDC vs USB JTAG

The USB interface on ESP32-C3 **cannot run both USB CDC and USB JTAG simultaneously**. You must choose:

| Mode | Use For | Enable With |
|------|---------|-------------|
| **USB CDC** | Serial.println() debugging | `-DARDUINO_USB_MODE=1 -DARDUINO_USB_CDC_ON_BOOT=1` |
| **USB JTAG** | Breakpoints, step-through debugging, variable inspection | Remove USB_MODE flag, add `debug_tool = esp-builtin` |

**Recommendation:** For most projects, use USB CDC (with the build flags). Only switch to USB JTAG if you need advanced debugging with breakpoints.

## Two Logging Systems

Once USB CDC is enabled, you have two logging options:

### 1. Serial (USB CDC) Logging

Works after enabling USB CDC flags:

```cpp
void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("Setup complete");
}

void loop() {
  Serial.print("Value: ");
  Serial.println(sensorValue);
}
```

**Characteristics:**
- Simple, human-readable
- Works immediately without additional setup
- No log levels or filtering
- Good for quick debugging

### 2. ESP-IDF Structured Logging

Works regardless of USB mode (doesn't require USB CDC flags):

```cpp
#include <esp_log.h>

static const char* TAG = "MyApp";

void setup() {
  // Set minimum log level (optional, default is INFO)
  esp_log_level_set("*", ESP_LOG_INFO);
  esp_log_level_set(TAG, ESP_LOG_DEBUG);
  
  ESP_LOGI(TAG, "Application started");
}

void loop() {
  ESP_LOGI(TAG, "Value: %d", sensorValue);
}
```

**Characteristics:**
- Structured: Every log has timestamp, level, tag, message
- Filterable by log level and component tag
- Can be sent to remote logging systems
- More professional, production-ready
- Works with or without USB CDC enabled

## Which Logging System Should You Use?

### Use Serial.println if:
- You want simple, immediate debugging
- You have USB CDC enabled in platformio.ini
- You don't need log level filtering

### Use ESP-IDF Logging if:
- You want structured, filterable logging
- You might switch USB modes later (ESP_LOGI works either way)
- You're building a production system
- You need log levels and component isolation

## Dual Logging: Best of Both Worlds

Combine both for maximum visibility:

```cpp
#include <Arduino.h>
#include <esp_log.h>

static const char* TAG = "SensorApp";

#define EVERY_N_MILLIS_ID(ID, N) \
  static uint32_t _timer_##ID = 0; \
  if (millis() - _timer_##ID >= (N)) { \
    _timer_##ID = millis(); \
    true; \
  } else { \
    false; \
  }

void setup() {
  // Setup Serial (requires USB CDC flags)
  Serial.begin(115200);
  delay(500);
  
  // Setup ESP-IDF logging
  esp_log_level_set("*", ESP_LOG_INFO);
  esp_log_level_set(TAG, ESP_LOG_DEBUG);
  
  // Log to both systems
  Serial.println("=== Sensor Logger Started ===");
  ESP_LOGI(TAG, "Application initialized");
}

float temperature = 0.0;

void loop() {
  // Read sensor every 500ms
  if (EVERY_N_MILLIS_ID(sensor_read, 500)) {
    temperature = readTemperature();
    
    // Detailed log to ESP-IDF (won't always print)
    ESP_LOGD(TAG, "Raw sensor read: %.2f", temperature);
  }
  
  // Log to Serial every 2 seconds (human-readable)
  if (EVERY_N_MILLIS_ID(serial_log, 2000)) {
    Serial.print("Temp: ");
    Serial.print(temperature);
    Serial.println(" C");
  }
  
  // Log to ESP-IDF every 5 seconds (structured, filterable)
  if (EVERY_N_MILLIS_ID(esp_log, 5000)) {
    ESP_LOGI(TAG, "Current temperature: %.1f C", temperature);
    
    // Conditional warnings
    if (temperature > 35) {
      ESP_LOGW(TAG, "High temperature detected: %.1f C", temperature);
    }
  }
  
  yield();
  esp_task_wdt_reset();
}

float readTemperature() {
  return 23.5 + random(-2, 2);
}
```

## Sample Log Output

**Serial output (human-readable):**
```
=== Sensor Logger Started ===
Temp: 23.45 C
Temp: 23.67 C
Temp: 25.34 C
```

**ESP-IDF output (structured, from serial monitor or log viewer):**
```
I (1234) SensorApp: Application initialized
D (2456) SensorApp: Raw sensor read: 23.45
I (5789) SensorApp: Current temperature: 23.5 C
D (2956) SensorApp: Raw sensor read: 23.67
W (10789) SensorApp: High temperature detected: 25.34 C
I (10795) SensorApp: Current temperature: 25.3 C
```

## Log Levels (Highest Priority First)

1. **ESP_LOG_ERROR** — System failures, critical errors
2. **ESP_LOG_WARN** — Warnings, degraded conditions  
3. **ESP_LOG_INFO** — Informational, state changes (default level)
4. **ESP_LOG_DEBUG** — Detailed debugging information
5. **ESP_LOG_VERBOSE** — Very detailed internal state

## Complete platformio.ini with USB CDC

```ini
[env:esp32-c3]
platform = espressif32
board = esp32-c3-devkitm-1
framework = arduino

; Enable USB CDC for Serial.println
build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1

monitor_speed = 115200
upload_speed = 460800
```

## Troubleshooting Serial Output

### No Serial Output Appears

**Checklist:**
1. ✓ Added build flags to platformio.ini?
2. ✓ Rebuilt and uploaded after adding flags?
3. ✓ Called Serial.begin(115200)?
4. ✓ Added delay(500) after Serial.begin()?
5. ✓ Correct port selected in serial monitor?
6. ✓ Baud rate 115200 in serial monitor?

### Serial Output Garbled

**Solutions:**
1. Check baud rate: Should be 115200
2. Check board selection: Ensure platformio.ini has correct board
3. Reset board: Press RESET button

### Can't Switch Back to USB JTAG

If you need USB JTAG debugging later:
1. Comment out or remove the USB_MODE flags
2. Add debug configuration:
   ```ini
   debug_tool = esp-builtin
   upload_protocol = esp-builtin
   ```
3. Rebuild and upload
4. You may need a separate USB-to-UART adapter for serial logging

## Key Rules

1. **Serial.println only works with USB CDC build flags** — Without them, output is silent
2. **Choose USB CDC for Serial, USB JTAG for advanced debugging** — Can't have both
3. **Always delay(500) after Serial.begin()** — Gives USB time to initialize
4. **Use both logging systems together** — Serial for immediate feedback, ESP-IDF for structured data
5. **ESP-IDF logging works regardless of USB mode** — Good fallback if you switch to USB JTAG
6. **Close serial monitor before uploading** — Blocks upload if connected
