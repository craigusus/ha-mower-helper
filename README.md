# Dave III Mower Package

Home Assistant package for the Mammotion Yuka Mini robotic mower (Dave III). All helpers, template sensors, and automations are defined in a single file: `mower_helpers.yaml`.

## Setup

Add to `configuration.yaml` as a package:

```yaml
homeassistant:
  packages:
    mower: !include mower/mower_helpers.yaml
```

Keep `automation: !include automations.yaml` in `configuration.yaml` as normal — HA merges the automation list from this package automatically. Although, they cannot be edited from the UI, because they are not stored in the `automations.yaml` file

---

## How to start mowing

There are four ways to trigger a mow, each with different condition checks:

### 1. Automatic schedule
Fires at **09:30 on Tuesday, Thursday, and Sunday**.

Checks before starting:
- All weather conditions are met (see [Conditions](#conditions))
- Mower is docked and charging
- Holiday Mode is off
- Not already mowed today

To change the schedule, edit the weekday lists in automations 1, 2, and 15 and the time in automation 1.
```
Python weekdays: Mon=0, Tue=1, Wed=2, Thu=3, Fri=4, Sat=5, Sun=6
```

### 2. Automatic retry
If conditions aren't met at 09:30, the mower retries **every 30 minutes until 20:00**. Same checks as the scheduled mow. Also retries calendar-triggered mows that couldn't start immediately.

### 3. Calendar trigger
Create a Google Calendar event with **"mow" anywhere in the title** (e.g. "Mow garden", "Dave III mow") at the time you want it to start.

- Checks all weather conditions and battery
- If conditions are met, starts immediately
- If not, sends a Telegram notification and retries every 30 minutes until 20:00
- Blocked by Holiday Mode
- Won't run if already mowed today
- The retry flag is cleared automatically at midnight

### 4. Mow Now (manual button)
A dashboard button that starts mowing immediately, bypassing the schedule, weather forecast, drying, and soil checks.

Still blocked by:
- Active rain
- Post-rain lockout (within 2 hours of last rain)
- Low battery (≤ 20%)

### 5. Can I Mow? (manual button)
A dashboard button that checks **all** conditions including weather forecast and drying. If clear, starts mowing. If not, sends a Telegram message explaining exactly why.

---

## Conditions

All automated and calendar triggers check these before mowing:

| Condition | Threshold |
|---|---|
| Not raining | Rain sensor = off |
| Rain rate | < 0.5 mm/h |
| Post-rain lockout | ≥ 2 hours since last rain |
| Rain forecast | < 50% precipitation probability (Met Office) |
| Drying (solar) | > 100 W/m², **or** |
| Drying (wind) | > 4.5 mph |
| Humidity | < 88% |
| Soil moisture | < 70% (`sensor.gw2000a_soil_moisture_3`) |
| Battery | > 20% |

Wind speed thresholds assume **mph**. 4.5 mph ≈ 2 m/s (light breeze).

---

## Holiday Mode

Toggle `input_boolean.mower_holiday_mode` to pause all scheduled and calendar-triggered mowing. Mow Now and Can I Mow? are not affected.

When Holiday Mode is on, `sensor.mower_next_scheduled` displays "Holiday Mode" instead of the next scheduled time.

---

## Notifications (Telegram)

Telegram messages are sent for:

| Event | Message |
|---|---|
| Left dock without mowing | Current state and battery |
| Mowing started | Battery, solar, wind, rain chance |
| Mowing completed | Time, battery, area covered, duration |
| Docked due to rain | Rain rate at time of return |
| Calendar mow delayed | Reason conditions weren't met |
| Can I Mow? blocked | Reason conditions aren't met |
| Error state | Error details and battery |
| Paused mid-mow | Progress and battery |

---

## Calendar integration

Two types of events are written to `calendar.craig_fews_gmail_com`:

| Event | When created |
|---|---|
| "Dave III Mowing" | When mowing completes — accurate start/end times |
| "Dave III - Mow Scheduled" | Next scheduled mow placeholder (2-hour window), updated on startup, midnight, after each mow, and when Holiday Mode is toggled |

> Note: `calendar.update_event` is not available in this HA instance. The mowing event is created once on completion with exact times rather than being updated from a placeholder.

---

## Dashboard sensors

Use these read-only template sensors on cards instead of the raw input helpers:

| Sensor | Example value | Description |
|---|---|---|
| `sensor.mower_next_scheduled` | `09:30 Tue 1 Apr` | Next scheduled mow time |
| `sensor.mower_last_completed_formatted` | `14:22 Sat 29 Mar` | Last completed mow time |
| `sensor.mower_last_rain_formatted` | `11:45 Fri 28 Mar` | Last detected rain time |
| `sensor.mower_block_reason` | `Post-rain lockout — 45 min ago, dry by 13:45` | Why mowing is currently blocked (or "OK to mow") |
| `binary_sensor.mower_ok_to_mow` | `on` / `off` | All conditions met or not |

---

## Automations reference

| # | Alias | Purpose |
|---|---|---|
| 1 | Mower - Scheduled Mow | Starts at 09:30 on mow days if conditions are met |
| 2 | Mower - Retry When Conditions Improve | Retries every 30 min until 20:00 on mow days or when calendar mow requested |
| 3 | Mower - Track Last Rain | Records timestamp when rain sensor activates |
| 4 | Mower - Dock on Rain | Returns mower to dock immediately if rain detected while mowing |
| 5 | Mower - Mow Now | Manual override button — minimal condition checks |
| 6 | Mower - Track Completion | Records completion timestamp; clears calendar mow flag |
| 7 | Mower - Can I Mow Check | Weather-aware manual check — starts or sends Telegram reason |
| 8 | Mower - Create Calendar Event | Records mow start time for use by automation 14 |
| 9 | Mower - Notify Left Dock Without Mowing | Telegram notification if mower leaves dock but doesn't start mowing |
| 10 | Mower - Notify Started | Telegram notification when mowing begins |
| 11 | Mower - Notify Completed | Telegram notification when mowing finishes |
| 12 | Mower - Notify Error | Telegram notification on error state |
| 13 | Mower - Notify Paused | Telegram notification when paused mid-mow |
| 14 | Mower - Update Calendar Event Duration | Creates accurate calendar event on completion |
| 15 | Mower - Sync Scheduled Calendar Placeholder | Keeps next scheduled mow on calendar |
| 16 | Mower - Calendar Triggered Mow | Fires on calendar events with "mow" in title |
| 17 | Mower - Clear Calendar Mow Flag | Clears retry flag at midnight |
