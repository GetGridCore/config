# GridCore config

GridCore comes pre-configured, but by tweaking a handful of `.yml` files you can make it your own: custom widgets, custom dashboard pages, custom data sources. This repo is the guide on how to do it, with the configs our own devices run as examples.

You will be able to upload a config to your [GridCore One](https://www.getgridcore.com/) through a CLI and the web UI later on.

## Start here

| You want to | Read | Example |
| --- | --- | --- |
| See how a config is put together | [docs/configuration.md](docs/configuration.md) | [cfg/main.yml](cfg/main.yml) (the comment block at the top describes every key) |
| Build or change a dashboard page | [docs/widgets.md](docs/widgets.md) | [cfg/overview.yml](cfg/overview.yml), [cfg/instrumentCluster.yml](cfg/instrumentCluster.yml), [cfg/widgetDemo.yml](cfg/widgetDemo.yml) (every widget) |
| Read a car over OBD2 | [docs/obd2-channel-source.md](docs/obd2-channel-source.md) | [cfg/genericDiag.yml](cfg/genericDiag.yml) (Bluetooth dongle), [cfg/genericDiagCAN.yml](cfg/genericDiagCAN.yml) (wired) |
| Get advanced channels with only OBD2 access (UDS) | [docs/obd2-channel-source.md](docs/obd2-channel-source.md#customudschannels-rows) | [cfg/mclarenDiag.yml](cfg/mclarenDiag.yml): engine and gearbox values the standard OBD2 list does not have |
| Get data from a BMW 318ti Compact cup car over the legacy K-line | | [cfg/ds2M52.yml](cfg/ds2M52.yml). Note: this one shows that GridCore can get data out of almost any car |
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

## AI and agents

**Pro tip:** throw this repo to your AI agent to make a custom config for you. You can truly get the best out of GridCore that way.

## Proposing a change

Open an issue or a pull request. A car we do not read yet is the most useful thing to send: the car, the year, and a log or a DBC if you have one.
