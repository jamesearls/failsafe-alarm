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
| Radar `OT2` | `IO5` EXT1 pin 8 | Presence signal (HIGH = detected) |
| Radar `TX` | `IO6` EXT1 pin 9 | Radar → ESP (UART config) |
| Radar `RX` | `IO3` EXT1 pin 6 | ESP → Radar (UART config) |
| Buzzer red (VCC) | `+5V` EXT1 pin 1 | Buzzer power |
| Buzzer black (GND) | `GND` EXT1 pin 2 | Ground |
| Buzzer yellow (Signal) | `IO4` EXT1 pin 7 | PWM tone output |

**Buzzer lead colours:** Red = VCC, Black = GND, Yellow = Signal. Verify against the silkscreen on the DFR0032 board before powering.

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
| **Debug Mode** switch | Makes the buzzer pip whenever presence is detected, ignoring the time window. Use this to calibrate the radar while mounted. Volume is reduced in this mode so you're not deafened. |
| **Test Buzzer** button | Sounds the full alarm for 3 seconds at the current volume setting. |
| **Apply Radar Config** button | Sends range and delay settings to the radar over UART. Press after changing the gate or delay sliders. |
| **Alarm Volume** slider | Loudness of the real alarm (0–100%). |
| **Window Start Hour / Minute** | When the alarm window opens each weekday. |
| **Window End Hour / Minute** | When the alarm window closes each weekday. |
| **Radar Max Gate** slider | Detection range. Each gate ≈ 0.70m. Set this so the radar just reaches the bed surface but not the rest of the room. Start at 2 (≈1.4m) and increase if it doesn't detect you lying down. |
| **Radar Disappearance Delay** slider | How many seconds after you leave before the radar reports absence. 1s = fastest shutoff. Increase if the alarm cuts out while you're still in bed. |
| **Radar Presence** indicator | Live detection state — ON/OFF. |
| **Radar Distance** sensor | Distance to detected target in cm (0 = nobody there). |
| **Alarm Status** text | `idle` / `WINDOW ACTIVE` / `DEBUG (calibration)` / `no time sync`. |
| **Uptime** | Seconds since last boot. Resets to 0 on power loss. |
| **Wi-Fi Signal** | Signal strength in dBm. |

## Calibrating the radar

The radar needs to see the bed but not the whole room. The default range (gate 2, ≈1.4m) assumes the sensor is mounted roughly 1–1.5m above the mattress.

1. Mount the radar pointing straight down at your sleeping position.
2. Turn on **Debug Mode** in the web UI.
3. Lie in your normal sleeping position — you should hear quiet pip beeps and see **Radar Presence = ON**.
4. Get up and walk out — beeps should stop within 1–2 seconds.
5. If it doesn't detect you lying down, increase **Radar Max Gate** by 1 and press **Apply Radar Config**.
6. If it detects you from across the room, decrease **Radar Max Gate** and press **Apply**.
7. Turn **Debug Mode** off when done.

## Testing the alarm without waiting for 08:30

Use the **Window Start / End** dropdowns to set the window to 2 minutes from now. Lie in bed — the full alarm should fire when the window opens. Get up to silence it. Then restore the real times.

## Changing the alarm time

Use the **Window Start Hour**, **Window Start Minute**, **Window End Hour**, **Window End Minute** dropdowns in the web UI. Changes take effect within one minute — no reflash needed.

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
- **Alarm fires on weekends:** Check the window — if the window is open you may have the dropdowns set to non-weekday behaviour. The firmware enforces Mon–Fri internally.
- **Device not reachable:** If Wi-Fi is down, the device brings up a fallback access point called `failsafe-fallback`. Connect to it and open http://192.168.4.1 to reconfigure Wi-Fi.
- **Radar detects all the time:** Radar Max Gate is probably too high. Reduce it and press Apply Radar Config.
- **Alarm doesn't fire when in bed:** Radar Max Gate might be too low. Increase it and press Apply.
- **Time shows 1970 in logs:** Normal on first boot before SNTP syncs. Give it 10 seconds after Wi-Fi connects.
