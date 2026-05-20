# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
(beta releases use a `-bN` suffix and are published as GitHub pre-releases).

## [Unreleased]

## [1.4-b6] - 2026-05-20

Cross-model fidelity, from Max's FTKQ12UV2S testing + the three-unit boot logs.
FTKD behavior is unchanged.

### Fixed
- **Indoor display brightness "High" now works.** The S21 code for High was wrong (`0x00`, which
  the unit ignores), so High behaved identically to Low. Captured the real codes from the remote
  (High=`0x04`, Low=`0x08`, Off=`0x0C`) and corrected both the write and read — High is now a
  distinct bright level. Live-verified on FTKD.
- **Special-mode entities no longer appear on units that can't use them.** On FTKC/FTKQ
  (which lack the FU00 special-mode extension), Powerful, Econo and Comfort Airflow used to show
  in Home Assistant but their writes silently bounced. They are now gated on the FU00 capability
  so they only appear where they actually work. FTKD keeps all of them; legacy v0/v1 units are
  unaffected (they keep the previous F6/F7 gate). LED brightness is *not* FU00-gated — it's a
  display setting on F6/D6 byte 3 that all v2 units support, so it stays available everywhere.

### Changed
- **Default fan-speed levels is now 5** (was 3) — most v2 wall units are 5-speed, so they now
  expose the full fan range out of the box. Units explicitly configured otherwise keep their
  setting; genuine 3-speed units can still select 3 in the web UI.

### Notes
- Streamer is intentionally left visible on all v2 units: FTKC (Flash Streamer line) has the
  hardware and the protocol can't distinguish it from FTKQ. Power Limit / demand control is
  unchanged — it works on FTKQ (compressor frequency responds).

## [1.4-b5] - 2026-05-20

A diagnostics overhaul following the full S21 read-surface mapping: drop the
sensors that carry no information, decode the ones that do, and fold capability
metadata onto the climate entity instead of scattering it across raw sensors.

### Added
- **Energy (FA)** diagnostic sensor — the `FA` register decoded as cumulative kWh
  (`device_class: energy`). Identical to the main Energy Meter (`FM`) on FTKD units
  but diverges on FTKC/FTKQ, so it is exposed separately as a long-term cross-check.
- **Graphable R-class sensors** `RA`/`RB`/`RF`/`Rg` with `state_class: measurement`.
  These single-field reads appear to mirror `F1` (RA≈power, RB≈mode); exposing them
  numerically lets Home Assistant statistics confirm or refute the duplication over
  time rather than on a single sample.
- **Climate diagnostic attributes** — the climate entity now carries `manual_url`,
  `special_modes` (powerful/econo/streamer availability from the `FU00` bitmap),
  `fk_raw` (the `FK` capability bitmap as ASCII), and `demand_available`.

### Changed
- **Manual link, FK and FU00** are no longer standalone diagnostic sensors; their
  information moved into the climate entity's attributes (stale entities are removed
  automatically via empty-retained discovery configs).
- **Louver Angle** and **Humidity** now declare `state_class: measurement` so Home
  Assistant records long-term statistics for them.

### Removed
- Diagnostic sensors that read constant/zero on every model tested: **Raw FL, Raw
  FS, Raw RW, Raw FU02**. `FU02` is also dropped from the poll cycle.

### Documentation
- Cross-checked the S21 read surface across three models (FTKD-ZV2S, FTKC-RV2S,
  FTKQ-UV2S); corrected the earlier mislabel of `FA` as an id-like field (it is the
  energy register, live-verified).

## [1.4-b4] - 2026-05-20

Refinements on top of 1.4-b3, informed by the Daikin technical manuals and a full
mapping of the S21 read surface.

### Added
- **Human-readable Error Code** — S21 fault codes are decoded to text in the Home
  Assistant discovery template (e.g. `A5 - Freeze-up protection`). `A5` is shown as
  the normal freeze-protection control action rather than a malfunction; unknown
  codes pass through verbatim.

### Changed
- **Compressor Frequency** now reports `device_class: frequency`.
- **Maximum setpoint** raised to **32 °C** (was 30) to match the FTKD-ZV2S manual's
  documented 16–32 °C range.

### Documentation
- Mapped the complete S21 read surface (F-class, R-class, and `FU` sub-command
  sweeps). Indoor humidity, real-time power, Mold Proof, model capacity-class, and
  Power-Limit readback are confirmed **not available** over S21 on these units.

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

[Unreleased]: https://github.com/cubinet-code/Daikin2MQTT/compare/v1.4-b6...HEAD
[1.4-b6]: https://github.com/cubinet-code/Daikin2MQTT/compare/v1.4-b5...v1.4-b6
[1.4-b5]: https://github.com/cubinet-code/Daikin2MQTT/compare/v1.4-b4...v1.4-b5
[1.4-b4]: https://github.com/cubinet-code/Daikin2MQTT/compare/v1.4-b3...v1.4-b4
[1.4-b3]: https://github.com/cubinet-code/Daikin2MQTT/compare/V1.2...v1.4-b3
[1.2]: https://github.com/cubinet-code/Daikin2MQTT/releases/tag/V1.2
