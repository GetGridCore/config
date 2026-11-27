# Live channels

The dashboard and data logs read **numeric and text channels**. Each channel id looks like **`namespace.rest`** — for example **`mclarenlife.ECM_EngineSpeed_131.nEngine`** or **`system.gps.latitudeDeg`**.

**`channelAliases`** (in config) gives short names (`rpm`, `gpsLat`, …) that expand to **comma-separated lists** of those ids; the **first path that produces a valid value** wins.

An entry in that list can also be a **value with an `=` in front of it** instead of a channel id:

- A fixed number — **`=100`**, **`=-12.5`**.
- A channel scaled and/or offset — **`=somechannel*9`**, **`=somechannel*9-1`**, **`=somechannel+5`**, **`=somechannel-3`**. Supported shapes are *channel × number*, *channel × number ± number*, and *channel ± number*. There is no division and no parentheses.

This is handy when one car reports a value in different units or with a different offset than another: list each car's real channel, plus an `=` conversion for the odd one out, and the first entry that gives a valid value wins as usual.

An entry can also be **another alias name**, which stands for that alias's whole list.

### Math channels

An `=` entry converts **one** channel. When a value has to be **worked out from several channels** — brake bias from the front and rear brake pressures, for example — it is a **math channel**, set up under **`mathChannels`** in config. Each one appears as **`math.<name>`** and is normally shown and logged through an alias.

A math channel is a **percentage share**:

```yaml
mathChannels:
  myBrakeBias:
    percent: brakePressureFront                    # this…
    of: [brakePressureFront, brakePressureRear]    # …as a % of the sum of these
    onlyWhen: { channel: brakePressureFront, atLeast: 10 }   # optional
```

- **`percent`** and **`of`** take anything a widget `channel:` takes: aliases (with their `=` conversions already applied), channel ids, or a fixed number like **`"=120"`** — so "front pressure as a % of 120 bar" is `percent: brakePressureFront`, `of: "=120"`.
- **`onlyWhen`** (optional): while that channel reads below **`atLeast`**, the value is **not updated and keeps showing its last reading**.
- It updates the moment its inputs do. Readings that arrive together (the same message from the car or sensor box) are always combined with each other.
- Nothing is shown until every input has a reading. An input that reads **below zero** — typically an unplugged sensor — is ignored, and the value keeps its last reading rather than jumping to nonsense.
- A math channel cannot use another math channel as an input.

---

---

## Sources declared in config (`ui.channelSources`)

Most vehicle and sensor data comes from rows in **`cfg/main.yml`** → **`ui.channelSources`**:

| Kind | Typical `bus` | Notes |
| --- | --- | --- |
| CAN + DBC | `can0`, `can1000`, … | Channel ids: `<sourceKey>.<Message>.<signal>` from the DBC |
| OBD-II / UDS | Same CAN bus | No `file`; see [obd2-channel-source.md](./obd2-channel-source.md) |
| Bluetooth BLE | `bluetooth-ble` | TPMS, RaceBox, … — **must** have a YAML row so the hub knows to scan and decode |

Full shapes and options are documented in the **`cfg/main.yml`** header comment block.

---

## Hub-internal channels (`system.*`)

These are **not** `channelSources` entries. The running hub fills **`system.*`** when the corresponding feature is active (Linux GPS runtime, lap timer, browser location, …).

### Session clock

| Channel | Meaning |
| --- | --- |
| **`system.sessionTime`** | Seconds since process start (monotonic-based), stable session clock |
| **`system.currentVehicle`** | Active vehicle profile id when set (`VEHICLE=` at launch, or future in-dash picker). Logged as a session marker when it changes. Empty when unset. |

### Lap timer (`system.LapTimer.*`)

Numeric channels cover lap count, best/last lap times (seconds), rolling lap time, distance along track (real/snapped), delta estimate, and related quantities. Text channels hold track selection and serialized helpers (track outline JSON, lap times JSON, candidate list). **`system.LapTimer.currentTrack`** is a common string channel for logging. **`system.LapTimer.closedCircuit`** says what kind of track is loaded: **`1`** a closed circuit, **`0`** a point-to-point course, **`-1`** none.

### Fuel (`system.fuel.*`)

From the fuel level and the lap counter. Numbers are in the fuel level's own unit (% on most cars); **`-1`** means not known yet — two completed laps are needed — or not on a closed circuit, where laps are not a stint.

| Channel | Meaning |
|--------|---------|
| **`system.fuel.perLap`** | Fuel used per lap, averaged over the last few laps (`fuel.predictionLapWindow`, default 3) |
| **`system.fuel.lapsLeft`** | Whole laps the tank lasts at that rate, minus the reserve laps (`fuel.reserveLaps`, default 2) |
| **`system.fuel.lapsLeftCalc`** | The same at the fixed consumption set in `fuel.calculatedLapConsumption`; absent when that is not set |

### Auto-upshift (`system.autoUpshift.*`) — Praga R1T only

If the steering wheel stops talking to the car, the paddles stop working. On a Praga R1T, GridCore can then shift up for you at your usual shift points, as long as you stay at full throttle. It never shifts down. The control bar shows a **gear with an "A"**. It is **gray** when the car is recognised and the feature is standing by, and **green** while GridCore is ready to shift for you. On any other car the icon never appears.

It switches on when:

- **the steering wheel goes silent** (about 0.2 s without it). It switches off the moment the wheel is back.
- **the engine keeps hitting the rev limiter** (a second limiter cut within a second of the first, about 0.1 s after you first touch it) with the wheel working. This is how you test it: bounce off the limiter in any gear below 6th and GridCore shifts up on the second bounce. It stays on, shifting up again at the next shift point, until you press any paddle or button on the wheel. After that it waits for you to come off the limiter before it can switch on again.

It asks for an upshift only once per gear. If the car does not change gear, it tries one more time 2 s later and then leaves that gear alone.

| Channel | Meaning |
| --- | --- |
| **`system.autoUpshift.state`** | `1` standing by (gray icon), `2` switched on (green icon). Not present on other cars. |
| **`system.autoUpshift.reason`** | Why it is on: `wheel` or `limiter`; empty while standing by |
| **`system.autoUpshift.requests`** | Upshifts requested since the app started |

### Car settings (aliases) — **not yet available on any real car**

What the driver sets on the wheel, for the `carSettingsBar` widget. These names are reserved for when a car's decode provides them. Today only the sim rig (SimHub) fills them; on a real car they stay empty and the bar stays hidden.

| alias | meaning |
|---|---|
| **`tcLevel`** / **`tcActive`** | Traction control setting; `1` while it is intervening |
| **`absLevel`** / **`absActive`** | ABS setting; `1` while it is intervening |
| **`engineMap`** | Engine map / drive mode number |
| **`brakeBiasPct`** | Brake bias, % to the front axle — on real cars measured from brake pressures, see below |

### Brake pressure, brake % and brake bias

| alias | meaning |
|---|---|
| **`brakePressureFront`** / **`brakePressureRear`** | Line pressure per brake circuit, bar. From two pressure sensors on the analog-to-CAN box |
| **`brakePressure`** | One brake pressure for the pedal trace: the car's single sensor, else the front circuit |
| **`brakePct`** | Brake effort, 0–100 %: front pressure as a share of 120 bar |
| **`brakeBiasPct`** | Brake bias, % front: front ÷ (front + rear) |

**Brake bias** is only worked out while the front circuit is at **10 bar or more** — at lighter pressure the ratio is meaningless. Between stops it **keeps showing the bias of the last real braking**, so you can read it on the straight. It shows on the overview page's bottom strip only on cars that have it.

**Analog-to-CAN box (Metis Engineering, 8 × 0–5 V).** For a car without brake pressure on its data bus, fit two brake pressure sensors (Bosch 0 261 545 053: 0–140 bar, 0.5–4.5 V, 3-pin — sensor pin 1 ground, pin 2 signal, pin 3 5 V):

| Box pin | Connect |
|---|---|
| 1 / 2 | 12 V / ground |
| 3 / 4 | CAN low / CAN high — the 1 Mbit/s bus. The box has no termination resistor |
| 5 / 6 | 5 V and ground for **both** sensors |
| **12 (A5)** | **Rear** sensor signal |
| **13 (A6)** | **Front** sensor signal |

The box ships at 500 kbit/s — set it to **1000 kbit/s** before connecting it. A different sensor range only changes the conversion numbers in the two brake pressure aliases. An unplugged sensor reads about −17.5 bar; brake % and bias ignore it.

### Moving or stopped (`system.motion.*`)

| Channel | Meaning |
| --- | --- |
| **`system.motion.stopped`** | **`1`** while the car stands, **`0`** while it moves. Absent until the first GPS speed reading |
| **`system.motion.state`** | The same as text: `stopped` or `moving` |

"Stopped" means GPS speed at or under 3 km/h for two seconds; "moving" means 5 km/h on any reading.
The gap keeps a parked car's GPS jitter from flickering the dashboard's STOPPED sign, and a GPS
dropout changes nothing. This is what the lap-delta STOPPED sign and the lap table's blank running
lap follow.

### GPS — u-blox on UART (**Linux hub only**)

With a supported **u-blox** GNSS module on the configured serial device:

| Channel | Meaning |
| --- | --- |
| **`system.gps.latitudeDeg`** / **`system.gps.longitudeDeg`** | WGS84 degrees |
| **`system.gps.gpsSpeedKph`** | Speed from GNSS |
| **`system.gps.horizontalAccuracyM`** | Horizontal accuracy (metres), when available |
| **`system.gps.tsMs`** | GNSS timestamp (UTC ms) of the current position fix; not updated without a fix |
| **`system.gps.lastObtainedMs`** | Hub wall-clock ms when a fix/sample was last applied |
| **`system.gps.satellites`** | Satellite count |
| **`system.gps.fixType`** | Fix quality indicator |
| **`system.gps.indoor`** | `1` when the receiver's antenna is healthy but has heard almost nothing for a minute — the device is indoors or under cover; the dashboard then asks to be taken outside. `0` otherwise |
| **`system.gps.isConnected`** | `1` while the receiver link is active, else `0` |
| **`system.gps.sourceMode`** | Short diagnostic label for how samples are fed |

On **non-Linux** builds this path is inactive unless something else supplies those aliases.

### GPS — browser (**WebGPS**)

When the dashboard uses **browser geolocation**, values appear under **`system.WebGPS.*`** (latitude, longitude, speed, accuracy, timestamps). Alias order for GPS usually prefers **`system.gps.*`** first — see **`cfg/channelAliases.yml`**.

### Clock sync (`system.clock.*`)

The hub has no battery-backed clock, so on a cold boot without internet the system time can be
wrong until the first GPS fix. The hub verifies its clock against GPS time (or NTP when internet
is available), corrects the system clock when it is off, and renames any files that were named
with the wrong boot time so filenames always carry real timestamps.

| Channel | Meaning |
| --- | --- |
| **`system.clock.status`** | `unsynced` until verified; then `ntp`, `gps` (clock was already right), `gps-set` (clock corrected from GPS), `gps-offset` (GPS time used without changing the OS clock), or `host` (non-Linux dev host) |
| **`system.clock.offsetMs`** | Last measured GPS-vs-system-clock offset (ms) |

### Driver selection

| Channel | Meaning |
| --- | --- |
| **`system.currentDriver`** | Selected dashboard driver name (string), or empty |

### BerryGPS-IMU (**Linux**, onboard I2C + UART GPS)

When the BerryGPS-IMU hat is present and calibrated:

| Channel | Meaning |
| --- | --- |
| **`system.imu.innerTempC`** | IMU chip die temperature (°C); not cabin or ambient air |
| **`system.imu.accelLat`** / **`accelLong`** / **`accelVert`** | Vehicle-frame acceleration (g) |
| **`system.imu.gyroLat`** / **`gyroLong`** / **`gyroVert`** | Vehicle-frame angular rate (°/s) |
| **`system.imu.isCalibrated`** | `1` after boot zero calibration, else `0` |
| **`system.hat.magHeadingDeg`** | Compass heading (0–360°, vehicle frame; flat, no tilt compensation) |
| **`system.hat.baroAltitudeM`** | Barometric altitude (m) vs standard sea-level pressure |
| **`system.hat.baroPressureHPa`** | Barometric pressure (hPa) |
| **`system.hat.baroTempC`** | Barometer die temperature (°C) |

Aliases in **`channelAliases.yml`**: **`imuInnerTempC`**, **`compassHeadingDeg`**, **`baroAltitudeM`**, **`baroPressureHPa`**, **`baroTempC`**.

### GPS altitude (**Linux**, BerryGPS u-blox UART)

When a 3-D fix is available:

| Channel | Meaning |
| --- | --- |
| **`system.gps.altitudeMSLM`** | Height above mean sea level from GNSS (m) |

Alias **`gpsAltitudeM`** also accepts RaceBox MSL altitude when present.

### GPS quality (**Linux**)

Extra receiver detail, written to the session log so a weak fix can be explained later.

| Channel | Meaning |
| --- | --- |
| **`system.gps.satsTracked`** / **`satsWithEph`** | Satellites heard / that have orbit data |
| **`system.gps.cnoUsedMean`** / **`cnoTop4`** | Average signal of the sats used in the fix, and of the four strongest sats heard (dBHz). `cnoTop4` has a value before a fix too, but with fewer than four sats heard it averages only those, so check `satsTracked` |
| **`system.gps.satsUsed_*`** / **`cnoMax_*`** | Per constellation (e.g. `_gps`, `_gal`, `_bds`); unused ones stay `0` |
| **`system.gps.skyView`** | One-line snapshot of where the satellites sit in the sky |
| **`system.gps.agcPercent`**, **`magI`**, **`magQ`**, **`noisePerMS`**, **`jamInd`** | Front-end health (overload, imbalance, noise, jamming) |
| **`system.gps.antStatus`** / **`antPower`** | Antenna present/power; may say `(indoor)` when that is the reason |
| **`system.gps.measuredHz`** / **`configuredHz`** | Actual vs requested fix rate |
| **`system.gps.navPvtCount`** / **`ubxInvalidCount`** | Since the app started: positions the GPS sent, and damaged GPS messages received. A rising damaged count means the GPS connection itself is corrupting data; it stays at 0 on a healthy device |
| **`system.gps.ttffMs`** | Time to first fix (ms) |
| **`system.gps.dgpsOrBetter`** / **`isRtkEnabled`** / **`carrSoln`** | Correction in use / RTK armed / carrier solution |
| **`system.gps.ntripConnected`** / **`ntripBytesReceived`** / **`ntripMountpoint`** | Correction stream: up/down, bytes in, station name |

### Raspberry Pi hardware (**Linux**)

| Channel | Meaning |
| --- | --- |
| **`system.hwPerf.tempC`** | SoC temperature (°C) when readable |
| **`system.hwPerf.throttledRaw`** | Throttle/voltage flags |
| **`system.hwPerf.throttledStatus`** | Human-readable summary of throttle/voltage flags (e.g. `softTempLimit`, `freqCapped@boot`) |
| **`system.hwPerf.underVoltageNow`** | Under-voltage indication |
| **`system.hwPerf.loadAvg1`** / **`loadAvg5`** / **`loadAvg15`** | System load average over the last 1 / 5 / 15 minutes |
| **`system.hwPerf.cpuPct`** | Overall CPU busy percent (all cores, last second) |
| **`system.hwPerf.cpu0Pct`**, **`cpu1Pct`**, … | Busy percent of each core over the last second |
| **`system.hwPerf.boardPowerW`** | Power the board is using (W) — the heat, not charger/wall power |
| **`system.hwPerf.ext5vV`** | The device's internal 5 V supply (V). Shows where the power is coming from: about **5.1 V** on external power; about **4.8–4.9 V** on the internal battery; about **4.5 V** when external power is too weak and there is no usable battery (expect screen flicker and restarts) |
| **`system.hwPerf.alwaysOnV`** | The device's always-on supply, which keeps the clock and the GPS memory alive while it is off (V). About **3.3 V** on devices that can wake on motion, about **3.0 V** on earlier ones |
| **`system.hwPerf.coreVoltsV`** | SoC DVFS core voltage (V) |
| **`system.hwPerf.fanDutyPct`** | Cooling-fan speed (0–100%) |
| **`system.batteryStatus`** | UPS/battery charge (0–100%); convenient flat status channel |
| **`system.battery.percent`** | UPS/battery charge (0–100%) when exposed through Linux `power_supply` |
| **`system.battery.status`** | Charging state of the battery (`Charging`, `Discharging`, `Full`, `Idle`). On external power it reads `Discharging` only when the battery is really running down: the 12 V feed is not carrying the device, or on USB-C alone the battery voltage has kept falling for several minutes |
| **`system.battery.voltageV`** | Battery voltage (V), when reported by the driver. With the battery removed it can still read a drifting voltage (2.5–3.4 V seen) at 0%, not zero |
| **`system.battery.currentA`** | Battery current (A), when reported by the driver |
| **`system.battery.powerW`** | Battery power (W), when reported by the driver |
| **`system.battery.externalPower`** | **`1`** while the device is plugged in — the car's 12 V feed or USB-C — and **`0`** while it runs on its own battery |
| **`system.power.mainInput`** | **`1`** while the car's 12 V feed is connected (on boards before v2.3 also a USB-C charger), **`0`** while it is not |
| **`system.power.mainSupply`** | Whether the car's 12 V feed is actually running the device: `carrying`; `notCarrying` when it has been connected for 30 s but the device still runs on its own battery (the feed cannot carry it — the battery icon shows "plugged in but draining"); `pending` for the first seconds after plugging in; `unmeasured` when the feed is not connected or the internal 5 V supply is not being read |

The Pi itself has no battery gauge. These channels appear automatically when an attached UPS/HAT
registers a battery under `/sys/class/power_supply`; no application configuration is required.

### Screen brightness (**Linux**, built-in panel)

| Channel | Meaning |
| --- | --- |
| **`system.display.brightnessPct`** | What the panel is showing right now (0–100%), including mid-fade |
| **`system.display.brightnessTargetPct`** | Where it is heading — the level the current conditions call for |
| **`system.display.brightnessMode`** | Why: `active (moving)`, `active (screen in use)`, `idle`, or `manual` |
| **`system.display.brightnessControl`** | `auto` or `manual` — what the settings page has chosen |
| **`system.display.manualPct`** | The remembered manual level, even while in `auto` |

Present only when the device has a built-in panel whose backlight it can drive. See
[configuration.md](./configuration.md#screen-brightness-display-in-yaml) for what sets the level.

### Network and modem (**Linux**)

These update even when Wi-Fi is carrying internet, so a session still has cellular reception.

| Channel | Meaning |
| --- | --- |
| **`system.net.kind`** | Link in use: `wifi`, `modem`, or `none` |
| **`system.net.signalPct`** | Bars of that link (0–100). **`-1`** = unknown |
| **`system.wifiSSIDName`** | Associated Wi-Fi name; empty when not on Wi-Fi |
| **`system.wifi.state`** | One JSON object for the settings page: radio on/off, the connected network, saved networks (name, priority, built-in or not), the networks in range (name, signal, security, whether saved), what it is busy with, and the last error |
| **`system.net.wired`** | One JSON object for the settings page's Ethernet card: whether a cable is in, its speed, the address the other end handed out, whether the internet (and cloud sync) goes over it, and the device's `<name>.local` |
| **`system.wifi.power`** | One JSON object for the settings page's "Disable wireless" switch: whether wireless is set to disabled, whether the Wi-Fi and Bluetooth hardware is actually powered right now (`on` / `off` / `partial` / `unknown`, plus the two enable-line levels when readable), whether a switch is in progress, and the last error |
| **`system.net.modem.state`** | Modem state (`connected`, `searching`, `failed`, …) |
| **`system.net.modem.tech`** / **`bands`** | Radio in use (`lte`, `5gnr`, …) and serving band |
| **`system.net.modem.rsrpDbm`** | Cell strength (dBm): better toward **−44**, unusable near **−140** |
| **`system.net.modem.rsrqDb`** / **`snrDb`** | How clean the cell is / signal-to-noise (dB) |
| **`system.net.modem.rssiDbm`** | Total power in the band (dBm). High RSSI with low RSRP usually means interference, not just a weak cell |

Unknown radio numbers are **−999**, so an old reading cannot linger after the modem drops.

### Cloud sync

| Channel | Meaning |
| --- | --- |
| **`system.syncStatus`** | `upToDate`, `uploading`, `syncingLogs`, `waitingUntilStopped` (video waits until the car is stopped), `waitingForConnection` (video held until the allowed link — cable, Wi-Fi, or any), `offline` (no network at all), `unreachable` (a network, but the cloud does not answer), or `disabled` (no cloud access configured) |
| **`system.videoSyncProgress`** | Upload percent 0–100. **`-1`** = nothing queued; **`-2`** = a log/session file is uploading |
| **`system.videosInUploadQueue`** | Videos waiting to upload |
| **`system.syncLink`** | Kind of link uploads would leave by: `wifi`, `wired`, `cellular`, or `none` |
| **`system.syncVideoPolicy`** | Whether video may use mobile data: `always` (it may) or `wifi` (only Wi-Fi or a cable) |
| **`system.syncVideoWhileDriving`** | `1` = live upload on: onboard video uploads while the car is moving too; `0` = video waits until the car is stopped. Absent when this device has no cloud sync |

### Data logging

| Channel | Meaning |
| --- | --- |
| **`system.canDump.upload`** | The settings switch "Upload raw CAN data": `on` (the default) or `off` |

### Vehicle diagnostics

| Channel | Meaning |
| --- | --- |
| **`system.dtc.autoClear`** | **`1`** while the settings switch "Automatically clear fault codes" is on, **`0`** while it's off (the default) |
| **`system.gnssSelfTest`** | One JSON object, refreshed every second while the settings page's GNSS self check runs: which step it is on and how long that can still take, and each test's numbers so far. Empty once the run ends (the result moves to the channel below); a check that could not start at all stays here, with the reason |
| **`system.gnssSelfTestResult`** | The last finished GNSS self check, as one JSON object: when it ran, how it ended (finished, stopped early and why, or could not run), its comment (editable on the settings page; it starts as the device and "car" or "bench"), whether it ran in a car and on external power, the first test's numbers (how old the live AssistNow data was, time to first fix, jitter, best accuracy reached, satellites, signal strength, SBAS), the 25 Hz test (positions a second, pass or fail, jitter and best accuracy against 10 positions a second just before and just after it), the recording test (best accuracy and best signal against the 10-a-second measurement just before it, or why it was left out), the score out of 100, and the last test's pass or fail, plus when each part started and the run ended — both in device time and in the session log's own clock, so the detailed readings of that window can be found in the session log. Kept across restarts; the session log carries it, so it reaches the cloud with it |
| **`system.peripherals.probe`** | One JSON object for the settings page's Peripherals status window, refreshed while it is open: whether a microphone is fitted, per CAN port whether a car is on it and how long it has been quiet, and whether the Bluetooth OBD dongle is connected, when it last heard data, and its error |
| **`system.dtc.clearNow`** | One JSON object for the last **Clear now** on the settings page: whether it is running, done or could not start (and why), and per control unit (ECU, TCU) whether it was cleared, did not answer or refused |

### Power

| Channel | Meaning |
| --- | --- |
| **`system.power.state`** | `shuttingDown` once "hold to power off" on the settings page has been accepted |
| **`system.power.autoShutdownInS`** | Seconds until the device powers itself off (parked, the internal battery draining), while that countdown is on screen; **`-1`** otherwise |
| **`system.power.autoWakeNearTrack`** | **`1`** while the settings switch "Auto wake up near a track" is on, **`0`** while it's off (the default) |
| **`system.power.wakeMode`** | How this boot started: `normal`; `rtcChecking` (woke by itself, screen dark, looking for a track); `rtcStayedUp` (a track is near — running as usual); `rtcPoweringOff` (no track near — switching off again) |
| **`system.power.weakCharger`** | Boards before v2.3: **`1`** while the external power — a USB-C charger or the car's 12 V feed — is too weak to run the device and charge it (the battery drains with the cable in), **`0`** otherwise |
| **`system.power.chargerHeldOff`** | Boards before v2.3: **`1`** while the device has paused its own battery charging so it can run on a weak USB-C charger alone (takes effect on v2.2 boards) |
| **`system.power.pfoEdgesPerMin`** | Boards before v2.3: how many times a minute the input dipped below the power-fail level — a supply that cannot carry the device (weak USB-C charger or weak 12 V feed) shows ~120–128, a healthy supply 0 |
| **`system.usbCharger.state`** | USB-C charger (boards from v2.3): `configured`, `configuring` (settings being written), `absent` (no USB-C charger on this board), `error` |
| **`system.usbCharger.input`** | What is plugged into USB-C: `none`, `usbHost` (a computer port), `adapter`, `unknown` |
| **`system.usbCharger.inputPresent`** | **`1`** while a USB-C charger or computer port is powering the device, **`0`** while nothing usable is plugged into USB-C |
| **`system.usbCharger.charge`** | `notCharging`, `preCharge` (deeply empty battery), `fastCharging`, `done` |
| **`system.usbCharger.fault`** | Comma-separated active faults (`input`, `thermal`, `safetyTimer`, `batteryOvp`, `ntc:cold`, …); empty when none |
| **`system.usbCharger.chargeCurrentMa`** | Charge current USB-C is set to (mA) — lower while 12 V is also connected |
| **`system.usbCharger.inputLimited`** | `1` when the USB-C source cannot supply what the charger asks and it is holding back |

### Video recording

Onboard camera recorder: **`system.videoRecordingStatus`** (`off` when no camera is available), **`system.videoRecordingFileSizeBytes`**, **`system.videoRecordingStartedAtMs`**, **`system.videoRecordingFileName`**. **`system.videoQuality`** is the quality the next recording uses (`720p`, `1080p`, `720p50`, `1080p50`) — the one picked in Settings, kept after a restart. After an automatic step down it can also read `720pLite`: a 720p mode with somewhat lower picture quality that is easier on the device, used only as the last step before a recording is stopped. It cannot be picked in Settings. **`system.videoStorageFull`** is `1` while there is not enough disk space left to record (the recording has stopped), else `0`; **`system.videoDiskFreeBytes`** is the free space at the last check. **`system.videoRestartRequired`** is a short reason the camera needs a device reboot (empty when it does not). **`system.videoCameraSensor`** names the attached built-in camera's sensor (empty on a USB camera or when no camera is recording) — the dashboard uses it to decide which recording qualities to offer.

During a track session the onboard recording starts once the car drives off (holds above about 10 km/h for a few seconds), not while it is still parked in the paddock. After that it records as usual until the session ends. If the device restarts mid-session, recording picks back up right away.

A phone uploading video publishes **`system.webVideoRecordingFileSizeBytes`** (bytes across open uploads; `0` when none). **`system.webVideoClient*`** (`PaintFps`, `PaintMaxMs`, `SnapshotHz`, `SnapshotMaxMs`, `BacklogBytes`, `BacklogChunks`) show how hard that phone is working; **`-1`** when idle.

Recording performance (updated every second while a recording is live):

| Channel | Meaning |
| --- | --- |
| **`system.videoRecordingFps`** | Measured framerate of the newest second of onboard video actually saved to disk. `0` means the recording has stalled (nothing new saved for ~10 s); `-1` means no measurement yet (just started, or not recording) |
| **`system.webVideoRecordingFps`** | Same measurement for the phone upload (target 25); with several phones uploading at once it follows the newest one; `0` stalled, `-1` no measurement |

Both are written to session logs. **`system.videoRecordingsJson`** additionally carries the per-recording detail (file, fps, target, size — one entry per recording, including each simultaneous phone upload) that feeds the live recordings table and the camera modal; it is not written to session logs.

The dashboard VIDEO button shows **DEGRADED** while any live recording measures below 90% of its target, with per-recording detail in the camera modal. An onboard recording that stays degraded, or runs too hot, automatically restarts one step lower: a 50 fps recording first drops to 30 fps at the same size, then 1080p drops to 720p, and 720p drops to `720pLite`. Each step starts a new video file, and the recording does not go back up on its own. Only if `720pLite` also overheats is the recording stopped. For phone recordings, see the tips in [phone-recorder.md](./phone-recorder.md).

While any recording is live, **`system.videoWeather`** holds the weather seen by the camera as a short label (`sunny`, `overcast`, `damp`, `lightRain`, `mediumRain`, `heavyRain`, `snow`, or `indoor` when the camera can't see outside). The first value appears about a minute after recording starts and refreshes every few minutes. The last value sticks around after recording stops.

---

## Sim rig

GridCore also runs as a sim dashboard, fed by SimHub. Sim data lands under **`simhubTelemetry.*`** and is the **last** fallback in the standard aliases, so real-car sources always win.

---

## Related

- **`channelAliases`** — canonical alias keys live in **`cfg/channelAliases.yml`**; lap logging and widgets depend on stable names such as **`gpsLat`** / **`gpsLon`** / **`gpsSpeedKph`** / **`gpsHorizontalAccuracyM`** staying paired.
- **`cfg/main.yml`** — authoritative **`ui.channelSources`** documentation in the file header.
