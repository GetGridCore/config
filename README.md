# GridCore config

The configuration files a [GridCore](https://www.getgridcore.com) device runs on, with the docs that explain them.

A GridCore is set up by a handful of YAML files: which data to read from the car, what each dashboard page shows, what goes into the video overlay and the logs. Everyday options (video quality, brightness, tire sensors, the overview page) are on the device's Settings page and need none of this. These files are for everything past that: your own dashboard pages, a car we have not seen yet, a sensor on the CAN bus.

Editing them on your own device is not open to owners yet. Until it is, this repo is the reference for what a device can do, and the place to propose a change.

## Start here

| You want to | Read | Example |
| --- | --- | --- |
| See how a config is put together | [docs/configuration.md](docs/configuration.md) | [cfg/main.yml](cfg/main.yml) (the comment block at the top describes every key) |
| Build or change a dashboard page | [docs/widgets.md](docs/widgets.md) | [cfg/overview.yml](cfg/overview.yml), [cfg/instrumentCluster.yml](cfg/instrumentCluster.yml), [cfg/widgetDemo.yml](cfg/widgetDemo.yml) (every widget) |
| Read a car over OBD2 | [docs/obd2-channel-source.md](docs/obd2-channel-source.md) | [cfg/genericDiag.yml](cfg/genericDiag.yml) (Bluetooth dongle), [cfg/genericDiagCAN.yml](cfg/genericDiagCAN.yml) (wired) |
| Read a car's own CAN bus | [docs/configuration.md](docs/configuration.md#data-sources-uichannelsources) | [cfg/dbc/porsche_718.dbc](cfg/dbc/porsche_718.dbc), [cfg/dbc/reversed/bmwGSeries.dbc](cfg/dbc/reversed/bmwGSeries.dbc) |
| Name a reading once and use it everywhere | [docs/live-channels.md](docs/live-channels.md) | [cfg/channelAliases.yml](cfg/channelAliases.yml), [cfg/mathChannels.yml](cfg/mathChannels.yml) |
| Add tire sensors | [docs/configuration.md](docs/configuration.md#data-sources-uichannelsources) | [cfg/simpleTpms.yml](cfg/simpleTpms.yml), [cfg/izzeTTPMSMichelin.yml](cfg/izzeTTPMSMichelin.yml) |
| Set warnings | | [cfg/warnings.yml](cfg/warnings.yml) |

## How the files fit

`cfg/main.yml` is the one the device loads. It pulls the others in with `!include`, so each page and each car lives in its own file.

A reading gets from the car to the screen in three steps:

1. A **source** in `ui.channelSources` reads it: OBD2, a CAN bus with a DBC file, or Bluetooth.
2. An **alias** in `channelAliases.yml` gives it a short name such as `rpm` or `coolantTemp`, the same on every car.
3. A **widget** on a page shows the alias, and the logs record it.

## Car files

Two DBC files are here as examples: the Porsche 718 Cayman / Boxster (982) and BMW F/G series. Both come from community sources, and the notes at the top of each file say what is confirmed and what is not. The configs also name files for other cars and sensors that are not published yet.

## Proposing a change

Open an issue or a pull request. A car we do not read yet is the most useful thing to send: the car, the year, and a log or a DBC if you have one.
