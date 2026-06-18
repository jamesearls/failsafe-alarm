# TODO

## Sleep-monitoring dashboard

Goal: prove the alarm is actually catching me during overnight sleep, not just my obvious motion. Plot `radar_presence`, `radar_distance`, and `radar_peak_energy` over a full night so I can see how often the radar drops me and for how long.

### Approach: Prometheus + Grafana

ESPHome has a built-in Prometheus exporter — add one line to `failsafe-alarm.yaml`:
```yaml
prometheus:
```
This exposes `/metrics` on the device. Point a Prometheus instance at `failsafe-alarm.local:80/metrics` with a 5s scrape interval.

### Hosting

Needs to be always-on and on the same LAN as the radar. Candidates:
- Existing NAS with Docker (Synology / Unraid / etc.) — drop the compose file in
- Spare Raspberry Pi (Pi 3 or newer) — best dedicated option
- Old laptop running Linux, suspend disabled

Not the right fit:
- Cloud (overkill, no need to expose anything)
- Daily-driver laptop (sleeps overnight)

### What to plot in Grafana

Most useful queries against the Prometheus series:
- `esphome_binary_sensor_value{name="Radar Presence"}` — the main one; gaps over the night are what I'm looking for
- `esphome_sensor_value{name="Peak Gate Energy"}` — drops here might correlate with lost-presence events
- `avg_over_time(esphome_binary_sensor_value{name="Radar Presence"}[8h])` — single-number "how much of last night did it hold detection". > 0.95 = solid; lower = problem
- Annotate with the configured alarm window so I can see whether detection held *through* the window specifically

### Stretch: detect false-cutoffs during the window

Synthetic alert: if `radar_presence` was 0 for more than `disappearance_delay` seconds during the configured window on any weekday, log an event. That's the signal that the alarm would have falsely silenced.

### Implementation order

1. Add `prometheus:` to `failsafe-alarm.yaml` and re-flash
2. Verify `curl http://failsafe-alarm.local/metrics` returns Prometheus-format data
3. Pick a host, install Docker
4. Write `docker-compose.yml` + `prometheus.yml`
5. Run for one night, build initial dashboard the next day
6. Tune disappearance delay / mounting based on what the data shows

## Hardware / mounting

- [ ] Try top-mount above the bed at 1.0–1.5m if practical — best geometry for breathing detection (chest movement is along the beam axis, not perpendicular)
- [ ] If keeping side-table mount, set Radar Disappearance Delay to 20–30s as insurance against breathing-gap dropouts
- [ ] Cover the board's red power LED with electrical tape / marker for total bedroom darkness
- [ ] Software-disable the GPIO8 user LED — add `status_led` with the pin held low at boot

## Speculative — only if blocked

- [ ] Read the Waveshare Windows host tool's UART exchange (USB packet capture) to find the real sensitivity write command for the HMMD chip. Both candidates we tried (`SetConfig` params `0x30+` and LD2410 cmd `0x64`) ACK as success but don't change behavior. The host tool clearly *can* set sensitivity — its protocol bytes would reveal how.
- [ ] Try the Auto-Calibrate Radar button (cmd `0x09`) — leave the room, press, wait 15s. Untested; effect on stillness detection unknown.
