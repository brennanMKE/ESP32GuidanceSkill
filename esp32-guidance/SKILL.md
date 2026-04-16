---
name: esp32-guidance
description: |
  Generate better ESP32 code aligned with proven patterns from real projects. Use this whenever working in a PlatformIO project with platformio.ini and src/main.cpp (especially with Espressif dependencies), or when the user is writing/reviewing Arduino-style ESP32 code. Helps with timing/threading (non-blocking patterns), WiFi setup and troubleshooting, serial/ESP-IDF logging, OLED displays, and hardware quirks. Primary use is generating better code from the start; secondary use is debugging existing projects (especially WiFi mesh compatibility issues).
compatibility: PlatformIO + Arduino framework (ESP32, ESP32-C2, ESP32-C3, ESP32-S2, ESP32-S3)
---

Generate better ESP32 code following patterns validated through real projects. Avoid common pitfalls that consume development time: blocking delays, silent WiFi failures, logging confusion, and hardware-specific quirks.

## Core Rules

1. **Never use `delay()` in loop()** — Always use `EVERY_N_MILLIS` macro for non-blocking timing
2. **Operation order matters** — WiFi.mode() FIRST, then setTxPower, then esp_wifi_set_protocol. Setting power before mode fails silently.
3. **Use dual logging** — Both Serial (human-readable) and ESP-IDF (structured) together, never one alone
4. **Account for hardware quirks** — ESP32-C3 OLED visible area offset, I2C pins, USB CDC flags, upload sequence

## When to Load References

Load relevant guidance based on what you're working on:

| Task | Reference File | When to Use |
|------|---|---|
| Non-blocking timing, periodic tasks | `references/timing.md` | Code uses delay() OR needs multiple timers |
| WiFi setup, eero mesh issues, connection | `references/wifi.md` | WiFi.begin() calls OR connection timeouts |
| Serial logging, USB CDC, ESP-IDF logs | `references/logging.md` | Serial setup OR logging output missing |
| OLED display, I2C, U8g2 initialization | `references/oled.md` | OLED display integration OR text positioning issues |
| USB modes, upload failures, board quirks | `references/hardware.md` | Upload failures OR serial not working |
| Project structure, build flags, boilerplate | `references/project-setup.md` | Starting new project OR configuration questions |

## How to Use This Skill

**For code generation:** Load references matching the domains you're implementing (timing + WiFi + logging, for example). Combine patterns from each reference.

**For debugging:** Load the reference that matches the problem area (e.g., WiFi connection → load `references/wifi.md`). Find the section describing the issue.

**For learning:** References include before/after examples, explanations of why patterns matter, and common anti-patterns to avoid.

## Critical Pitfalls to Avoid

❌ Using `delay()` in loop — causes watchdog timeouts, missed interrupts, WiFi drops  
❌ Setting WiFi power before WiFi.mode() — fails silently with no error  
❌ Using only Serial OR only ESP-IDF logging — use both together  
❌ Drawing OLED text at (0,0) — appears off-screen on 0.42" displays  
❌ Not feeding watchdog during long operations — triggers resets  
❌ Missing build flags for USB CDC — Serial won't output  

## References

Detailed guidance is organized by domain:

- `references/timing.md` — EVERY_N_MILLIS macro, multiple timers, watchdog reset
- `references/wifi.md` — WiFi setup sequence, eero mesh fix, non-blocking polling
- `references/logging.md` — Dual logging setup, build flags, ESP-IDF vs Serial
- `references/oled.md` — I2C pins, visible area offset quirk, U8g2 patterns
- `references/hardware.md` — USB CDC, JTAG, upload sequence, board quirks
- `references/project-setup.md` — platformio.ini boilerplate, project structure
