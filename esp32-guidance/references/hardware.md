# Hardware Quirks, USB CDC, and Upload Sequence

## USB CDC Serial Communication

### Build Flags Required

Add these to `platformio.ini` to enable USB serial on ESP32-C3:

```ini
[env:esp32-c3]
platform = espressif32
board = esp32-c3-devkitm-1
framework = arduino

build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1
```

**What each flag does:**
- `ARDUINO_USB_MODE=1`: Use USB for serial (not UART)
- `ARDUINO_USB_CDC_ON_BOOT=1`: Serial available immediately at boot (no driver wait)

### Without These Flags

```
❌ Serial.begin(115200) does nothing - no output appears
❌ Upload may fail - device not detected
❌ Debugging impossible - no serial feedback
```

### With These Flags

```
✓ Serial output appears immediately
✓ Upload works reliably
✓ Serial monitor connects without delay
```

## Upload Sequence for ESP32-C2/C3

The ESP32-C3 (and C2) require a specific button sequence:

### Manual Upload with BOOT + RESET

1. **Press and hold BOOT button**
2. **Press RESET button once** (while holding BOOT)
3. **Release BOOT button**
4. Upload begins
5. After upload completes, press RESET again to run code

### With Auto-Reset Circuit (Built-in)

Many ESP32-C3 boards include auto-reset:
- Connects RESET and GPIO0 to USB serial DTR/RTS pins
- Automatic button sequence during upload
- Just plug in and upload, no manual buttons needed

**Check your board:** Look for resistor/capacitor network connected to RESET and GPIO0.

## Board-Specific Quirks

### ESP32-C3 (DevKit)

**Default I2C pins:** SDA=5, SCL=6 (non-standard, always specify)
**LED pin:** GPIO8 (not GPIO5)
**Upload:** Usually automatic, but manual buttons work: BOOT + RESET
**Strengths:** Low power, Bluetooth, WiFi, small form factor
**USB:** Built-in USB CDC serial (requires build flags)

### ESP32-C2 Super Mini

**Very small board, common for prototyping:**
- **I2C pins:** SDA=20, SCL=21
- **LED pin:** GPIO5 (commonly used for blink tests)
- **Upload:** Manual BOOT + RESET sequence required
- **Flash size:** Usually 2MB (watch for overflow on large sketches)
- **Strengths:** Tiny, cheap, adequate for sensors
- **Weakness:** Limited GPIO, limited memory

Example initialization:
```cpp
void setup() {
  Serial.begin(115200);
  delay(500);
  
  // Explicit I2C pins for C2 Super Mini
  Wire.begin(20, 21);
  
  // LED blink test
  pinMode(5, OUTPUT);
  digitalWrite(5, HIGH);
}
```

### ESP32-S2 / ESP32-S3

**Larger boards, more GPIO and memory:**
- **I2C pins:** SDA=8, SCL=9 (default and usually safe)
- **LED pin:** GPIO21 or GPIO46 (board-dependent)
- **Upload:** Usually automatic with auto-reset circuit
- **Strengths:** More GPIO, more memory, more features
- **USB:** Both S2 and S3 support native USB (require build flags)

### ESP32 (Original, 30-pin)

**Legacy board, still widely available:**
- **I2C pins:** SDA=21, SCL=22 (Arduino-compatible defaults)
- **UART serial:** TTL pins (not USB)
- **Upload:** External USB-to-Serial adapter required
- **Strengths:** Mature, well-documented, tons of examples
- **Weakness:** External serial adapter needed, more pins to wire

## Serial Monitor Troubleshooting

### No Output Appears

**Checklist:**
1. ✓ Build flags set: `-DARDUINO_USB_MODE=1 -DARDUINO_USB_CDC_ON_BOOT=1`
2. ✓ Serial.begin(115200) called in setup()
3. ✓ USB cable is data cable (some cables are charge-only)
4. ✓ Correct board selected in PlatformIO
5. ✓ Correct port selected in serial monitor (e.g., /dev/ttyUSB0 or COM3)

### Serial Output Garbled

**Solutions:**
1. Check baud rate: 115200 is standard (some boards use 74880)
2. Check board type: Ensure platformio.ini matches your board
3. Check build flags: Missing flags cause garbled output
4. Reset board: Press RESET button, monitor should reconnect

### Upload Fails

**Checklist:**
1. ✓ Board connected via USB
2. ✓ Correct board selected in platformio.ini
3. ✓ Correct port selected (should say "Uploading to ESP32-C3 over JTAG")
4. ✓ No other serial monitor connected (blocks upload)
5. ✓ For C2/C3: Try manual BOOT + RESET sequence

### Port Not Detected

**Try:**
1. Unplug USB, wait 2 seconds, plug back in
2. Check cable: Try different USB cable
3. Try different USB port on computer
4. Check drivers: Plug in board, check Device Manager (Windows) or System Report (Mac)
5. Update PlatformIO: May need latest drivers

## GPIO Pin Reference

### ESP32-C3 (Common Pins)

| Pin | Name | Uses |
|-----|------|------|
| 0 | GPIO0 | General purpose (BOOT during upload) |
| 1 | GPIO1 | General purpose, TX (Serial0) |
| 2 | GPIO2 | General purpose |
| 3 | GPIO3 | General purpose, RX (Serial0) |
| 4 | GPIO4 | General purpose |
| 5 | GPIO5 | General purpose, **SDA (I2C)** |
| 6 | GPIO6 | General purpose, **SCL (I2C)** |
| 7 | GPIO7 | General purpose |
| 8 | GPIO8 | LED pin, TX1 (Serial1) |
| 9 | GPIO9 | RX1 (Serial1) |
| 10 | GPIO10 | General purpose |

### ESP32-C2 Super Mini (Limited)

| Pin | Name | Uses |
|-----|------|------|
| 0 | GPIO0 | General purpose (BOOT during upload) |
| 1 | GPIO1 | TX (Serial0) |
| 2 | GPIO2 | General purpose |
| 3 | GPIO3 | RX (Serial0) |
| 4 | GPIO4 | General purpose |
| 5 | GPIO5 | **LED pin** |
| 20 | GPIO20 | **SDA (I2C)** |
| 21 | GPIO21 | **SCL (I2C)** |

## Complete Example: Minimal Setup with USB Serial

```cpp
#include <Arduino.h>

void setup() {
  // USB serial setup (requires build flags)
  Serial.begin(115200);
  delay(500); // Wait for serial to initialize
  
  // GPIO setup
  pinMode(8, OUTPUT); // LED for ESP32-C3
  
  // Print startup message
  Serial.println("=== ESP32 Started ===");
  Serial.print("Board: ");
  Serial.println(ARDUINO_BOARD);
  
  Serial.print("Free memory: ");
  Serial.print(ESP.getFreeHeap());
  Serial.println(" bytes");
}

uint32_t lastBlink = 0;

void loop() {
  // Blink LED every 500ms
  if (millis() - lastBlink >= 500) {
    lastBlink = millis();
    digitalWrite(8, !digitalRead(8));
    Serial.println("Blink!");
  }
  
  yield();
}
```

## Power and Reset Issues

### Board Won't Start

**Symptoms:** No output, no enumeration
**Solutions:**
1. Check USB power: Board should have power LED on
2. Check RESET button: May be stuck
3. Try fresh USB cable
4. Try different USB port on computer
5. Check for shorts: Inspect for loose wires, bent pins

### Spurious Resets (Watchdog Triggering)

**Symptoms:** Board resets every few seconds
**Causes:**
- Long blocking operations without yield()
- Missing esp_task_wdt_reset() calls
- Using delay() for long periods

**Fix:** Use EVERY_N_MILLIS + yield() + esp_task_wdt_reset()

## JTAG Programming (Advanced)

Some ESP32-C3 boards support JTAG programming for debugging:
- Requires additional hardware (JTAG adapter, usually ~$20)
- Enables breakpoint debugging, memory inspection
- Not needed for basic development
- Requires separate pins (MTDI, MTDO, MTCK, MTMS)

## Key Rules

1. **Set USB CDC build flags** — Required for serial output on C3
2. **Always specify I2C pins explicitly** — Default pins vary by board
3. **Press BOOT + RESET for upload on C2** — If auto-reset fails
4. **Check USB cable is data cable** — Charge-only cables won't work
5. **Call Serial.begin(115200) in setup()** — 115200 is standard baud
6. **Call yield() frequently** — Prevents watchdog reset
7. **Close serial monitor before uploading** — Blocks upload if connected
