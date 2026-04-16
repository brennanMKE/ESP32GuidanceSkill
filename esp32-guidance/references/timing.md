# Non-Blocking Timing Patterns

## Core Problem

Using `delay()` in `loop()` blocks the entire thread, causing:
- Watchdog timeouts (task watchdog triggered)
- Missed WiFi packets and connection drops
- Unresponsive serial input
- Skipped interrupt handlers
- All concurrent tasks starve while blocking

## Solution: EVERY_N_MILLIS Macro

The EVERY_N_MILLIS pattern uses a counter and timestamp to make decisions without blocking.

### Basic Pattern

```cpp
#define EVERY_N_MILLIS(N) \
  static uint32_t _timer = 0; \
  if (millis() - _timer >= (N)) { \
    _timer = millis(); \
    true; \
  } else { \
    false; \
  }

void loop() {
  // Non-blocking updates
  if (EVERY_N_MILLIS(100)) {
    // Runs every 100ms
    updateSensors();
  }
  
  if (EVERY_N_MILLIS(1000)) {
    // Runs every 1 second
    logData();
  }
  
  // Rest of loop continues to run every iteration
  handleWiFi();
  yield(); // Feed watchdog
}
```

### Multiple Timers (Unique IDs)

When using EVERY_N_MILLIS multiple times, each needs a **unique static variable**. Use unique IDs:

```cpp
#define EVERY_N_MILLIS_ID(ID, N) \
  static uint32_t _timer_##ID = 0; \
  if (millis() - _timer_##ID >= (N)) { \
    _timer_##ID = millis(); \
    true; \
  } else { \
    false; \
  }

void loop() {
  if (EVERY_N_MILLIS_ID(sensor_read, 500)) {
    readSensors(); // Every 500ms
  }
  
  if (EVERY_N_MILLIS_ID(logging, 1000)) {
    logToSerial(); // Every 1 second
  }
  
  if (EVERY_N_MILLIS_ID(wifi_check, 5000)) {
    checkWiFiStatus(); // Every 5 seconds
  }
  
  yield(); // Always feed watchdog
}
```

### Why Unique IDs Matter

Without unique IDs, all timers share the same static variable:
```cpp
// ❌ WRONG - All share same _timer
if (EVERY_N_MILLIS(500)) { updateA(); }
if (EVERY_N_MILLIS(1000)) { updateB(); } // Won't work correctly
```

With unique IDs, each timer has its own counter:
```cpp
// ✓ CORRECT - Each has independent _timer_*
if (EVERY_N_MILLIS_ID(a, 500)) { updateA(); }
if (EVERY_N_MILLIS_ID(b, 1000)) { updateB(); }
```

## Watchdog Management

ESP32 has a task watchdog timer. Any task that doesn't yield for >5 seconds triggers a reset.

### Reset the Watchdog

Call in long-running operations or after blocking code:
```cpp
void setup() {
  Serial.begin(115200);
  
  // Initialize WiFi (may take time)
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  // Reset watchdog after blocking operation
  esp_task_wdt_reset();
}

void loop() {
  if (EVERY_N_MILLIS_ID(long_task, 10000)) {
    // Long-running operation
    performExpensiveCalculation();
    esp_task_wdt_reset(); // Reset after expensive work
  }
  
  // Also feed watchdog by yielding frequently
  yield();
}
```

### yield() vs esp_task_wdt_reset()

- `yield()`: Gives other tasks a chance to run, indirectly feeds watchdog
- `esp_task_wdt_reset()`: Explicitly resets watchdog timer
- **Use both** - yield() for normal operation, esp_task_wdt_reset() after known blocking sections

## Complete Example: Multi-Rate Timer

```cpp
#define EVERY_N_MILLIS_ID(ID, N) \
  static uint32_t _timer_##ID = 0; \
  if (millis() - _timer_##ID >= (N)) { \
    _timer_##ID = millis(); \
    true; \
  } else { \
    false; \
  }

uint16_t sensorValue = 0;

void setup() {
  Serial.begin(115200);
  pinMode(A0, INPUT);
}

void loop() {
  // Update sensor every 500ms
  if (EVERY_N_MILLIS_ID(sensor_read, 500)) {
    sensorValue = analogRead(A0);
  }
  
  // Log every 2 seconds
  if (EVERY_N_MILLIS_ID(logging, 2000)) {
    Serial.print("Sensor: ");
    Serial.println(sensorValue);
  }
  
  // Blink LED every 1 second
  if (EVERY_N_MILLIS_ID(blink, 1000)) {
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));
  }
  
  // Always yield to feed watchdog
  yield();
}
```

## Comparison: Before and After

### ❌ Blocking Approach (Problems)
```cpp
void loop() {
  readSensor();
  delay(100); // BLOCKS HERE - WiFi drops, watchdog at risk
  logData();
  delay(1000); // BLOCKS HERE again
}
```

### ✓ Non-Blocking Approach (Works)
```cpp
void loop() {
  if (EVERY_N_MILLIS_ID(read, 100)) {
    readSensor();
  }
  
  if (EVERY_N_MILLIS_ID(log, 1000)) {
    logData();
  }
  
  // Rest of code always runs
  handleWiFi();
  yield();
}
```

## Key Rules

1. **Never use `delay()` in loop()** — Always use EVERY_N_MILLIS for timing
2. **Each timer needs a unique ID** — Prevents collisions
3. **Always call `yield()`** — Feeds watchdog, lets other tasks run
4. **Call `esp_task_wdt_reset()` after blocking operations** — Explicit watchdog reset
5. **Test without blocking** — Verify WiFi stays connected during your timers
