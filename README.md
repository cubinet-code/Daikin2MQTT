# Daikin2MQTT

Control your Daikin Air Conditioner locally with Home Assistant using ESP32. Communicates directly with A/C using Serial Communication via S21/X50A port, replacing the original BRP072C42/BRP072C42-1 adapter from Daikin.

## Features

- Bi-directional control, syncs true A/C status to Home Assistant
- Support for Daikin wall-type A/C (S21 connector) and SkyAir (X50A connector)
- Automatic protocol detection (S21 tested first, then X50)
- Home Assistant MQTT Discovery with climate entity, sensors, switches, selects, and numbers
- Powerful Mode exposed as HA climate preset (boost)
- Local compressor power-limit (S21 demand control) — cap power draw without the Daikin cloud
- Built-in web UI for configuration and control
- OTA firmware updates (web upload or ArduinoOTA)
- Configurable temperature offsets for inside/outside sensors
- Auto-skips unsupported S21 commands after first NAK
- 8 languages: English, Danish, Spanish, French, Italian, Japanese, Chinese, Thai

## Controls

- **Climate entity**: Power, mode, target temperature, fan speed, vertical/horizontal swing
- **Presets**: Powerful mode (boost)
- **Switches**: Controller LED, beep, powerful mode, comfort airflow, outdoor quiet, streamer, econo, IR remote enable/disable
- **Selects**: Vertical vane, horizontal vane, indoor LED brightness, power limit (compressor demand control)
- **Numbers**: ON timer, OFF timer

Model-dependent controls (comfort, quiet, streamer, LED brightness, econo, power limit) appear only on units that support them; unsupported S21 commands are auto-skipped after the first NAK.

## Sensors

- Room temperature
- Outdoor unit temperature
- Coil temperature
- Humidity (model-dependent)
- Fan RPM
- Fan target RPM
- Compressor frequency
- Compressor load
- Real target temperature
- Louver angle
- Error code
- Timer status
- Energy meter (model-dependent)
- Uptime

## Configuration

All settings are configurable via the built-in web UI:

- **MQTT**: Host, port, credentials, topic, friendly name
- **WiFi**: SSID, password, hostname, OTA password
- **Unit**: Temperature unit (C/F), min/max temp, temp step, inside/outside temp offset, fan speed levels (3/5), mode support, update interval, LED, beep
- **Others**: HA autodiscovery, availability reporting, debug mode

## Debugging & Raw Protocol Access

The firmware exposes MQTT topics for probing the A/C protocol live, without flashing new firmware. All topics are under `daikin2mqtt/<device-name>/`. Source `mosquitto_pub`/`mosquitto_sub` from your broker to use them.

### Raw S21 command probe

Send arbitrary space-separated hex bytes; the reply is published with framing/CRC stripped. The first two bytes are the command (e.g. `46 36` = `F6`); any further bytes are the payload.

```bash
# Query F6 (auto-polled status); reply lands on .../recv/s21
mosquitto_sub -t 'daikin2mqtt/DAIKIN_BEDROOM/recv/s21' &
mosquitto_pub -t 'daikin2mqtt/DAIKIN_BEDROOM/send/s21' -m '46 36'
```

Reply format is `<cmd> <status> <hex-payload>`, where `<status>` is `OK`, `NAK`, or `TIMEOUT` (ACK-only writes report empty hex). The format is grep-friendly: `mosquitto_sub … | grep '^F6 OK'`. Requires the S21 protocol; works without debug mode.

### Batch S21 query

Probe several commands in one shot — space- or comma-separated 2-char command codes. One reply line per command is published to `.../recv/s21exp`.

```bash
mosquitto_pub -t 'daikin2mqtt/DAIKIN_BEDROOM/send/s21exp' -m 'F1 FN FP'
```

### Raw serial pass-through

`.../serial/send` writes raw bytes straight to the serial port and returns the reply (read until `ETX`) on `.../serial/recv`. This requires **debug mode** to be enabled and the unit connected.

### Debug mode

Publish `ON`/`OFF` to `.../debug/set` to toggle debug mode. While enabled, normal status polling is suspended and the device reports itself unavailable to Home Assistant, so you can probe the bus without the poll loop interfering. Status messages are published to `.../debug`.

```bash
mosquitto_pub -t 'daikin2mqtt/DAIKIN_BEDROOM/debug/set' -m 'ON'
```

Every poll is also logged to the in-memory log buffer, viewable at `http://<device-ip>/api/logs` — look for `S21 >> ` / `S21 << ` lines.

## Build

Requires [PlatformIO](https://platformio.org/).

```bash
# Build firmware
pio run -e wifikit-serial-esp32-s3

# Run tests
pio test -e native

# Upload via USB
pio run -e wifikit-serial-esp32-s3 --target upload

# Monitor serial
pio device monitor
```

The built firmware is automatically copied to `releases/` with a versioned filename.

### OTA Update

1. Open `http://<device-ip>/upgrade` in a browser
2. Upload the `.bin` file from `releases/`
3. Device restarts automatically

## Hardware

![wifikitserial](https://github.com/maxmacstn/Daikin2MQTT/blob/cec3a3b90637d39c1544068fdf562466f342510a/hardware-wifikitserial/header.jpg)

This repository is intended to be used with [WiFiKit Serial](https://wifikitserial.magiapp.me) module. The module consists of ESP32-S3, logic level converter, and buck converter.

If you would like to use your own hardware, you need:
- ESP32-S3 board
- Logic level converter (5V to 3.3V)
- Buck converter (14VDC supply to 3.3V)

### Pin Map

| Pin | Function |
|-----|----------|
| GPIO 43 | AC TX |
| GPIO 44 | AC RX |
| GPIO 5 | Power LED |
| GPIO 6 | Activity LED |
| GPIO 0 | Button (boot) |
| GPIO 14 | Buzzer |

## Links

- [ESP32-Faikin](https://github.com/revk/ESP32-Faikin) - Reference implementation
- [S21 Protocol](https://github.com/revk/ESP32-Faikin/blob/main/Manuals/S21.md) - Protocol documentation
- [esphome-daikin-s21](https://github.com/joshbenner/esphome-daikin-s21) - ESPHome implementation
- [Mitsubishi2MQTT](https://github.com/maxmacstn/mitsubishi2MQTT) - Inspiration for this project
