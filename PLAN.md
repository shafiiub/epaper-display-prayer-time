# Prayer Time Display on TTGO T5 e-Paper

Planned: 2026-11-06

## Goal

Show today's five daily prayer times plus Sunrise and Sunset on the TTGO T5 e-paper
board. Refresh the screen every 5 minutes and highlight the next prayer.

## What's already in place

| Item | Status |
|---|---|
| Device in HA > ESPHome Device Builder | `ttgo-t5-epaper.yaml`, friendly name **TTGO-T5-EPaper**, online at `192.168.1.189` |
| Board / framework | `esp32dev`, ESP-IDF |
| Panel | LilyGO T5 V2.3.1 **2.13"**, `waveshare_epaper` model `2.13in-ttgo-b74`, `rotation: 270`, so **250 × 122 px landscape**, black/white. (The original config said `2.90in`; that driver never refreshed this panel.) |
| Pins | SPI CLK 18, MOSI 23, CS 5, DC 17, BUSY 4, RST 16 |
| Firmware | Deployed 2026.3.2, Device Builder is on 2026.10.1 (update pending) |
| HA location | Sydney, `Australia/Sydney` (uses HA home location) |
| Prayer data in HA | **None yet**: only `sun.sun` exists, and no Islamic Prayer Times integration |

## Architecture

```
Aladhan API ──> HA "Islamic Prayer Times" integration (core, daily)
                    │  sensor.*_fajr/sunrise/dhuhr/asr/maghrib/isha (UTC timestamps)
sun.sun ────────────┤  next_setting → Sunset
                    ▼
        HA template sensors (format to local "HH:MM", work out the next prayer)
                    │  native API (already encrypted and connected)
                    ▼
        ESPHome on TTGO T5: homeassistant text_sensors + time
                    │  display update_interval: 5 min
                    ▼
                2.9" e-paper
```

**Why HA does the formatting:** the integration reports UTC ISO timestamps.
Converting them to local time and picking the next prayer is a one-liner in Jinja.
In the ESPHome lambda it would mean parsing strings and handling the timezone in C++.
With HA doing the work, the device only prints short strings.

**Why Sunset comes from `sun.sun`:** the HA Islamic Prayer Times integration exposes
Fajr, Sunrise, Dhuhr, Asr, Maghrib, Isha and Midnight, but not Sunset.

## Steps

### 1. Add prayer-time data to Home Assistant
- Add the **Islamic Prayer Times** integration (Settings > Devices & services > Add).
  It uses HA's home location (Sydney).
- Set options: calculation method and Asr school (see *Decisions*).
- Note the real entity IDs it creates (for example `sensor.islamic_prayer_times_fajr_prayer`).

### 2. HA template sensors for the display
Create these as Template helpers (UI) or as a package:

| Sensor | Template (sketch) |
|---|---|
| `sensor.epd_fajr` … `sensor.epd_isha`, `sensor.epd_sunrise` | `{{ as_timestamp(states('sensor.<prayer>')) \| timestamp_custom('%H:%M') }}` |
| `sensor.epd_sunset` | `{{ as_timestamp(state_attr('sun.sun','next_setting')) \| timestamp_custom('%H:%M') }}` |
| `sensor.epd_next_prayer` | Name of the first of Fajr/Dhuhr/Asr/Maghrib/Isha later than `now()`; after Isha → `Fajr` |
| `sensor.epd_next_in` | Countdown such as `1h 20m` to that prayer |

Use 12-hour (`%-I:%M`) or 24-hour (`%H:%M`) format. See *Decisions*.

### 3. ESPHome YAML changes (`ttgo-t5-epaper.yaml`)
Keep the existing `esphome`, `esp32`, `api`, `ota`, `wifi`, `captive_portal` and `spi` blocks
exactly as they are. Add or replace the following:

```yaml
time:
  - platform: homeassistant
    id: ha_time

text_sensor:
  - { platform: homeassistant, id: t_fajr,    entity_id: sensor.epd_fajr }
  - { platform: homeassistant, id: t_sunrise, entity_id: sensor.epd_sunrise }
  - { platform: homeassistant, id: t_dhuhr,   entity_id: sensor.epd_dhuhr }
  - { platform: homeassistant, id: t_asr,     entity_id: sensor.epd_asr }
  - { platform: homeassistant, id: t_sunset,  entity_id: sensor.epd_sunset }
  - { platform: homeassistant, id: t_maghrib, entity_id: sensor.epd_maghrib }
  - { platform: homeassistant, id: t_isha,    entity_id: sensor.epd_isha }
  - { platform: homeassistant, id: t_next,    entity_id: sensor.epd_next_prayer }
  - { platform: homeassistant, id: t_next_in, entity_id: sensor.epd_next_in }

font:
  - { file: "gfonts://Roboto", id: f_small, size: 14 }
  - { file: { type: gfonts, family: Roboto, weight: 700 }, id: f_time, size: 20 }
  - { file: { type: gfonts, family: Roboto, weight: 700 }, id: f_head, size: 16 }

display:
  - platform: waveshare_epaper
    id: epd
    model: 2.90in
    cs_pin: 5
    dc_pin: 17
    busy_pin: 4
    reset_pin: 16
    rotation: 270
    update_interval: 300s        # refresh every 5 min
    full_update_every: 12        # full refresh hourly to clear ghosting
    lambda: |-
      // header: date + next prayer countdown
      // 2×4 grid of name/time cells; next prayer cell drawn inverted
```

Notes:
- Prayer data is pushed by HA as soon as it changes. The 5-minute timer only controls
  how often the panel redraws, so the e-ink isn't worn out and doesn't flicker.
- Add a `homeassistant` `on_value` trigger, or an `api` `on_client_connected` trigger,
  that calls `component.update: epd` once. This stops the first screen after boot
  from showing empty times.
- If a value is missing, draw `--:--` so the screen never goes blank.

### 4. Screen layout (originally drawn for 296 × 128; the code now scales to 250 × 122)

```
┌──────────────────────────────────────────────────┐
│ Fri 6 Nov 2026                  Asr in 1h 20m     │  header 0–22, f_head
├────────────────────────┬─────────────────────────┤
│ Fajr          04:18    │ Sunset        19:28     │
│ Sunrise       05:50    │ Maghrib       19:28     │  4 rows × ~25 px
│ Dhuhr         12:38    │ Isha          20:52     │  name f_small, time f_time
│█Asr███████████16:12████│ Updated       22:35     │  next prayer = inverted
└────────────────────────┴─────────────────────────┘
```
(The times above are placeholders.)

### 5. Build and deploy (HA > ESPHome Device Builder)
1. Save the new YAML with `devices/update_config` (or the editor in the UI).
2. **Validate**, then **Install > Wirelessly** (OTA to `192.168.1.189`). This also
   moves the device from firmware 2026.3.2 to 2026.10.1.
3. In HA, allow the device to call/read HA entities if prompted. Check the
   `homeassistant` text_sensors in the logs.

### 6. Verify
- Logs show all 9 text_sensors receiving values.
- The screen matches HA's times, and the next-prayer highlight moves after each adhan time.
- After midnight, the times roll over to the new day (the integration refreshes daily).
- Leave it running for 24 hours and check for ghosting. If needed, tune `full_update_every`.

## Decisions (defaults chosen; change if needed)

| Decision | Default | Alternatives |
|---|---|---|
| Calculation method | **Muslim World League** (widely used in Australia) | ISNA, Umm al-Qura, Egyptian, Karachi, or match your local mosque |
| Asr school | **Shafi** (standard) | Hanafi |
| Time format | **24-hour** | 12-hour without am/pm |
| Extras | Date + next-prayer countdown | Hijri date, Jumu'ah time on Fridays, battery % |

## Out of scope for v1
- Deep sleep or battery operation. The board stays on Wi-Fi powered by USB. Deep sleep
  could come later, waking every 5 minutes, but it adds HA reconnect time on each wake.
- Adhan audio and notifications.

## Status (2026-11-06): v1 deployed

- HA: added the **Islamic Prayer Times** integration with method `mwl`, school `shafi`.
  Its sensors are `sensor.islamic_prayer_times_{fajr_prayer,sunrise_time,dhuhr_prayer,asr_prayer,maghrib_prayer,isha_prayer}`.
- HA: added 9 Template helpers `sensor.epd_{fajr,sunrise,dhuhr,asr,sunset,maghrib,isha,next_prayer,next_in}`.
  A reference copy is in `homeassistant/prayer_display_templates.yaml`.
- HA: added the device to the ESPHome integration. It had been in Device Builder only, so it
  had no time sync and no HA states.
- Device: `ttgo-t5-epaper.yaml` in this repo is the exact config in Device Builder. Secrets are
  in Device Builder's `secrets.yaml` (`epd_api_key`, `epd_ota_password`, `epd_ap_ssid`,
  `epd_ap_password`, plus the existing `wifi_ssid`/`wifi_password`). Flashed over OTA with ESPHome 2026.10.1.
- Unused leftover secrets `epd_wifi_ssid` / `epd_wifi_password` can be deleted from `secrets.yaml`.
