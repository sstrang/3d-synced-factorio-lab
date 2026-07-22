# 3D-Synced Factorio Lab

A physical, 3D-printed Factorio science laboratory that syncs its lighting to
your game in real time — replicating the [Disco Science](https://mods.factorio.com/mod/DiscoScience)
mod's effect on real LED hardware.

When you start a research in-game, the lab's LEDs flicker and cycle through the
colours of the science packs being consumed. When research completes, the lab
flashes white five times. When research is cancelled, the light turns off.

<p align="center">
  <img src="https://makerworld.com/dimension/558821" alt="3D-printed Factorio Lab" width="400">
</p>

## How It Works

```
Factorio → Events Logger mod → game-events.json → MQTT Bridge → MQTT Broker → Home Assistant → WLED → LEDs
```

1. The **Events Logger** Factorio mod writes each in-game event (research
   started, finished, cancelled, etc.) as a line of JSON to a file called
   `game-events.json`.

2. The **MQTT Bridge** tails that file in real time and republishes each event
   to an MQTT broker, publishing Home Assistant discovery config so events
   appear automatically as sensors.

3. A **Home Assistant automation** listens on the MQTT topics for the three
   research events and drives a **WLED**-controlled LED strip inside the
   3D-printed lab.

## Components

### 1. 3D-Printed Model

**[Factorio Science Laboratory with Disco Science Mod](https://makerworld.com/en/models/558821-factorio-science-laboratory-with-disco-science-mod)**

The original model is designed for a specific LED controller. Instead, I used
an **ESP32 running [WLED](https://kno.wled.ge/)** — a popular open-source
firmware for addressable LED strips. Other LED controllers could work, but may
require significant modification to the Home Assistant automation.

### 2. Events Logger (Factorio Mod)

**[sstrang/events-logger](https://github.com/sstrang/events-logger)**

A fork of the [Events Logger](https://github.com/Ralnoc/factorio-events-logger)
mod that logs in-game events to a JSON file. This fork adds:

- Science pack ingredients (name + amount per unit) to the `RESEARCH_STARTED`
  event payload
- Factorio 2.1 compatibility

> Hopefully these changes will be merged upstream so the mod can be installed
> directly from the Factorio Mod Portal. Until then, clone the fork into your
> Factorio `mods` directory.

**The `RESEARCH_STARTED` event looks like this:**

```json
{
  "name": "logistics",
  "event": "RESEARCH_STARTED",
  "level": 1,
  "science_packs": [
    ["automation-science-pack", 1],
    ["logistic-science-pack", 1]
  ],
  "tick": 12345
}
```

### 3. MQTT Bridge

**[sstrang/factorio-events-logger-mqtt-bridge](https://github.com/sstrang/factorio-events-logger-mqtt-bridge)**

A command-line tool that tails the `game-events.json` file and republishes each
event to an MQTT broker. It publishes Home Assistant MQTT Discovery config so
all event types appear automatically as sensors under a "Factorio Server"
device.

```bash
python bridge.py /opt/factorio/script-output/game-events.json 192.168.1.50
```

> Tested on macOS. Should work on any platform with Python 3 and
> `paho-mqtt`. See the bridge repo for full documentation.

### 4. Home Assistant Automation

**[automation.yaml (GitHub Gist)](https://gist.github.com/sstrang/2ae7ab91881e04508e5af4a56c02367e)**

The automation listens on the three research MQTT topics and drives the WLED
light:

| Event | Behaviour |
|---|---|
| `research-started` | Cycles the LED through the colours of each science pack being consumed, using WLED's **Candle Multi** effect at speed 150 / intensity 255, swapping colours at a random interval between 0.5–2.0 seconds |
| `research-finished` | Blinks **solid white** five times (300ms on / 300ms off) |
| `research-cancelled` | Light **off** |

The automation uses queued mode with `wait_for_trigger` so that completion
blinks always finish in full before a new research's colour cycle begins —
important when using the in-game research queue, where one research finishes
and the next starts almost instantly.

## Requirements

- [Factorio](https://factorio.com) (2.0 or 2.1) running the Events Logger mod
- A 3D-printed lab model with an addressable LED strip
- An [ESP32](https://en.wikipedia.org/wiki/ESP32) flashed with
  [WLED](https://kno.wled.ge/) (0.14+) driving the LEDs
- An [MQTT broker](https://mosquitto.org/) (e.g. Mosquitto)
- [Home Assistant](https://www.home-assistant.io/) with the MQTT integration
  configured and the WLED device added
- A machine to run the MQTT Bridge (e.g. the Factorio server itself or a
  Raspberry Pi)

## Wiring Overview

```
┌──────────┐     game-events.json      ┌──────────────┐     MQTT      ┌──────────┐
│ Factorio │ ───────────────────────── │  MQTT Bridge │ ──────────── │   MQTT   │
│  + mod   │   (newline-delimited JSON)│  (Python)    │   (events)   │  Broker  │
└──────────┘                           └──────────────┘              └────┬─────┘
                                                                          │
                                                                     ┌────┴─────┐
                                                                     │   Home   │
                                                                     │ Assistant│
                                                                     └────┬─────┘
                                                                          │ WLED API
                                                                     ┌────┴─────┐
                                                                     │   WLED   │
                                                                     │  (ESP32) │
                                                                     └────┬─────┘
                                                                          │
                                                                     ┌────┴─────┐
                                                                     │ 3D Lab   │
                                                                     │ + LEDs   │
                                                                     └──────────┘
```

## Credits

- The 3D model is by [mrktmen](https://makerworld.com/@mrktmen) on MakerWorld.
- The [Events Logger](https://github.com/Ralnoc/factorio-events-logger) mod is
  by James Boylan, forked from
  [royvandongen](https://github.com/royvandongen/Factorio-Event-Logger-Mod).
- [WLED](https://kno.wled.ge/) by Aircoookie.
- The [Disco Science](https://mods.factorio.com/mod/DiscoScience) mod by
  _aidan_364 inspired this project's lighting behaviour.

## License

MIT
