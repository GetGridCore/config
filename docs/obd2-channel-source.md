# OBD-II channel sources (`ui.channelSources`)

OBD sources have **no** `file` (no DBC). `bus` picks the CAN interface (same as CAN+DBC).

**What you get:** live OBD PIDs (Mode 01), optional manufacturer DIDs via UDS (`customUDSChannels`), and stored fault codes (`readDTCs`).

**FYI:** some cars switch off the power on their OBD2 port after being parked for a few hours, or when the car battery gets low. GridCore then runs on its own battery and switches itself off after 10 minutes. This is normal.

## Arguments (documented further down)

- `fromObd2`: all data channels to read
- `readDTCs` (and `.readDTCFromUDS`, `.moduleIds`)
- `customUDSChannels`
- `onlyIfBusNotSilent`/`onlyIfBusNotSilentWithinMs`
- `onlyIfVehicle`/`onlyIfNotVehicle`/`onlyIfVehicleKnown`
- `waitBetweenPolls`
- `extendedSession`
- `heavyOperationsDebounce`
- `obdAddressing`: `11` (default), `29`, or `auto` — some vehicles answer on longer CAN addresses; `auto` tries the usual short address first, then the long one, and keeps whichever the vehicle replies on (wired CAN only)

## Channel naming

- **OBD2:** `<key>.<signal>` — one value per name in `fromObd2`.
- **UDS:** `<key>.uds.<name>` — one value per `customUDSChannels` row.
- **DTCs:** `totalNumberOfDTCs`, `DTCsAtLastRead`, `uniqueDTCsSeen`, `lastSuccessfulDTCReadTs`; after a successful clear, `lastDTCClear` / `lastDTCClearModule`.

## Bluetooth ELM327 adapters (vLinker MC+ etc.)

Set `bus: can-bluetooth` instead of a wired bus name to use a Bluetooth Low Energy ELM327/STN dongle (Vgate vLinker MC+ is the reference device). All other fields — `fromObd2`, `customUDSChannels`, `readDTCs`, `extendedSession`, `onlyIfBusNotSilent`, … — work exactly the same as on a wired CAN source.

What you need to know:

- **No pairing or MAC required.** The dongle is auto-discovered: the first BLE peripheral whose advertised name contains `vlinker`, `veepeak`, or `obdii` is used. Power on **one** Bluetooth OBD dongle at a time; extra ones get a one-time warning and are ignored.
- **Pinning a specific dongle (optional).** Add `preferredId: <id>` (or a list) to the source to lock onto one particular dongle when more than one might be in range — for example a daily-driver vLinker in the garage while a track-car vLinker is connected. Any other matching dongle is still accepted as a fallback if the preferred one is missing (~8 s grace at startup, then any matching dongle wins). The id is the platform-specific token printed on first connect — the `[ELM327-BLE] locking on dongle id=<...>` log line tells you what to paste:
   - **macOS**: an opaque CoreBluetooth UUID (32 hex chars). Apple hides the real BLE MAC, and the UUID is per-host, so the same dongle gets a different value on a different Mac.
   - **Linux / Raspberry Pi**: the dongle's actual BLE MAC (12 hex chars, often printed on the device label).
   - The two are **not interchangeable** — a macOS UUID will never match on Linux, and vice versa. If the same `cfg/*.yml` runs on both, set `preferredId` to a **list** containing both ids; whichever id matches on the current host wins. Dashes / colons in the value are accepted but optional (it's normalised to lowercase alphanumeric).
- **Bluetooth must be enabled** on the Pi / Mac running the app. On Raspberry Pi this is the same `setcap cap_net_raw+eip $(node)` and `rfkill unblock bluetooth` setup already used for the BLE TPMS / RaceBox sources.
- **Coexisting with BLE TPMS / RaceBox on Linux: set both `NOBLE_MULTI_ROLE=1` *and* `NOBLE_REPORT_ALL_HCI_EVENTS=1`.** On a Raspberry Pi (and most Linux boxes with a single built-in Bluetooth radio), TPMS / RaceBox values stop updating as soon as the vLinker is connected unless these env vars are set before launching the app:
   - `NOBLE_MULTI_ROLE=1` lets the radio keep scanning while the vLinker GATT connection is up. Without it, scanning is silently suspended for the duration of the connection.
   - `NOBLE_REPORT_ALL_HCI_EVENTS=1` makes every individual advertisement surface to the app immediately, instead of being held back until a second sighting confirms it. This matters because TPMS sensors broadcast sparsely (every few seconds) and don't reply to scan requests; with sparse scan time (vLinker connected + RF noise in motion) the "wait for a second sighting" default can drop a broadcast cycle entirely and TPMS values stop updating mid-drive.
   - Both vars have no effect on macOS (CoreBluetooth handles this automatically); they're harmless to set anyway.
   - If env vars don't help your specific hardware, the fallback is a second USB Bluetooth dongle dedicated to the vLinker.
- **OBD-II port only.** The dongle exposes the diagnostic CAN bus the car routes to OBD-II — typically the powertrain ECU (engine, sometimes transmission). It does **not** see manufacturer back-bones such as PT-CAN, K-CAN, etc.
- **Diagnostics-only by design — no live CAN sniffing.** Between OBD requests the dongle sits quietly at the ELM prompt; broadcast frames on the OBD bus are *not* surfaced to the app. This is intentional: on busy modern car buses (e.g. BMW G-chassis with chassis-broadcast frames at hundreds of Hz) trying to monitor and poll at the same time saturates the BLE pipe, drowns the OBD response window, and makes every poll miss. If you need to sniff the OBD bus, use a wired CAN adapter.
- **Slower than wired CAN.** Each diagnostic round-trip travels over BLE and the ELM327 firmware adds its own protocol overhead; expect a few hundred milliseconds per request rather than tens. The runtime automatically uses an 800 ms per-PID response budget for `bus: can-bluetooth` (vs 200 ms wired). Set `responseTimeoutMs: <ms>` on the source to override per car. `onlyIfBusNotSilentWithinMs` should be ~5000 ms (instead of the wired-CAN default of 2000) to ride out brief lulls between successful replies.
- **Mode 01 PIDs are batched** to keep BLE round-trips down. SAE J1979 lets one request carry up to **6 PIDs** at a time; the ECU answers with all values packed into a single response. The runtime defaults to a batch size of **6** for `bus: can-bluetooth` (and **1** for wired CAN, where the round-trip cost is already negligible), so a 17-PID `fromObd2` list costs ~3 BLE round-trips per pass instead of ~17 — typically **~3× faster** end-to-end. Set `obdMode01BatchSize: 1` to disable batching if a quirky ECU rejects multi-PID requests; you'll see `7F 01 ...` (negative response) in the `[OBD2]` log if that happens. To get the polling cycle faster than the batched default, trim the `fromObd2` list itself (each PID still costs BLE bandwidth even when batched).
- **Same channel naming.** Channels still come out as `<key>.<signal>` and `<key>.uds.<name>` — `cfg/genericDiag.yml` is a complete worked example for Bluetooth OBD-II.

## `fromObd2`

Strings are Mode 01 PID names; each listed name updates `<key>.<signal>`. `cfg/genericDiag.yml` lists the ones in everyday use.

A value the car never answers is asked for less and less often (at most every 15 seconds), so it does not slow the others down; it comes back on its own as soon as the car starts answering. Intake air temperature and manifold pressure have a second standard form used by cars with several sensors: if the usual one gets no answer, GridCore switches to that form under the same name. A value the car does not offer at all stays empty — some cars (e.g. a 2015-era Mercedes S-Class coupé) report neither oil temperature nor fuel level over standard OBD-II.

## `readDTCs`

Optional block: `moduleIds` (ECU addresses to poll; default `[0x7e0]` when you use `readDTCs: true` or leave `moduleIds` empty), plus optional `mode03TimeoutMs` and `readDTCFromUDS` (bool). Populates the DTC channels above.

NOTE: `readDTCFromUDS` is much slower than regular read (see below)

## `customUDSChannels` rows

Each row needs `name`, `did` (UDS data identifier, hex), `dataBits` (8|16|24|32), `multiplier`, `addBefore`, `addAfter` (scaling: decoded raw value is `(raw + addBefore) * multiplier + addAfter`), and optionally:

- **`dataByteOffset`** — bytes to skip after `0x62` + DID before decoding `dataBits` (default 0; values are MSB-first / big-endian for multi-byte widths). Aliases: **`valueByteOffset`**, **`byteOffset`**. **`valueStartBit`** / **`dataStartBit`** / **`startBit`** — divided by 8 to get the same offset (if set, it overrides the byte-offset fields).
- **`toAddress`** — physical request arbitration id (tester → ECU). Aliases: **`txID`**, **`txId`** (like `obdTxId`). If omitted, the source’s `obdTxId` is used, then functional **`0x7df`**.
- **`fromAddress`** — response id for ISO-TP (ECU → tester). Aliases: **`rxID`**, **`rxId`** (like `obdRxId`). If omitted, the runtime uses the usual physical offset (**request + 8**, same as `physicalResponseId` in `obdIsoTp.ts`); for functional **`0x7df`**, the source’s **`obdRxId`** (default **`0x7e8`**).

If the car requires extended session to read UDS data (rare), you must use **`extendedSession`** and include the module IDs you're requesting data from in it explicity.

**Temporary debug (car / bench):** **`UDS_DEBUG_MULTIFRAME_RESPONSES`** — `[UDSDBG]` line per positive ReadDID (`pduLen`, full hex, decoded `raw`).

## Queue and timing

Each cycle runs the **fast queue** in order:

1. **Mode 01** — one step per `fromObd2` name. Default request/response IDs: the usual short pair unless you override `obdTxId` / `obdRxId`. With `obdAddressing: auto` on wired CAN, a miss on that pair is retried on the longer truck-style addresses and the working pair is kept.
2. **Custom UDS** — one step per `customUDSChannels` row (read-by-DID).

If **`readDTCs`** is set, the runtime then does a **Mode 03** read per module (updates DTC channels). After that, it may start a **heavy** pass on a separate timer (see below).

Between full queue cycles it waits **`waitBetweenPolls`** ms (default 200). `<key>.lastPollTimeMs` is how long the last cycle took; the wait can shrink a bit if the cycle was already slow.

### When answers mostly don’t come back (eg you are reading channels that the car does not respond to)

Every fast queue pass, if there's more fails than successes, we throttle: it **waits longer before the next batch** (you’ll see “cooling off” in the log), and the wait can grow if that keeps happening. **Ties** (same number of hits and misses) or **more hits than misses** are treated as “good enough” for pacing - it keeps the normal gap between runs.

You care because this is how the app avoids hammering a dead/sleeping or overloaded bus, and because **the heavy DTC work** (below) only runs when the fast queue has been mostly healthy lately — if you’re constantly in “mostly failing” territory, snapshot Mode 03 still runs, but the longer UDS/clear path waits until things look stable again.

**Practical warning:** a long `fromObd2` / `customUDSChannels` list where only a couple of items actually work will **still** count as “mostly failing” every time. You’ll get noisy logs and aggressive slowdowns even though two channels update fine. Shorten the list to what you need, fix TX/RX/DIDs, or split sources — don’t read it as “the backoff is wrong.”

## `readDTCs` and the heavy pass

**Snapshot:** OBD2 Mode 03 to read errors after each fast pass. **Extended diagnostic session** (`0x10 0x03`) is sent first to ECUs listed in **`extendedSession`** that also appear in **`readDTCs.moduleIds`** (see below). Renewal spacing: **`extendedSessionRenewMs`**.

**Heavy pass:** Runs at most every **`heavyOperationsDebounceMs`** (default 30 s), and only after the fast queue has had a **stretch of mostly-successful batches** (see above). If you have **no** Mode 01 / custom UDS steps — DTC reads only — that gate is skipped so heavy can still run. The pass does a longer DTC workflow: extended session, re-read stored codes, and if **`readDTCFromUDS`** is true an extra UDS DTC list is merged into the same DTC fields. While a heavy pass is active, the quick Mode 03 snapshot is skipped so we don’t stack two diagnostic conversations on the same IDs.

**Which DTCs are reported (UDS path):** when `readDTCFromUDS` is enabled, only faults that are **confirmed** or have **failed at least once since the last clear** are shown. ECUs keep a much larger historical record internally — requesting everything would surface dozens of old, inactive codes that are not meaningful in practice. This matches what professional scan tools show by default.

## Clearing fault codes

Clearing is not set per source — it is on the device's **Settings** page, under **Vehicle diagnostics**:

- **Automatically clear fault codes** (off by default): whenever a source that reads DTCs finds codes on the engine ECU (`0x7e0`) or the gearbox TCU (`0x7e1`), it clears them in its next heavy pass.
- **Clear now** clears the ECU and the TCU once, straight away, through every OBD source that is talking to the car at that moment, and shows per unit whether it worked.

Only the ECU and the TCU are ever cleared. Before each clear the unit gets an extended diagnostic session automatically — you do not need to list it in `extendedSession`.

## Advanced: `extendedSession`

**NOTE:** in theory, opening an extended session is required on some cars to read DTCs and UDS data. In practice it's probably rare, so do not use this unless you know what you're doing.

**NOTE:** if you add module IDs to `readDTCs.moduleIds`, they will not automatically get their session extended - you need to explicitly list them here in `extendedSession`.

Array of tester -> ECU **request** ids (`0x7e0`, `0x7e1`, …) on which we send **UDS extended diagnostic session** (`0x10 0x03`) before:

- **Custom UDS ReadDID** (`0x22`) steps in the fast queue (same list for every row — auxiliary ECUs such as ESC are **not** included unless you list them),
- **Mode 03** snapshot and **heavy** DTC work — only for ids that are both here **and** in **`readDTCs.moduleIds`**.

**Default when omitted:** **`[]`** (no extended session). **Examples:** **`[0x7e0]`** — ECM / engine ECU only; **`[0x7e0, 0x7e1]`** — ECM and a second ECU (e.g. TCU).

## Other options

- **CAN link (always on)** — OBD polling does not start until the configured CAN adapter is connected and ready to transmit (wired USB/socket or Bluetooth dongle). You’ll see `OBD paused reason=link` in the log until then; no YAML flag required.
- **`heavyOperationsDebounceMs`** (alias `heavyOperationsDebounce`) — minimum time between heavy passes (default 30000 ms).
- **`onlyIfBusNotSilent`** — don’t transmit until the live adapter has seen CAN traffic within `onlyIfBusNotSilentWithinMs` (default 2000 ms). Use this when you are on a live car backbone (PT-CAN) so a sleeping car is not woken by chatter. Do **not** set it on a normal OBD-II port: that port is often silent until we ask, so after a restart the gate never opens and diagnostics stay stuck. Wired generic OBD leaves this off; the McLaren source keeps it. If the bus goes quiet mid-pass, the rest of that cycle is aborted.
- **`onlyIfChannelNonZero`** — don’t transmit until a given numeric channel is non-zero (full path, e.g. from another source). Same mid-pass abort behaviour as above.
- **`onlyIfVehicle` / `onlyIfNotVehicle`** — only talk to the car when it is (or is not) one of the listed vehicle profiles. While the car is still being identified both of these **pass**: an unidentified car means "any profile". The gate is re-checked every pass, so identifying the car mid-session hands the bus over without a restart — you'll see `OBD paused reason=vehicle` on the one standing down and `OBD resumed` on the one taking over.
- **`onlyIfVehicleKnown`** — stay silent until the car has actually been identified, instead of the default above. Set this on a source that must not talk to a car it may not belong to.

  Together these let two diagnostic sources share one bus: put `onlyIfVehicle: mclaren` + `onlyIfVehicleKnown: true` on the car-specific one and `onlyIfNotVehicle: mclaren` on the generic fallback, and exactly one of them is ever allowed to transmit — including during the moment before the car is identified, which the fallback covers. Prefer that arrangement: only one diagnostic conversation per bus at a time is the safe rule, because two testers interleaving requests on the same addresses is a known way to make an ECU stop answering.

  If you do let two overlap, the damage is bounded rather than fatal. A reply is matched to a request by the PID echoed back in it, and a source only ever writes channels it actually asked for, so a reply reaching the wrong source costs that reading a pass (it backs off and retries) instead of producing a wrong number. This holds only while both sources ask for one value at a time (`obdMode01BatchSize: 1`, the default on a wired bus) — two testers trading multi-part replies would also trade the handshake frames that go with them, and that is what upsets a gateway.
- **`verboseFails`** — log every failed queue step; default is to throttle failure spam when the bus is down.

## Status bar / UI

`obdDtcSourceKey` ties the DTC column to a source; other widget params are in `cfg/main.yml`.
