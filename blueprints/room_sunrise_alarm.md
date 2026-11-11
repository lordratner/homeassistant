# Sunrise Alarm with Music

**Script blueprint**: `room_sunrise_alarm.yaml`.
Create one configured script per room; there are no Start/Cancel input buttons.
This replaces the earlier, undeployed automation-blueprint draft.

## Configuration versus calls

The script editor's blueprint configuration holds:

- Optional color-temperature light/group and optional speaker (at least one).
- Minimum, intermediate and final brightness and color temperature.
- Intermediate and final speaker volume. Minimum audio volume is always zero.
- Actions to replace white noise with wake-up music on the selected speaker.

The **only call-time field is `duration`**, with choices 1, 5 or 10 minutes.
No brightness, color temperature, volume, speaker or light overrides are exposed
at call time. Configuration is loaded within the sequence so incoming variables
cannot replace the configured room settings.

Example call after a room script has been created:

```yaml
- action: script.kids_room_sunrise
  data:
    duration: "5"
```

Use `script.turn_on` with `data.variables.duration` for a nonblocking launch.
The example entity is illustrative, not an installed script.
The caller starts the script the selected number of minutes before the desired
alarm time; there is no separate target-time field or time trigger.

## Five phases

| Phase | 1-minute lead-in | 5-minute lead-in | 10-minute lead-in |
| --- | --- | --- | --- |
| 1: Reset | Immediate minimum light/temperature; audio zero | Light transitions to minimum over 10 seconds while audio fades to zero over 60 seconds | Same as 5-minute |
| 2: Gentle rise | Music starts at zero; audio/light reach intermediate in 15 seconds, 3-second steps | Reach intermediate in 60 seconds, 10-second steps | Same as 5-minute |
| 3: Hold | 45 seconds | 3 minutes | 8 minutes |
| 4: Alarm time | Volume rises from intermediate to final over 60 seconds, 10-second steps; lighting holds | Same | Same |
| 5: Final lighting | Light rises from intermediate to final over 120 seconds, 10-second steps; volume holds | Same | Same |

Nominal total runtime is **4, 8 or 13 minutes**. A 10-minute call starting at
06:50 begins the final volume ramp at 07:00 and finishes lighting at 07:03.
Without audio or lighting, the remaining channel keeps the same timeline.

Each stepped command uses **phase start + (target − phase start) × progress**.
The white-noise starting volume is captured once, never re-read and decremented.
Light steps use native transitions while speaker steps use explicit commands.
Phase boundaries and step deadlines are anchored to the run's starting timestamp;
normal service-call overhead is absorbed by the remaining wait rather than added
to every delay. Overdue steps catch up; blocking integrations can still make the
physical result late. This is not a hard-real-time alarm scheduler.

Brightness is bounded to 1–100%, volume to 0–100%, and each configured color
temperature is clamped to the selected light's advertised limits. Choose a group
with compatible members. Missing color-temperature limits use 1000–10000 K as
fallback bounds; successful playback/lighting is not guaranteed for offline or
incompatible devices. Configure minimum ≤ intermediate ≤ final for rising ramps;
the arithmetic also supports intentionally decreasing targets.

## Music and independent rooms

The blueprint is independent of any music integration. Configure a short music
start sequence, e.g. `music_assistant.play_media`, pointing at the selected speaker.
It must replace the existing white noise, must not set volume, and should not
contain waits/delays. Empty music actions mean no new playlist is started.
With no selected speaker the music actions are skipped entirely.

Light and speaker work run in separate branches. Device/action errors are marked
in traces and allowed to continue so the other channel can proceed. Music actions
are awaited within the audio branch; slow playback startup can shorten the audio
rise/hold. Light timing remains independent within that phase.
No speech, display, room helpers, HVAC changes or Adaptive Lighting ownership
changes are introduced.

## Restart and stopping

Each room script uses `mode: restart`. Another call replaces that room's active
sequence; separate room scripts are independent. Do not point independent room
instances at the same hardware. Invalid calls are rejected before device actions,
but restart mode may already have stopped a previous run.

`script.turn_off` stops further actions **without** stopping accepted playback,
undoing a light transition or restoring prior settings. This redesign has no
automatic cancellation cleanup; callers needing cleanup must explicitly define
it. HA reload/restart also interrupts without cleanup or recovery. Normal
completion deliberately leaves music and lighting at their final levels.

## Migration and validation status

**Published for testing; not imported, deployed or run on devices.** Existing sunrise scripts,
room controls and morning-routine callers remain unchanged.
Import the published [blueprint](https://raw.githubusercontent.com/lordratner/homeassistant/main/blueprints/room_sunrise_alarm.yaml)
under **Settings → Automations & scenes → Blueprints**. This script blueprint cannot replace an
installed automation blueprint by re-import; create new per-room script instances
through HA's blueprint API/UI after approval, then migrate callers separately.
Keep the original live scripts/callers for rollback; do not run old and new paths
for the same room. No production migration is authorized by the local revision.

Local YAML round-trip/input checks and strict read-only HA template rendering
passed for the three durations with both devices, light only and speaker only,
including missing-attribute defaults. Invalid duration cases were rejected by the
guard expression. These are not HA script/blueprint schema acceptance, execution,
restart/stop, audible playback or device-transition tests. Import/schema acceptance
and per-room physical tests remain outstanding.

