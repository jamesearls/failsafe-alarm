# Failsafe Alarm

A "get out of bed" alarm that runs entirely on a small Wi-Fi microcontroller. A 24GHz radar above your bed detects whether you're there. During a configured morning window on weekdays, if the radar sees you it sounds a buzzer. The **only** way to stop it is to physically get out of bed — there is no snooze, no button, no app.

## How it works

1. On boot the device connects to Wi-Fi and syncs the time (Europe/London, BST/GMT handled automatically).
2. Every weekday morning within the configured window, it checks: is someone in bed?
3. If yes → buzzer sounds continuously until the person leaves.
4. The moment the radar stops detecting presence, the buzzer stops.
5. Outside the window it does nothing.

## Hardware

| Part | Notes |
|------|-------|
| Olimex ESP32-C3-DevKit-Lipo | The brains. Powered + flashed over USB-C. |
| Waveshare HMMD mmWave Sensor (S3KM1110) | 24GHz radar. 3.3V only — never connect to 5V. |
| DFRobot Gravity Digital Buzzer (DFR0032) | Passive piezo module. Powered at 5V for max volume. |
| DuPont jumper wires | M-F etc. |

## Wiring

| From | To (Olimex) | Purpose |
|------|-------------|---------|
| Radar `3V3` | `+3V3` EXT2 pin 1 | Radar power — 3.3V only |
| Radar `GND` | `GND` EXT2 pin 2 | Ground |
| Radar `OT2` | `IO5` EXT1 pin 8 | Presence signal — **drives the alarm** |
| Radar `TX` | `IO7` EXT1 pin 10 | Monitoring stream (ASCII presence + distance) |
| Radar `RX` | `IO3` EXT1 pin 6 | **Optional** — only needed for binary Report Mode with energy data |
| Buzzer red (VCC) | `+5V` EXT1 pin 1 | Buzzer power |
| Buzzer black (GND) | `GND` EXT1 pin 2 | Ground |
| Buzzer yellow (Signal) | `IO4` EXT1 pin 7 | PWM tone output |

**Buzzer lead colours:** Red = VCC, Black = GND, Yellow = Signal. Verify against the silkscreen on the DFR0032 board before powering.

**OT2 is the safety guarantee.** The alarm uses the OT2 hardware pin, not UART, so monitoring data being absent doesn't affect the alarm itself.

## Setup

### First time

1. Copy `secrets.yaml` and fill in your Wi-Fi credentials and generate an API key:
   ```bash
   cp secrets.yaml.example secrets.yaml   # edit with your details
   openssl rand -base64 32                # paste output as api_key
   ```
2. Install ESPHome: `pipx install esphome`
3. Flash over USB-C (first time only):
   ```bash
   esphome run failsafe-alarm.yaml
   ```
   The device shows up as `/dev/ttyACM0` on Linux.

### After first flash — OTA updates

All future updates go over Wi-Fi:
```bash
esphome run failsafe-alarm.yaml --device failsafe-alarm.local
```

## Web interface

Open **http://failsafe-alarm.local** in a browser on the same network. You'll find:

| Control | What it does |
|---------|-------------|
| **Debug Mode** switch | Makes the buzzer pip whenever presence is detected, ignoring the time window. Use this to calibrate the radar while mounted. Volume is reduced so you're not deafened. |
| **Test Buzzer** button | Sounds the full alarm for 3 seconds at the current volume setting. |
| **Apply Radar Config** button | Pushes Radar Max Gate + Radar Disappearance Delay to the radar over UART, and re-asserts Report Mode. Requires Radar RX wired to IO3. |
| **Auto-Calibrate Radar** button | Triggers the radar's self-calibration (cmd `0x09`). Leave the room first — the radar samples background noise for ~10s and tunes its noise floor. Speculative for HMMD; effect not verified. |
| **Alarm Volume** slider | Loudness of the real alarm (0–100%). |
| **Window Start** time | When the alarm window opens each weekday (HH:MM picker). |
| **Window End** time | When the alarm window closes each weekday (HH:MM picker). |
| **Radar Max Gate** slider | Limits the OT2 hardware detection range. Each gate ≈ 0.70m. Requires Radar RX wired + **Apply Radar Config** pressed. |
| **Radar Disappearance Delay** slider | Seconds of no-detection before the radar reports absence. Higher = more reliable stillness detection, slower alarm cutoff. Requires Radar RX wired + **Apply Radar Config** pressed. |
| **Radar Presence** indicator | OT2 hardware pin: ON/OFF. This is what triggers the alarm. |
| **Radar Distance** sensor | Distance to detected target in cm. In ASCII mode this is gate × 70cm; in binary Report Mode it's the radar's actual cm reading. |
| **Target Type** text | `present` / `absent` as reported over UART. |
| **Peak Gate Energy** sensor | Strongest gate signal across gates 0..Max Gate. Only updates in binary Report Mode (Radar RX wire required). |
| **Alarm Status** text | `idle` / `WINDOW ACTIVE` / `DEBUG (calibration)` / `no time sync`. |
| **Uptime** | Seconds since last boot. Resets to 0 on power loss. |
| **Wi-Fi Signal** | Signal strength in dBm. |

## Monitoring modes

The radar starts in **ASCII Normal Mode** and outputs lines like `ON\r\nRange 4\r\n` continuously. The firmware parses this for the **Radar Distance** and **Target Type** sensors. This works with only the Radar TX wire connected to IO7.

To get per-gate energy data, connect Radar RX to IO3 and press **Apply Radar Config**. The radar will switch to binary frames and **Peak Gate Energy** will start updating. The parser auto-detects whichever format is arriving.

## Calibrating the radar

The radar's detection range is limited via the **Radar Max Gate** slider (requires Radar RX wired). Each gate ≈ 0.70m, so gate 2 ≈ 1.4m, gate 3 ≈ 2.1m. Without RX wired, the radar operates at its full default range (~8.5m).

1. Mount the radar — ideally pointing straight down at your sleeping position from 1–1.5m above. Side-table mounting works for the alarm logic but breathing detection while motionless is harder from that angle (chest movement is perpendicular to the beam).
2. Turn on **Debug Mode** in the web UI.
3. Lie in your normal sleeping position — you should hear quiet pip beeps and see **Radar Presence = ON**.
4. Get up and walk out — beeps should stop after the **Radar Disappearance Delay** elapses (default 5s).
5. If it triggers from across the room: reduce **Radar Max Gate** and press **Apply Radar Config**.
6. If it cuts out while you're still in bed: increase **Radar Disappearance Delay** (try 20s) and press **Apply Radar Config**.
7. Turn **Debug Mode** off when done.

## Testing the alarm without waiting for 08:30

Use the **Window Start** / **Window End** time pickers to set the window to 2 minutes from now. Lie in bed — the full alarm should fire when the window opens. Get up to silence it. Then restore the real times.

## Changing the alarm time

Use the **Window Start** and **Window End** time pickers in the web UI. Changes take effect within one minute — no reflash needed.

## Timezone and DST

Configured for Europe/London (Belfast). GMT↔BST is handled automatically. The device re-syncs time from NTP daily so clock changes are always applied correctly.

## Secrets

`secrets.yaml` is excluded from git (see `.gitignore`). Never commit it. It contains:

```yaml
wifi_ssid: "your network name"
wifi_password: "your password"
fallback_ap_password: "used if main Wi-Fi is unreachable"
ota_password: "used for OTA firmware updates"
api_key: "32-byte base64 key for the ESPHome API"
```

## If something goes wrong

- **Buzzer sounds constantly at boot:** Debug Mode may have been left on. Open the web UI and turn it off.
- **Alarm fires on weekends:** The firmware enforces Mon–Fri internally. If you see this, check `Alarm Status` and the day-of-week of the device clock.
- **Device not reachable:** If Wi-Fi is down, the device brings up a fallback access point called `failsafe-fallback`. Connect to it and open http://192.168.4.1 to reconfigure Wi-Fi.
- **Radar detects continuously across the whole room:** Reduce **Radar Max Gate** to 2 or 3 and press **Apply Radar Config** (requires Radar RX wired to IO3). Without RX, the only option is to physically tighten the radar's beam coverage.
- **Doesn't reliably detect stillness:** Increase **Radar Disappearance Delay** (try 15–30s) and press **Apply Radar Config**. The radar's micro-motion sensitivity itself isn't user-configurable on this chip — undocumented commands are silently ignored.
- **Radar Distance / Target Type stay NA:** The Radar TX wire isn't reaching IO7, or the radar's UART output is silent. Reseat the wire and power-cycle the whole device (unplug USB-C completely for 5+ seconds).
- **Peak Gate Energy is NA:** Expected unless Radar RX is wired to IO3 and the radar is in Report Mode (press **Send Report Mode Command**).
- **Time shows 1970 in logs:** Normal on first boot before SNTP syncs. Give it 10 seconds after Wi-Fi connects.
