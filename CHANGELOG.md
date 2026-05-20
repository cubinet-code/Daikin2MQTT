# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
(beta releases use a `-bN` suffix and are published as GitHub pre-releases).

## [Unreleased]

## [1.4-b3] - 2026-05-20

Largest update since 1.2: full support for newer S21 "v2" wall units
(e.g. FTKD-ZV2S / model 7B91), many new Home Assistant entities, and a much
snappier control experience.

### Added
- **Power Limit** select (`Off / 80% / 70% / 60% / 40%`) — caps the compressor's
  power draw over S21 demand control; a local, cloud-free equivalent of the
  throttle behind Daikin Thailand's "Bill on Demand". Verified on FTKD-ZV2S.
- **Comfort airflow**, **Outdoor quiet**, **Streamer**, and **Econo** modes
  (read + write) on S21 v2 units; Comfort is also exposed as a climate preset.
- **Indoor LED brightness** control (High / Low / Off).
- **Settable ON/OFF timers** as Home Assistant number entities.
- **Indoor humidity** sensor (model-dependent).
- **Long-term statistics** for Fan RPM, Compressor Frequency, Fan Target RPM, and
  Compressor Load (reported as measurements so Home Assistant graphs and retains
  history).
- **Friendly model name** and a **per-model manual link** in the web UI and the
  Home Assistant device page.
- **Raw S21 probing** over MQTT (`send/s21`, `send/s21exp`) for diagnostics.
- **CI release automation** — GitHub Actions builds the firmware and publishes a
  release with the OTA binary on tag.

### Changed
- **S21 protocol v2 support** for newer wall units.
- **Instant command feedback** — removed the ~25 s lag between issuing a command
  and seeing the new state in Home Assistant; the web UI and physical button echo
  changes immediately.
- Honors the configured update interval, with a shorter settle delay after a set.
- Moved Compressor Load, Louver Angle, and Timer from Diagnostics into Sensors.
- Build binaries are no longer committed to the repository (now published as
  release assets).

### Fixed
- **IR-remote lock-out** on newer units (model 7B91, S21 v2).
- **Fan-speed mapping** for 5-speed units.
- **Humidity sensor detection** — only exposed on units that report it.
- Hardened input handling — invalid setter values are rejected and no-op changes
  are silenced.
- Streamer entity icon (it's a plasma ionizer, not a weather function).

## [1.2] - 2025-01-22

Previous published release.

[Unreleased]: https://github.com/cubinet-code/Daikin2MQTT/compare/v1.4-b3...HEAD
[1.4-b3]: https://github.com/cubinet-code/Daikin2MQTT/compare/V1.2...v1.4-b3
[1.2]: https://github.com/cubinet-code/Daikin2MQTT/releases/tag/V1.2
