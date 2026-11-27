# Configuration

The active config is `cfg/main.yml`. The **authoritative field-by-field overview** of the YAML shape (top-level keys, `ui`, widgets) lives in the **comment header at the top of `cfg/main.yml`** — read that before adding or editing keys. This page covers behaviour notes and examples that supplement it.

Related references:
- **Channels, aliases & math channels** — [live-channels.md](./live-channels.md)
- **Available widgets** — [widgets.md](./widgets.md)

## DBC layout (`cfg/dbc/`)

- **`cfg/dbc/official/`** — vendor- or OEM-style DBC sources only. That does not prove they are free of mistakes, but they are the set we treat as the primary reference when one exists.
- **`cfg/dbc/reversed/`** — DBCs from reverse engineering or auto-generation. Not all of them are tested.

## Vehicle profile

Which car the device is fitted to. Known ids: `mclaren`, `bmwG`, `cayman`, `praga`, `obd2`.

You normally do not have to set this. Left alone, the app recognises the car from its CAN traffic — usually within the first second of the ignition being on — and the status bar and session log then show which one it picked. Set it explicitly when you want the choice pinned: a car whose traffic is not recognised, a bench where several cars' data is present at once, or a recording you need to be certain used a particular profile. An explicit setting always wins over what the app would have detected.

What the choice controls:

- **Which readings are used.** Each car has its own set of known channels; only the matching car's are read.
- **Which diagnostics run.** Some cars have a tailored diagnostics set that reads more than the standard one — engine and gearbox values, and fault codes the standard request does not return on that car. Where a tailored set exists it is used instead of the generic one, never alongside it: only one diagnostic conversation runs on a bus at a time, so the two can never talk over each other.

Because recognition takes a moment, a car with tailored diagnostics starts on the generic set and switches over as soon as the car is identified. Nothing is lost — the switch happens well inside the first second, and no session data is affected. Pinning the car skips the switch entirely.

## Lap timer (`lapTimer` in YAML)

Tracks are loaded from `cfg/tracks/*.gpx`. The hub auto-detects the current track and publishes lap channels under `system.LapTimer.*` (`lapCount`, `bestLapS`, `lastLapS`, `rollingLapTimeS`, `distanceSnapped`, and the track polyline for the map widget).

GPS source priority is set via `channelAliases` — list your preferred source first (the built-in GPS), then fallbacks (RaceBox, browser GPS). The first source with a valid fix wins.

### Filtering logged channels (`lapTimer.dataLog.includeChannels`)

By default every decoded channel is included in session files. To reduce file size, you can limit what each source contributes:

```yaml
lapTimer:
  dataLog:
    includeChannels:
      racebox: nothing              # drop all racebox channels
      ttpms: "prefix:ttpms.LF_"    # only ttpms.LF_* channels
```

Keys match the `key` field in `ui.channelSources`. Values: `everything`, `nothing`, `"prefix:SomePrefix"`, or `"list:path.a,path.b"`. Channel aliases are always included regardless of this setting.

## Drivers (`drivers` in YAML)

Optional list of drivers shown in the status bar and recorded in session files. Each entry needs a `name`; `pfp` (profile image URL) is optional. Select a driver at `/driverSelect` or via `?selectDriver=` — the choice is not persisted across hub restarts.

## Screen brightness (`display` in YAML)

The built-in screen dims itself when the car is parked and brightens when it matters, so a device
left on in the paddock is not running its panel at full power all day.

Everything scales off **where the sun is** at the device's location, which is the only sense of
ambient light the device has: full brightness (100%) while the sun is up, down to 1% once it is
dark, changing gradually through twilight. Because it follows the sun rather than the clock, it
tracks the season and where you are: in Sofia the evening dimming runs from about 16:30 to 17:45
in November, but from about 20:30 to 21:45 in June. The location comes from GPS; before the first fix it uses the last place the
device had one, or the city of its timezone. It errs on the bright side — the device would rather
spend a little power than be unreadable in daylight.

On top of that curve:

- **Driving** — the full level for the light outside. The screen brightens the moment the car starts
  moving.
- **Parked** — 70% of it, so roughly 70% at midday, and at night it stays at the 1% minimum. It only dims after the car has
  been stopped for half a minute, so a red light, a grid slot or a pit stop never dims the screen on
  you. Losing GPS does not dim it either.
- **Parked with a menu open** — back to the full level, for as long as the menu is on screen. This
  is the "someone is standing at the device reading it" case, and it only counts on the device's own
  screen: a menu open on your phone's view of the dashboard changes nothing.
- **On battery** — at most 50%, and 70% of that when parked. This applies whenever the battery is
  running the device: unplugged, or plugged into a supply too weak to carry it. The full screen uses
  about a third of the device's power, so this gives you noticeably more time on the battery. Plug
  into a supply that carries the device and full brightness comes back. Brightness you set yourself
  with the slider is never capped.

Changes fade rather than stepping, quickly upward and slowly downward. The screen is never blanked
or switched off by this, and all three numbers are adjustable:

```yaml
display:
  brightness:
    auto: true        # false starts the device in manual mode, at the level the panel already shows
    dayPct: 100       # driving, in daylight
    nightPct: 1       # driving, at night (1% is the darkest glow)
    idleFactor: 0.7   # parked and unattended, as a fraction of the above
```

**Manual brightness.** The settings page on the dashboard has a brightness slider and an Auto switch.
Dragging the slider pins the screen at that level, immediately and regardless of driving or time of
day; switching Auto back on returns to the behaviour above. The choice is remembered across restarts,
so `auto` in the YAML only decides how a device behaves before anyone has touched the slider. The
slider follows your eyes rather than the panel's power: half way is about a quarter of the panel's
light, and the bottom of the slider (1%) is a faint glow meant for a dark road. It never switches the
screen off.

On a device with no built-in panel, or an external screen whose backlight cannot be controlled,
nothing happens.

## Data sources (`ui.channelSources`)

Each entry defines one data source and its channel namespace:

```yaml
ui:
  channelSources:
    - key: ttpms          # CAN source via DBC
      file: official/IZZE_TTPMS.dbc
      bus: can0
    - key: cayman
      file: porsche_718.dbc
      bus: can1
    - key: racebox        # Bluetooth BLE — no DBC needed
      bus: bluetooth-ble
```

- **`key`**: prefix for all channel paths from this source (e.g. `ttpms.LF_TTPMS_1.LF_TTPMS_P_GAUGE`). Used in widget `channel:` bindings and in `lapTimer.dataLog.includeChannels`.
- **`file`**: DBC file under `cfg/dbc/`. Required for CAN sources.
- **`bus`**: `can0`, `can1`, or `bluetooth-ble`.

**Forwarding decoded data to a CAN bus** (e.g. BLE TPMS → CAN):

```yaml
    - key: chineseBLETPMS
      bus: bluetooth-ble
      forward:
        toBus: can0
        outputFormatDbc: BLE_TPMS_forward.dbc
```

## Dashboard pages and widgets

Pages are defined under `ui.pages`. Each page has an `id` (its URL path, e.g. `/main`) and a list of widgets. `ui.globalLayout` holds Tailwind grid classes for the outer shell.

Each widget entry:
- **`id`**: unique within the page.
- **`widget`**: widget type (see [widgets.md](./widgets.md)).
- **`gridClass`**: Tailwind classes for placement and sizing.
- **`params`**: widget-specific settings (channel bindings, thresholds, etc.).
