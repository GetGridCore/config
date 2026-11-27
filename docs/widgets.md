# Available widgets

Widgets are placed under `ui.pages` in your config — see [configuration.md](./configuration.md#dashboard-pages-and-widgets) for the entry shape (`id`, `widget`, `gridClass`, `params`).

| Widget | Aliases | Description |
| --- | --- | --- |
| `textData` | — | Numeric or text card. Params: `channel`, `format` (`lapTime`, `signedDelta`, …), `decimals`, `unit`. |
| `textDataConditional` | — | Like `textData` with conditional styling. |
| `textDataLapDelta` | — | Lap delta in seconds with green/red color. |
| `textDataGearMismatch` | — | Like `textData` but highlights when requested and engaged gear disagree. |
| `textDataGpsFix` | — | GPS fix quality indicator with stale detection. |
| `barGauge` | — | Vertical fill bar between `min` and `max`. |
| `rpmTach` | — | RPM arc display. |
| `rpmAndMaps` | — | Shift-light strip plus drive-mode pills (MAP/TC/ABS levels; a pill flashes while its aid intervenes). Shows only while sim data is streaming — hides itself otherwise. |
| `carSettingsBar` | — | The car's driver-adjustable settings as pills: **TC**, **ABS**, **MAP** and **BB** (brake bias, % front); TC and ABS flash while the aid intervenes. A pill shows only when the car reports that setting, and the bar hides itself when none is reported. **Not yet available on any real car** — only the sim rig reports these today. Params: `pills` (list of `label`, `levelChannel`, optional `activeChannel`, `unit`, `decimals`) to change or reorder them. |
| `pedalBars` | — | Side-by-side throttle and brake bars. |
| `inputsChart` | — | RaceLab-style driver inputs: rolling chart (throttle, optional brake and steering), live pedal bars, optional steering wheel. |
| `gMeterWithInputs` | `gMeter` | Lateral/longitudinal G circle with optional trails and envelopes. |
| `miniMetricsGrid` | — | 2-column grid of numeric tiles; configure via `params.items`. |
| `fuel` | — | (also a `textDataStripGrid` column `variant`) Tank level with consumption per lap, laps left (measured over the last laps) and laps left at a fixed consumption, with two laps held in reserve. Params: `channel` (default `fuelLevel`), `unit` (default `%`), `decimals`, `warnUnder`, `lapsWarnUnder` (default 1). The per-lap numbers need two completed laps and appear only on a closed circuit (not on a point-to-point course or with no track); **calc laps left** appears only when `fuel.calculatedLapConsumption` is set. The rates and reserve are set in the top-level `fuel:` block. |
| `ttpms` | — | IZZE TTPMS corner widget (`corner`: LF/RF/LR/RR). Shows temp channels, pressure, and battery. |
| `simpleTire` | — | Single-tire temp, pressure, and optional battery strip. |
| `tpmsTempLegend` | — | Static color legend for tire temperature bands. |
| `weatherForecast` | — | Weather forecast using live GPS position. |
| `satellitePreview` | — | Live GPS position on a satellite map. Optional `requiresTapToActivate` shows a tap prompt before loading imagery. |
| `debugChannelsTable` | — | Developer view: all channel sources, decode errors, last update times. |

**Layout demo:** `cfg/widgetDemo.yml` has every widget on one set of pages, with the params each one takes.
