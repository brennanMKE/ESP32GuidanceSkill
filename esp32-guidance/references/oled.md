# OLED Display with U8g2

## Hardware Quirk: Visible Area Offset on 0.42" Displays

The 0.42" OLED (SSD1306) has a critical quirk: the **physical display glass is smaller than the controller's memory buffer**.

### The Problem

- **Controller memory**: 128×64 pixels
- **Visible glass area**: ~72×40 pixels (centered on the left side)
- **Off-screen area**: Left ~28 pixels, right ~28 pixels, top/bottom margins

Drawing at (0, 0) writes to invisible memory, so text appears off-screen.

### The Solution

Offset text coordinates by approximately **28 pixels to the right** and **12 pixels down**:

```cpp
// Text appears at approximately (28, 12) not (0, 0)
u8g2.drawStr(28, 12, "Hello");  // ✓ Visible
u8g2.drawStr(0, 0, "Hello");    // ❌ Off-screen (invisible)
```

## I2C Pins and Initialization

### Pin Configuration

ESP32-C3 I2C pins vary. **Always use explicit pins:**

```cpp
#include <U8g2lib.h>
#include <Wire.h>

// Explicit pins: SDA=5, SCL=6 (ESP32-C3)
U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE, 6, 5); // SCL, SDA

void setup() {
  // Initialize with explicit pins
  Wire.begin(5, 6); // SDA=5, SCL=6
  
  u8g2.begin();
  u8g2.setFont(u8g2_font_ncenB08_tr); // Select a font
  
  // Clear display
  u8g2.clearBuffer();
  u8g2.drawStr(28, 28, "Hello World");
  u8g2.sendBuffer();
}
```

### Common I2C Pin Configurations

| Board | SDA | SCL |
|-------|-----|-----|
| ESP32-C3 DevKit | 5 | 6 |
| ESP32 DevKit | 21 | 22 |
| ESP32-C2 Super Mini | 20 | 21 |
| ESP32-S2 | 8 | 9 |
| ESP32-S3 | 8 | 9 |

## Complete OLED Setup Example

```cpp
#include <Arduino.h>
#include <U8g2lib.h>
#include <Wire.h>
#include <esp_log.h>

static const char* TAG = "OLED";

// Initialize with explicit I2C pins (SDA=5, SCL=6 for ESP32-C3)
U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE, 6, 5);

#define EVERY_N_MILLIS_ID(ID, N) \
  static uint32_t _timer_##ID = 0; \
  if (millis() - _timer_##ID >= (N)) { \
    _timer_##ID = millis(); \
    true; \
  } else { \
    false; \
  }

void setup() {
  Serial.begin(115200);
  delay(500);
  
  esp_log_level_set(TAG, ESP_LOG_INFO);
  
  // Initialize I2C with explicit pins
  Wire.begin(5, 6);
  
  // Initialize display
  u8g2.begin();
  u8g2.clearBuffer();
  u8g2.setFont(u8g2_font_ncenB08_tr);
  
  // Account for visible area offset
  u8g2.drawStr(28, 20, "OLED Ready");
  u8g2.sendBuffer();
  
  ESP_LOGI(TAG, "Display initialized");
  Serial.println("OLED display ready");
}

uint32_t counter = 0;

void loop() {
  // Update display every 100ms (non-blocking)
  if (EVERY_N_MILLIS_ID(display_update, 100)) {
    u8g2.clearBuffer();
    u8g2.setFont(u8g2_font_ncenB08_tr);
    
    // Draw text with offset for visible area
    u8g2.drawStr(28, 20, "Counter:");
    
    char buf[16];
    snprintf(buf, sizeof(buf), "%d", counter);
    u8g2.drawStr(28, 36, buf);
    
    u8g2.sendBuffer();
    counter++;
  }
  
  yield();
  esp_task_wdt_reset();
}
```

## Font Selection

U8g2 includes many fonts. Common choices:

```cpp
// Monospace (good for numbers)
u8g2.setFont(u8g2_font_ncenB08_tr);    // 8px fixed
u8g2.setFont(u8g2_font_5x8_tr);         // Tiny 5px

// Proportional (good for text)
u8g2_font_courB08_tr                   // Courier 8px
u8g2_font_fixed_v0_tr                  // Fixed-width

// For more fonts, see: https://github.com/olikraus/u8g2/wiki/fntlistall
```

## Display Modes and Refresh

### Single Refresh (No Buffering)

```cpp
void setup() {
  u8g2.begin();
  u8g2.setFont(u8g2_font_ncenB08_tr);
}

void loop() {
  // Direct draw (slow, flicker risk)
  u8g2.firstPage();
  do {
    u8g2.drawStr(28, 28, "Hello");
  } while (u8g2.nextPage());
  
  delay(100);
}
```

### Buffered Refresh (Recommended)

```cpp
void setup() {
  u8g2.begin();  // Enables buffer automatically
  u8g2.setFont(u8g2_font_ncenB08_tr);
}

void loop() {
  // Buffer and send (clean, no flicker)
  u8g2.clearBuffer();
  u8g2.drawStr(28, 28, "Hello");
  u8g2.sendBuffer();
  
  delay(100);
}
```

## Centering Text

Since you have the offset, you can calculate centering:

```cpp
// Visible area is ~72 pixels wide, centered at x=64 (middle of 128)
// So visible area spans from ~28 to ~100

void drawCenteredText(const char* text, uint8_t y) {
  u8g2.setFont(u8g2_font_ncenB08_tr);
  
  // Get text width
  u8g2_uint_t width = u8g2.getStrWidth(text);
  
  // Center in visible area (28 to 100, width ~72)
  // Center point is 64, so left offset is 64 - width/2
  u8g2_uint_t x = 64 - (width / 2);
  
  // Clamp to visible area if needed
  if (x < 28) x = 28;
  
  u8g2.drawStr(x, y, text);
}

void setup() {
  u8g2.begin();
}

void loop() {
  u8g2.clearBuffer();
  drawCenteredText("Hello World", 32);
  u8g2.sendBuffer();
}
```

## Troubleshooting

### Text Not Visible
- **Solution**: Add ~28 pixel X offset, ~12 pixel Y offset
- **Example**: Use (28, 20) instead of (0, 0)

### I2C Not Found
- **Check pins**: Verify SDA/SCL pins match your board
- **Check address**: Confirm I2C address is 0x3C (common for SSD1306)
- **Use I2C scanner**: Run a sketch that lists I2C devices

### Display Garbled or Flickering
- **Use buffering**: Call clearBuffer() + sendBuffer(), not firstPage()/nextPage()
- **Slow update rate**: Reduce update frequency to 50-100ms
- **Check power**: 3.3V and GND must be solid

### I2C Scanner Code

```cpp
void scanI2C() {
  for (uint8_t addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
      Serial.print("I2C device found at 0x");
      Serial.println(addr, HEX);
    }
  }
}

void setup() {
  Serial.begin(115200);
  Wire.begin(5, 6); // Your pins
  delay(500);
  
  Serial.println("Scanning I2C...");
  scanI2C();
}

void loop() {}
```

## Key Rules

1. **Offset text by ~28 pixels right, ~12 pixels down** — Account for visible area on 0.42" glass
2. **Use explicit I2C pins** — Wire.begin(SDA, SCL)
3. **Use buffered mode** — clearBuffer() + sendBuffer() for clean display
4. **Non-blocking updates** — Use EVERY_N_MILLIS for refresh, not delay()
5. **Always setFont() before drawing** — Font affects text positioning
6. **Check I2C address** — SSD1306 is typically 0x3C
7. **Wire.begin(5, 6) on ESP32-C3** — Use correct pins for your board
