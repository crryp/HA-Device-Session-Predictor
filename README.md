# Device Session Predictor

> Built for [Home Assistant](https://www.home-assistant.io/) — this is a Home Assistant package (YAML), not a standalone app. You need a working Home Assistant instance to use it.
>
> *(Nederlandse versie: [README.nl.md](README.nl.md))*

Estimates how much longer a "dumb" appliance (washing machine, dishwasher,
dryer, ...) still needs to run — based purely on a power sensor (Watts). No
smart connection to the appliance itself required, just an energy meter /
smart plug that reports current power draw.

Built and tested on a washing machine with a HomeWizard Energy Socket, but
works with any sensor that reports a power value in Watts.

## How it works

1. **Session detection**: power above one threshold = "on", power below a
   different (lower) threshold = "off". Both require the reading to hold
   continuously for a minimum duration, so brief spikes/dips between
   program phases don't get mistaken for a real start/end.
2. **Checkpoints during the session**: cumulative energy use is tracked at
   two speeds:
   - **Fine**: every 10 minutes, up to 6x (covers the first hour) — usually
     captures the most distinctive phase of a program (e.g. the heating
     phase).
   - **Coarse**: every 20 minutes, up to 9x (covers up to 3 hours) — needed
     to tell long and medium-length programs apart, something the fine
     curve alone isn't good at (it stops after the first hour).

   The fine and coarse curve lists are kept strictly 1:1 (one slot per
   stored curve, same order). A run too short to log any coarse checkpoint
   stores a literal `-` placeholder so the slot still exists — the matcher
   pairs `curves_coarse[i]` with `curves_fine[i]` directly.
3. **At session end**, the duration, energy use, and curve are saved: the
   curve into a scarce history (default: 8 curves), the plain
   duration/energy values into a roomier log (last 20).
4. **During a new session**, the sensor compares the curve-so-far against
   all stored curves, and uses the closest match to estimate the remaining
   time. The distance is the mean absolute error per series (fine and
   coarse each normalised by their own overlap count, then averaged), so a
   curve scored on both fine and coarse points isn't penalised against one
   scored on fewer points. If the session runs longer than the best match
   predicted (with a 5-minute grace period), the estimate falls back to the
   second-best match. The matched curve's "last used" timestamp is refreshed, so
   curves you actually rely on don't get evicted (see point 6).
   `sensor.device_match_curve` reports which curve is currently the best
   match, for display on a dashboard.
5. **Duplicate detection**: a session that closely resembles an
   already-stored one in both duration (±5 min) and energy use (±10%, or
   ±0.05 kWh absolute) is not stored as a new curve — this keeps the
   limited 8 curve slots from filling up with 8 copies of the same
   program. On a duplicate, the existing curve is only replaced if the new
   curve is *more complete* (more checkpoints relative to what the
   duration would lead you to expect), and its "last used" timestamp is
   bumped.
6. **Eviction when full**: once all 8 curve slots are used and a genuinely
   new curve comes in, one slot is overwritten instead of the oldest one
   simply dropping off. The shortest- and longest-duration curves are
   always protected (they anchor the range of programs the predictor
   knows about); among the rest, the least-recently-used curve — oldest
   "last used" timestamp, set on creation and refreshed on every match —
   is the one that gets dropped.

## Known limitations

- **The first few sessions are inaccurate.** With no history, the estimate
  falls back to a fixed 90 minutes. Curve matching only becomes meaningful
  after a few sessions with some variety.
- **255-character limit.** The curve storage uses `input_text` helpers (HA
  limit: 255 characters). That bounds how many checkpoints × how many
  sessions you can keep. The defaults (6 fine + 9 coarse, 8 curves) still
  fit, but with less headroom than a 6-curve history — if you extend this,
  keep an eye on the limit.
- **Programs without a clear heating phase** (e.g. a cold cycle) may be
  harder to distinguish, since much of the fine curve's discriminating
  power comes from that heating spike.
- **You need to find the right thresholds yourself.** There's no universal
  start/end threshold that works for every appliance — measure what a
  reliable signal looks like for yours (see installation step 3 below).

## Files in this repo

| File | What |
|---|---|
| `packages/device_session_predictor.yaml` | **Required.** The core: session detection, checkpoints, curve matching, remaining-time sensor. |
| `packages/device_ready_notification.yaml` | **Optional.** Push notification + timed light signal once a session finishes, then a deterministic restore (your resting scene if the light should be on, otherwise off). |
| `dashboard-card.yaml` | Compact Mushroom card (power + remaining time) that links through to the detail subview. |
| `dashboard-subview.yaml` | Full-page detail subview: status, live chart, per-day session counts, curve history, raw data. |

---

## Installation — option A: as a package (recommended)

A *package* is a single YAML file where helpers, automations, and sensors
live together, instead of being created individually through the UI. That
makes this project "copy, fill in a few lines, restart" instead of dozens
of separate UI steps.

### Step 1 — Make sure Home Assistant loads packages

Check whether your `configuration.yaml` already has this (often present by
default):

```yaml
homeassistant:
  packages: !include_dir_named packages
```

If not, add it. Create a `packages/` folder next to your
`configuration.yaml` if it doesn't exist yet.

### Step 2 — Copy the main file

Place `packages/device_session_predictor.yaml` into your `config/packages/`
folder.

### Step 3 — Fill in the placeholders

Open the file and search for `FILL_IN` and `ADJUST` — at minimum you need
to:

1. **Your power sensor**: replace every `sensor.FILL_IN_YOUR_POWER_SENSOR`
   with the entity_id of your sensor that reports current power in Watts.
2. **Start threshold**: the `above:` value in the first automation — how
   many Watts reliably indicates "a program is running" (not standby or
   menu navigation)?
3. **Start duration** (`for:`): how long must that hold continuously
   before you trust it as a real start?
4. **End threshold**: the `below:` value in the second automation —
   ideally as close as possible to your appliance's true "idle" level. The
   closer to the real zero point, the smaller the chance a pause between
   program phases gets mistaken for "done".
5. **End duration** (`for:`): how long must that hold continuously?

**Tip for finding your thresholds**: look at your power sensor's history
graph during a full program cycle. Pay attention to:
- How high the power draw is, at minimum, during the "genuinely active"
  phases (that becomes your start threshold).
- How low the power can dip between phases without the appliance actually
  being done (your end threshold needs to sit below that, and/or the end
  duration needs to be longer than the longest "normal" pause).

### Step 4 — Restart Home Assistant

Packages load most reliably with a full restart (Settings → System →
Restart). "Reload YAML" sometimes works too, but a restart is safer for a
brand-new package.

### Step 5 — Verify

Go to Settings → Devices & Services → Entities, search for "Device" — you
should see all the helpers, the energy sensor, and the remaining-time
sensor. Go to Settings → Automations and confirm the two new automations
are there and enabled.

### Step 6 (optional) — Ready notification + light

Want a push notification (and optionally a light signal) once a session is
done? Repeat steps 2-4 with `packages/device_ready_notification.yaml`,
filling in your own notify service, the signal light, and a resting scene
for that light (what it should look like whenever it is normally on). The
signal colour is held for up to 20 minutes (or until you tap the
notification), then the light is put back deterministically — resting
scene if it should be on, otherwise off. Remove the light steps if you
only want the notification.

---

## Installation — option B: via the UI (step by step)

Not a fan of packages, or prefer to click through the UI instead? That
works too — it just takes more clicks. Create the following helpers via
**Settings → Devices & Services → Helpers → Add Helper**:

| Type | Name |
|---|---|
| Toggle | Device session active |
| Date and time | Device session start |
| Text (max 255) | Device session durations |
| Text (max 255) | Device session kwh |
| Text (max 255) | Device curve durations |
| Text (max 255) | Device curve kwh |
| Text (max 255) | Device curve last used |
| Text (max 255) | Device session curves |
| Text (max 255) | Device session curves coarse |
| Text (max 255) | Device current curve |
| Text (max 255) | Device current curve coarse |
| Counter | Device session counter |

Also create:
- An **Integration helper** ("Riemann sum") with your power sensor as the
  source, unit prefix "k", unit time "h" — this gives you cumulative kWh.
- A **Utility Meter helper** with the integration sensor you just created
  as the source, cycle "none".
- Two **Template sensors** using the `value_template` blocks from
  `packages/device_session_predictor.yaml`: `device_remaining_time` (the
  ETA) and `device_match_curve` (a short label of which stored curve the
  ETA is currently based on, e.g. "Curve #3 · ~125 min · 0.63 kWh", or
  "No session" when idle).
- *(Optional)* Seven **History stats helpers** — "count" type, entity
  `input_boolean.device_session_active`, tracked state `on` — one per day,
  each with its start/end window shifted a day further back. These feed a
  "sessions per day" dashboard chart and are not used by the prediction.
  Copy the `start:`/`end:` templates from the `history_stats` sensors in
  `packages/device_session_predictor.yaml`.

Then create the two automations via **Settings → Automations → Add
Automation → Edit in YAML**, and paste in the contents of the corresponding
`automation:` items from the package file (create one automation at a
time).

Note: with this route you'll need to make sure the entity names you chose
for the helpers match what's referenced in the automations and the sensor
template (Home Assistant usually auto-generates the entity_id from the
name you enter, e.g. "Device session active" →
`input_boolean.device_session_active`).

---

## Dashboard

Two files, both pasted in by hand (dashboards aren't part of the YAML
package):

- **`dashboard-card.yaml`** — a compact
  [Mushroom](https://github.com/piitaya/lovelace-mushroom) card (needs the
  Mushroom custom card via HACS) for your main dashboard. Shows power draw
  and, during a session, the remaining time; tapping it navigates to the
  detail subview. Add it via the card editor ("Manual" / YAML mode).
- **`dashboard-subview.yaml`** — the full-page subview the card links to:
  live status (including a "currently matching" tile that only shows while
  a session runs), a session chart, per-day session counts, the
  stored-curve history table, and a raw-data dump. Add it as a new view in
  your dashboard's raw configuration editor. The "Live progress" chart
  uses apexcharts-card (HACS); a built-in `history-graph` fallback is
  noted inline. Give the view's `path:` and the card's `navigation_path`
  the same value.

---

## Adjusting margins and settings

All the "magic numbers" are deliberately kept loose in the automation file,
with explanations in the comments:

- Checkpoint interval and count (fine/coarse)
- Duplicate margins (±5 min duration, ±10%/±0.05 kWh)
- Fallback grace period (5 minutes before switching to the second-best
  match)
- How many sessions/curves are kept (default 20 for the plain
  duration/kWh log, 8 for the curve slots)
- Curve eviction strategy — protect the shortest + longest curve, then
  drop the least-recently-used of the rest (the `evict_index` variable in
  the session-end automation)

Feel free to adjust these to fit your appliance and usage — there's no
universally "correct" number here, these are all rules of thumb that
worked well during development on one specific washing machine.

## License

Do whatever you want with it.
