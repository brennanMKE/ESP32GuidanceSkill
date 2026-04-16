# WiFi Setup and Mesh Compatibility

## Critical: Operation Order Matters

WiFi configuration has a **strict order requirement**. Setting power before mode fails silently with no error.

### The Three-Part Sequence

**This order is critical. Do not deviate:**

1. **WiFi.mode(WIFI_STA)** — Set station mode FIRST
2. **WiFi.setTxPower(WIFI_POWER_8_5dBm)** — Set transmit power second
3. **esp_wifi_set_protocol()** — Configure protocol compatibility third

```cpp
#include <esp_wifi.h>

void setupWiFi() {
  // Step 1: Set mode first
  WiFi.mode(WIFI_STA);
  
  // Step 2: Set TX power
  WiFi.setTxPower(WIFI_POWER_8_5dBm);
  
  // Step 3: Configure protocol
  // Disable 802.11n (b/g only) for mesh compatibility
  uint8_t protocol = WIFI_PROTOCOL_11B | WIFI_PROTOCOL_11G;
  esp_wifi_set_protocol(WIFI_IF_STA, protocol);
}
```

### Why Order Matters

Setting TX power before mode() appears to succeed but fails silently:
```cpp
// ❌ WRONG ORDER - Fails silently
WiFi.setTxPower(WIFI_POWER_8_5dBm); // No error, but doesn't work
WiFi.mode(WIFI_STA);
// Result: Default TX power is used, connection fails on mesh networks
```

```cpp
// ✓ CORRECT ORDER
WiFi.mode(WIFI_STA); // Set mode first
WiFi.setTxPower(WIFI_POWER_8_5dBm); // Then set power
// Result: Power setting applies correctly
```

## Eero Mesh WiFi Compatibility Fix

ESP32-C3 boards often fail to connect to eero mesh networks. The fix combines lower TX power with disabled 802.11n:

```cpp
#include <WiFi.h>
#include <esp_wifi.h>

const char* ssid = "your_ssid";
const char* password = "your_password";

void setupWiFi() {
  // Critical sequence
  WiFi.mode(WIFI_STA);
  WiFi.setTxPower(WIFI_POWER_8_5dBm); // Reduced power
  
  // Disable 802.11n (use b/g only)
  uint8_t protocol = WIFI_PROTOCOL_11B | WIFI_PROTOCOL_11G;
  esp_wifi_set_protocol(WIFI_IF_STA, protocol);
  
  // Now connect
  WiFi.begin(ssid, password);
  
  Serial.println("WiFi connecting...");
}
```

### Why This Works

- **Lower TX power (8.5dBm)**: Reduces noise, improves reliability on mesh networks
- **Disable 802.11n**: Eero mesh may have compatibility issues with 802.11n; falling back to 11b/g is more stable
- **Order first, then connect**: Ensures settings apply before WiFi negotiation begins

## Non-Blocking WiFi Connection

Never use `delay()` while connecting. Use polling with timeout protection:

```cpp
#define EVERY_N_MILLIS_ID(ID, N) \
  static uint32_t _timer_##ID = 0; \
  if (millis() - _timer_##ID >= (N)) { \
    _timer_##ID = millis(); \
    true; \
  } else { \
    false; \
  }

#define WIFI_CONNECT_TIMEOUT 30000 // 30 seconds

void setup() {
  Serial.begin(115200);
  delay(500); // Only delay needed: serial startup
  
  setupWiFi();
}

void setupWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.setTxPower(WIFI_POWER_8_5dBm);
  uint8_t protocol = WIFI_PROTOCOL_11B | WIFI_PROTOCOL_11G;
  esp_wifi_set_protocol(WIFI_IF_STA, protocol);
  
  WiFi.begin("ssid", "password");
  Serial.println("Waiting for WiFi...");
}

uint32_t wifiStartTime = 0;
bool wifiConnected = false;

void loop() {
  // Poll WiFi connection status
  if (EVERY_N_MILLIS_ID(wifi_poll, 500)) {
    if (WiFi.status() == WL_CONNECTED && !wifiConnected) {
      wifiConnected = true;
      Serial.print("WiFi connected! IP: ");
      Serial.println(WiFi.localIP());
    } else if (WiFi.status() != WL_CONNECTED && wifiConnected) {
      wifiConnected = false;
      Serial.println("WiFi disconnected");
    }
  }
  
  // Timeout protection: if unable to connect, log and continue
  if (!wifiConnected && wifiStartTime == 0) {
    wifiStartTime = millis();
  }
  
  if (!wifiConnected && (millis() - wifiStartTime) > WIFI_CONNECT_TIMEOUT) {
    Serial.println("WiFi connection timeout - continuing without WiFi");
    wifiStartTime = 0; // Reset for potential retry
  }
  
  // Always feed watchdog
  yield();
  esp_task_wdt_reset();
}
```

## Complete WiFi Setup Example

```cpp
#include <WiFi.h>
#include <esp_wifi.h>

const char* ssid = "your_ssid";
const char* password = "your_password";

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
  
  // Critical sequence - order matters!
  WiFi.mode(WIFI_STA);
  WiFi.setTxPower(WIFI_POWER_8_5dBm);
  
  uint8_t protocol = WIFI_PROTOCOL_11B | WIFI_PROTOCOL_11G;
  esp_wifi_set_protocol(WIFI_IF_STA, protocol);
  
  WiFi.begin(ssid, password);
  
  Serial.println("WiFi setup complete, polling for connection...");
}

bool wifiConnected = false;

void loop() {
  // Non-blocking WiFi polling
  if (EVERY_N_MILLIS_ID(wifi_check, 1000)) {
    if (WiFi.status() == WL_CONNECTED && !wifiConnected) {
      wifiConnected = true;
      Serial.print("Connected! IP: ");
      Serial.println(WiFi.localIP());
    } else if (WiFi.status() != WL_CONNECTED && wifiConnected) {
      wifiConnected = false;
      Serial.println("Disconnected from WiFi");
    }
  }
  
  // Do other work while WiFi connects
  if (EVERY_N_MILLIS_ID(work, 500)) {
    Serial.println("Doing other work...");
  }
  
  yield();
  esp_task_wdt_reset();
}
```

## TX Power Reference

ESP32 TX power levels:
- `WIFI_POWER_19_5dBm` = 19.5 dBm (highest, can cause mesh issues)
- `WIFI_POWER_17dBm` = 17 dBm
- `WIFI_POWER_15dBm` = 15 dBm
- `WIFI_POWER_13dBm` = 13 dBm
- `WIFI_POWER_11dBm` = 11 dBm
- `WIFI_POWER_8_5dBm` = 8.5 dBm (recommended for mesh networks)
- `WIFI_POWER_7dBm` = 7 dBm
- `WIFI_POWER_5dBm` = 5 dBm (lowest)

For eero and other mesh networks, `WIFI_POWER_8_5dBm` is the sweet spot.

## Debugging WiFi Issues

```cpp
void debugWiFi() {
  Serial.print("WiFi Status: ");
  Serial.println(WiFi.status());
  
  // 0 = WL_IDLE_STATUS
  // 1 = WL_NO_SSID_AVAIL
  // 2 = WL_SCAN_COMPLETED
  // 3 = WL_CONNECTED
  // 4 = WL_CONNECT_FAILED
  // 5 = WL_CONNECTION_LOST
  // 6 = WL_DISCONNECTED
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.print("SSID: ");
    Serial.println(WiFi.SSID());
    Serial.print("RSSI (signal strength): ");
    Serial.println(WiFi.RSSI());
  }
}
```

## Key Rules

1. **WiFi.mode() FIRST** — Must come before any other WiFi configuration
2. **setTxPower() second** — Sets TX power after mode is set
3. **esp_wifi_set_protocol() third** — Configure protocol last
4. **Never use delay() while connecting** — Use polling instead
5. **Include <esp_wifi.h>** — Required for esp_wifi_set_protocol()
6. **Use WIFI_POWER_8_5dBm for mesh networks** — Eero and similar benefit from lower TX power
7. **Feed watchdog** — Call yield() and esp_task_wdt_reset() frequently
