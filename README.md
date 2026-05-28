# ESP32 Guidance Skill

A Claude Code skill that encodes proven ESP32 development patterns from real projects. Generates better ESP32 code from the start by preventing common pitfalls that consume development time.

**GitHub:** [brennanMKE/ESP32GuidanceSkill](https://github.com/brennanMKE/ESP32GuidanceSkill)

## What This Skill Does

This skill helps Claude generate correct, non-blocking ESP32 code aligned with your proven patterns for:

- **Non-blocking timing** — EVERY_N_MILLIS macro with unique IDs for multiple timers
- **WiFi connectivity** — Three-part setup sequence, eero mesh compatibility fix, non-blocking polling
- **Dual logging** — Serial (USB CDC) + ESP-IDF structured logging together
- **OLED displays** — I2C pins, visible area offset quirk, U8g2 initialization
- **Hardware-specific quirks** — USB CDC build flags, upload sequences, board differences
- **Project structure** — platformio.ini boilerplate, build configuration

## Who This Is For

- **ESP32-C3 developers** — Especially those working with WiFi mesh networks (eero) or I2C displays
- **PlatformIO users** — Those using Arduino framework with Espressif dependencies
- **Anyone tired of debugging** — Silent WiFi failures, off-screen OLED text, watchdog timeouts

## Installation

### Quick install (recommended) — `npx skills`

Installs across supported AI coding agents (Claude Code, Codex, Cursor, and others)
with one command:

```bash
npx skills add brennanMKE/ESP32GuidanceSkill --skill esp32-guidance
```

Target specific tools, or run non-interactively:

```bash
npx skills add brennanMKE/ESP32GuidanceSkill --skill esp32-guidance -a claude-code -a codex
npx skills add brennanMKE/ESP32GuidanceSkill --skill esp32-guidance -a claude-code -g -y
```

> **Claude Code note:** if the skill installs but Claude Code doesn't see it, it
> likely landed in `~/.agents/skills/` without a `~/.claude/skills/` symlink. Fix:
> ```bash
> ln -s ~/.agents/skills/esp32-guidance ~/.claude/skills/esp32-guidance
> ```

### Manual install — `install.sh`

Clone the repo and run the installer. It links the skill into the global skills dir
of every detected tool (Claude Code, Codex):

```bash
git clone https://github.com/brennanMKE/ESP32GuidanceSkill.git
cd ESP32GuidanceSkill
./install.sh
```

Cursor has no global skills directory, so install it per project:

```bash
./install.sh --project /path/to/your/project   # links into <project>/.cursor/skills
```

Install straight from git without a manual clone (caches and updates on re-run):

```bash
REPO_URL=https://github.com/brennanMKE/ESP32GuidanceSkill.git ./install.sh
```

Because `install.sh` uses symlinks, edits to the skill files are picked up without
reinstalling.

### Where skills live

| Tool        | Global               | Project            |
|-------------|----------------------|--------------------|
| Claude Code | `~/.claude/skills/`  | `.claude/skills/`  |
| Codex CLI   | `~/.codex/skills/`   | `.codex/skills/`   |
| Cursor      | *(project only)*     | `.cursor/skills/`  |

## Removal

```bash
rm ~/.claude/skills/esp32-guidance
rm ~/.codex/skills/esp32-guidance
rm /path/to/project/.cursor/skills/esp32-guidance
```

Removing a symlink leaves the source repo untouched.

## How It Works

The skill automatically loads when you:
- Work in a PlatformIO project with `platformio.ini` and `src/main.cpp`
- Write or review Arduino-style ESP32 code
- Reference Espressif dependencies

### What You Get

When the skill triggers, Claude will:
- Generate non-blocking code (EVERY_N_MILLIS, no `delay()` in loop)
- Use correct WiFi setup sequence (operation order matters!)
- Include dual logging (Serial + ESP-IDF together)
- Account for hardware quirks (OLED offsets, I2C pins, USB CDC flags)
- Apply proven patterns from real projects

### Example Triggers

**Generate ESP32-C3 code with WiFi and logging:**
> "I'm building a sensor logger for my ESP32-C3. It needs to read sensors every 500ms, log to serial and ESP-IDF every second, and attempt WiFi connection on startup. Use non-blocking patterns."

**Debug WiFi connection issues:**
> "My ESP32-C3 keeps timing out on my eero mesh network. Generate the WiFi setup code with the compatibility fix."

**Set up OLED display:**
> "I have a 0.42\" OLED on an ESP32-C3 with SDA=5, SCL=6. Generate code to initialize U8g2 and display 'Hello World' centered on the visible glass area."

## Core Rules

The skill enforces these critical patterns:

1. **Never use `delay()` in loop()** — Always use EVERY_N_MILLIS for non-blocking timing
2. **Operation order matters** — WiFi.mode() FIRST, then setTxPower, then esp_wifi_set_protocol
3. **Use dual logging** — Both Serial (USB CDC) and ESP-IDF together, never one alone
4. **Account for hardware quirks** — ESP32-C3 OLED visible area offset, I2C pins, USB CDC flags

## Reference Guides

Detailed guidance is organized by domain:

- **[timing.md](https://github.com/brennanMKE/ESP32GuidanceSkill/blob/main/esp32-guidance/references/timing.md)** — EVERY_N_MILLIS macro, multiple timers, watchdog reset
- **[wifi.md](https://github.com/brennanMKE/ESP32GuidanceSkill/blob/main/esp32-guidance/references/wifi.md)** — WiFi setup sequence, eero mesh fix, non-blocking polling
- **[logging.md](https://github.com/brennanMKE/ESP32GuidanceSkill/blob/main/esp32-guidance/references/logging.md)** — Dual logging setup, build flags, log levels
- **[oled.md](https://github.com/brennanMKE/ESP32GuidanceSkill/blob/main/esp32-guidance/references/oled.md)** — I2C pins, visible area offset quirk, U8g2 patterns
- **[hardware.md](https://github.com/brennanMKE/ESP32GuidanceSkill/blob/main/esp32-guidance/references/hardware.md)** — USB CDC, JTAG, upload sequence, board quirks
- **[project-setup.md](https://github.com/brennanMKE/ESP32GuidanceSkill/blob/main/esp32-guidance/references/project-setup.md)** — platformio.ini boilerplate, project structure

## Quick Start Example

### Generate a Complete Sensor Logger

Ask Claude Code (with skill active):
> "Generate a complete ESP32-C3 sketch that reads a temperature sensor every 500ms, logs to both Serial and ESP-IDF every second, attempts WiFi connection on startup, and uses non-blocking patterns throughout."

**What you'll get:**
- EVERY_N_MILLIS timers with unique IDs (500ms, 1000ms, WiFi polling)
- WiFi setup with correct operation order (mode → power → protocol)
- Dual logging (Serial.println + ESP_LOGI)
- Non-blocking WiFi polling with watchdog reset
- No `delay()` calls in loop()
- Proper includes and build flags

## Compatibility

- **Boards:** ESP32, ESP32-C2, ESP32-C3, ESP32-S2, ESP32-S3
- **Framework:** Arduino (via PlatformIO)
- **Build System:** PlatformIO
- **Dependencies:** Espressif32 platform

## Common Pitfalls Prevented

### ❌ Without Skill
```cpp
void setup() {
  WiFi.mode(WIFI_STA);
  delay(100);
  WiFi.setTxPower(WIFI_POWER_8_5dBm); // Silent failure - mode not set yet
  WiFi.begin(ssid, password);
}

void loop() {
  readSensor();
  delay(500); // Blocks everything, WiFi drops
  Serial.println(value);
}
```

### ✅ With Skill
```cpp
void setup() {
  WiFi.mode(WIFI_STA);
  WiFi.setTxPower(WIFI_POWER_8_5dBm); // Correct order
  esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_11B | WIFI_PROTOCOL_11G);
}

void loop() {
  if (EVERY_N_MILLIS_ID(sensor, 500)) {
    readSensor(); // Non-blocking
  }
  if (EVERY_N_MILLIS_ID(logging, 1000)) {
    Serial.println(value);
  }
  yield();
  esp_task_wdt_reset();
}
```

## Testing

The skill has been validated with 5 test cases:

| Test | Focus | Result |
|------|-------|--------|
| LED Matrix Timing | EVERY_N_MILLIS patterns | ✅ Non-blocking patterns |
| WiFi Eero Fix | Operation order, mesh compatibility | ✅ Correct sequence enforced |
| Dual Logging | Build flags, Serial + ESP-IDF | ✅ Both logging systems |
| OLED Display | Hardware offset quirk, I2C pins | ✅ Correct offset applied |
| Sensor Logger | Pattern integration | ✅ Coherent pattern composition |

## Contributing & Feedback

This skill encodes patterns from real ESP32 projects. Found a bug, missing pattern, or better way to do something?

### Report Issues or Suggest Improvements

1. **GitHub Issues:** [Create an issue](https://github.com/brennanMKE/ESP32GuidanceSkill/issues) to report bugs or suggest new patterns
2. **Pull Requests:** [Submit a PR](https://github.com/brennanMKE/ESP32GuidanceSkill/pulls) with improvements

### Adding New Patterns

1. Test your pattern in a real ESP32 project
2. Add or update the relevant reference file in `esp32-guidance/references/`
3. Update `esp32-guidance/SKILL.md` if needed
4. Create a pull request with your changes

### What Makes a Good Contribution

- **Proven in real projects** — Not theoretical, tested in actual ESP32 development
- **Solves a real problem** — Prevents debugging time or common mistakes
- **Well documented** — Includes before/after examples, explanations of why it matters

## License

This skill and its guidance are provided as-is for personal and educational use.

## Status

✅ **Ready for Use**

The skill successfully prevents common ESP32 debugging pitfalls, particularly:
- Silent WiFi failures due to operation order
- Off-screen OLED text from hardware offset quirks
- Blocking operations that trigger watchdog timeouts

Install and use in your projects to get better code from Claude Code immediately.
