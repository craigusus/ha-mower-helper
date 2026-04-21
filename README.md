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

## Secrets

Add these entries to your Home Assistant `secrets.yaml`:

```yaml
# Mobile app notify service — found under Settings → Devices & Services → Companion App
mower_notify_service: "notify.mobile_app_your_phone"

# Telegram — entity ID from your Telegram bot integration
telegram_notify_service: "telegram_bot.send_message_YOUR_CHAT_ID"

# Discord — webhook URL from a Discord channel
# Channel Settings → Integrations → Webhooks → New Webhook → Copy Webhook URL
discord_webhook_url: "https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN"

# Google Calendar entity ID
mower_calendar: "calendar.your_google_calendar"
```

---

## Optional services — what to comment out

If you don't use all of the above, comment out the relevant actions in `mower_helpers.yaml` or HA will log errors every time a notification fires.

| Service | Used in automations | What to comment out |
|---|---|---|
| Mobile app (push notifications) | 7 (Can I Mow?) | `action: !secret mower_notify_service` blocks in automations 7 and 7a |
| Telegram | 4, 7, 9, 10, 11, 12, 13, 14, 17 | Every `action: telegram_bot.send_message` block |
| Discord | 4, 7, 9, 10, 11, 12, 13, 14, 17 | Every `action: rest_command.discord_webhook` block |
| Google Calendar | 15, 16, 17 | Automations 15 and 16 entirely; the calendar trigger in 17 |

> If you remove all calendar automations (15, 16, 17), you can also remove `mower_calendar_event_time` and `mower_scheduled_event_time` from the `input_text` section, and the `rest_command` block at the top.

---

## How to start mowing

There are four ways to trigger a mow, each with different condition checks:

### 1. Automatic schedule
Fires at the configured start time on enabled mow days.

- Set the start time via `input_datetime.mower_start_time` in the UI
- Set the end time (retry cutoff) via `input_datetime.mower_end_time` in the UI (default 20:00)
- Toggle mow days via `input_boolean.mower_day_*` helpers in the UI (Mon–Sun)

Checks before starting:
- All weather conditions are met (see [Conditions](#conditions))
- Mower is docked and charging
- Mower is not already mowing or returning
- Holiday Mode is off
- Not already mowed today

A 1-minute delay is applied before sending the start command, to allow the Mammotion integration time to sync mow path data after connection is established.

### 2. Automatic retry
If conditions aren't met at the scheduled time, or if the mower aborts early, the mower retries **every 30 minutes until the configured end time** (`input_datetime.mower_end_time`, default 20:00). Same checks as the scheduled mow. Also retries calendar-triggered mows that couldn't start immediately. A 1-minute delay is also applied before each retry.

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
- Error code 1415 (Mammotion empty path error)

### 5. Can I Mow? (manual button)
A dashboard button that checks **all** conditions including weather forecast and drying. If clear, sends an actionable push notification to your phone and a persistent HA notification asking you to confirm before mowing starts. Tapping **Start Mowing** in the notification starts the mower; tapping **Cancel** or dismissing does nothing. If conditions aren't met, sends the block reason via push notification, persistent notification, Telegram, and Discord.

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
| Error code | Not 1415 (Mammotion empty path error) |

Wind speed thresholds assume **mph**. 4.5 mph ≈ 2 m/s (light breeze).

---

## Holiday Mode

Toggle `input_boolean.mower_holiday_mode` to pause all scheduled and calendar-triggered mowing. Mow Now and Can I Mow? are not affected.

When Holiday Mode is on, `sensor.mower_next_scheduled` displays "Holiday Mode" instead of the next scheduled time.

---

## Notifications (Telegram + Discord)

Both Telegram and Discord notifications are sent for all events. Each requires its own secret (see [Secrets](#secrets)).

| Event | Message |
|---|---|
| Left dock without reaching mowing state | Current state and battery |
| Mowing started | Battery, solar, wind, rain chance |
| Mowing aborted early (returned <5 min) | Duration and battery; retry will follow |
| Mowing completed | Time, battery, area covered, duration |
| Docked due to rain | Rain rate at time of return |
| Calendar mow delayed | Reason conditions weren't met |
| Can I Mow? blocked | Reason conditions weren't met |
| Error state | Error details and battery |
| Paused mid-mow | Progress and battery |

Discord messages use bold titles and line-separated fields. The webhook URL is stored as `discord_webhook_url` in `secrets.yaml`. To create a webhook: **Discord channel → Edit Channel → Integrations → Webhooks → New Webhook → Copy Webhook URL**.

---

## Calendar integration

Two types of events are written to the calendar configured via `!secret mower_calendar`:

| Event | When created |
|---|---|
| "Dave III Mowed" | When mowing completes — uses stored start time and actual end time |
| "Dave III - Mow Scheduled" | Next scheduled mow placeholder (2-hour window), updated on startup, midnight, after each mow, and when Holiday Mode is toggled |

> **Note:** `calendar.update_event` is not available in this HA instance. The mowing event is created once on completion with exact times rather than being updated from a placeholder.

> **Note:** The Google Calendar integration does not return a uid when creating events, so old "Mow Scheduled" placeholder events cannot be deleted programmatically. They become past events and do not affect any automation logic. Manual cleanup in Google Calendar is needed if the mow schedule changes.

---

## Dashboard sensors

Use these read-only template sensors on cards instead of the raw input helpers:

| Sensor | Example value | Description |
|---|---|---|
| `sensor.mower_next_scheduled` | `09:30 Tue 1 Apr` | Next scheduled mow time (human-readable); shows `Retrying — today until HH:MM` while retries are active |
| `sensor.mower_next_scheduled_iso` | `2026-04-10T09:30:00` | Next scheduled mow time (ISO format, for TimeFlow Card) |
| `sensor.mower_estimated_finish` | `2026-04-08T11:45:00` | Estimated mow finish time based on `sensor.dave_iii_time_left` (ISO format, only set while mowing) |
| `sensor.mower_last_completed_formatted` | `14:22 Sat 29 Mar` | Last completed mow time (human-readable) |
| `sensor.mower_last_completed_iso` | `2026-03-29T14:22:00` | Last completed mow time (ISO format, for TimeFlow Card) |
| `sensor.mower_last_rain_formatted` | `11:45 Fri 28 Mar` | Last detected rain time |
| `sensor.mower_block_reason` | `Post-rain lockout — 45 min ago, dry by 13:45` | Why mowing is currently blocked (or "OK to mow"); shows `Outside mowing hours` when outside the configured start/end window; shows `OK to mow — no RTK fix` when all conditions are met but GPS positioning is degraded |
| `binary_sensor.mower_ok_to_mow` | `on` / `off` | All conditions met or not |

---

## Known limitations (Mammotion integration)

The following are outside the control of this package and depend on the Mammotion integration's polling behaviour:

- **State reporting delay** — HA may detect the `mowing` state significantly later than the actual start. This affects `mower_started_at`, calendar event start times, and duration calculations in Telegram and Discord notifications.
- **`sensor.dave_iii_time_left`** — Can briefly drop to 0 between polling updates while mowing, causing `sensor.mower_estimated_finish` to flicker. Guarded by also checking the mower is in `mowing` state.
- **`binary_sensor.dave_iii_charging`** — If reported incorrectly, scheduled and calendar mows may silently fail their precondition check.
- **MowPathSaga empty path (error 1415)** — The Mammotion integration occasionally starts a mow with an empty zone/line list (`line hash list was empty — falling back to zone_hashs=[]`), causing the mower to leave the dock and return immediately. Diagnostics confirm the map data is present but `work.zone_hashs` is populated with zeros — a pymammotion library bug. The 1-minute pre-start delay is a workaround. All mowing automations also block if `sensor.dave_iii_last_error_code` is `1415` to prevent repeated failed attempts.

---

## Automations reference

| # | Alias | Purpose |
|---|---|---|
| 1 | Mower - Scheduled Mow | Starts at the configured time on mow days if all conditions are met; 1-minute delay before sending command |
| 2 | Mower - Retry When Conditions Improve | Retries every 30 min until 20:00 on mow days or when calendar mow requested; 1-minute delay before sending command |
| 3 | Mower - Track Last Rain | Records timestamp when rain sensor activates |
| 4 | Mower - Dock on Rain | Returns mower to dock immediately if rain detected while mowing |
| 5 | Mower - Mow Now | Manual override button — minimal condition checks |
| 6 | Mower - Track Completion | Records completion timestamp when docked only if mower ran >5 min today; clears calendar mow flag |
| 7 | Mower - Can I Mow Check | Weather-aware manual check — sends actionable notification to confirm, or notifies with block reason |
| 7a | Mower - Confirm Mow Action | Fires when user taps "Start Mowing" on the Can I Mow? notification; starts mower and clears persistent notification |
| 8 | Mower - Create Calendar Event | Records mow start time; sets calendar event start marker |
| 9 | Mower - Notify Left Dock Without Mowing | Telegram + Discord notification if mower leaves dock but never reaches mowing state |
| 10 | Mower - Notify Started | Telegram + Discord notification when mowing begins |
| 11 | Mower - Notify Completed | Telegram + Discord notification when mowing finishes (>5 min run, not already completed today) |
| 12 | Mower - Notify Aborted Early | Telegram + Discord notification when mower starts but returns within 5 min (GPS/boundary abort) |
| 13 | Mower - Notify Error | Telegram + Discord notification on error state |
| 14 | Mower - Notify Paused | Telegram + Discord notification when paused mid-mow |
| 15 | Mower - Update Calendar Event Duration | Creates accurate calendar event on completion (>5 min run only) |
| 16 | Mower - Sync Scheduled Calendar Placeholder | Keeps next scheduled mow on calendar; queries calendar via `calendar.get_events` to prevent duplicate placeholders |
| 17 | Mower - Calendar Triggered Mow | Fires on calendar events with "mow" in title (excluding "Dave III - Mow Scheduled") |
| 18 | Mower - Clear Calendar Mow Flag | Clears retry flag at midnight |
| 19 | Mower - Notify Blade Maintenance Due | Telegram + Discord alert once when blade time reaches the mower's warn threshold |
| 20 | Mower - Reset Blade Maintenance Flag | Clears the blade alert flag when blade time resets to near zero after replacement |
