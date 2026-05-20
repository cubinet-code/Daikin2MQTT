# S21 Protocol Reference — Daikin FTKD-ZV2S

Canonical reference for the Daikin S21 serial protocol as implemented by Daikin2MQTT. Covers the
read/write surface, field encodings, per-model differences, and how the firmware maps each command.
Derived from live probing and cross-model captures. Version-by-version change history lives in
`CHANGELOG.md`, not here.

Coverage: protocol-v2 wall units **FTKD-ZV2S** (`7B91`, primary, fully probed), **FTKC-RV2S**
(`A7D0`) and **FTKQ-UV2S** (`DD11`) (cross-checked). Confidence is noted per item; "verified" =
exercised on a live FTKD unit.

---

## 1. Conventions

- **Framing direction.** `F`-class commands are reads; the reply command byte is `G` (F1→G1).
  `R`-class are reads with reply `S` (RH→SH). `D`-class are **writes** (D6, D7, …).
- **ASCII-offset values.** Most payload bytes are ASCII digits offset from `0x30` (`'0'`). A bit
  field of value `n` is sent as `'0' + n`; decode as `byte & mask`.
- **Reversed-digit temperature** (R-class temps): digits little-endian with a trailing sign,
  `÷10 °C`. e.g. `35 34 32 2B` = `"542+"` → 24.5 °C.
- **Reversed-nibble counter** (FM, FA energy): `hex(p[3])<<12 | hex(p[2])<<8 | hex(p[1])<<4 | hex(p[0])`,
  then `÷10 = kWh`. e.g. `01C5` → 453 → 45.3 kWh.
- Payloads are shown as colon/space-separated hex; offsets are 0-based into the **data** portion
  (after the framing/command bytes).

## 2. Probing over MQTT (no firmware change)

- Send: publish space-separated hex to `daikin2mqtt/<dev>/send/s21` — e.g. `46 36` = "F6",
  `46 55 30 30` = "FU"+"00" (FU sub-command 00).
- Receive: read `daikin2mqtt/<dev>/recv/s21` — self-labelled `F6 OK <hex>` / `NAK` / `TIMEOUT`.
- **Only probe `F`(→G) and `R`(→S) reads. Never sweep `D` — those are writes** and mutate unit state.
- Space queries ≥ 2 s apart to avoid colliding with the firmware's auto-poll.

---

## 3. F-class commands (reads, F→G)

### 3.1 Decoded
| Cmd | Payload (FTKD) | Meaning / decode | Firmware | Confidence |
|---|---|---|---|---|
| F1 | `31 33 4C 35` | Basic state: byte0 power, byte1 mode, byte2 setpoint (`(b−28)×0.5 °C`), byte3 fan | core state | verified |
| F2 | `34 30 00 80` | Capabilities: byte0 bit2 = vertical-swing, bit3 = horizontal-swing | gates swing entities | verified |
| F3 | — | Timer + powerful: byte0 enable, byte1 on-timer, byte2 off-timer, byte3 bit1 powerful (authoritative timer source) | timers, powerful fallback | verified |
| F4 | — | Error code: byte0 bit0 = fault present, byte1 = division/detail nibbles | Error Code sensor | verified |
| F5 | `30 30 30 80` | Swing state: byte0 bit0 vertical, bit1 horizontal | swing state | verified |
| F6 | `30 30 30 30` | Special modes + LED brightness — see §6.1 | powerful/comfort/quiet/streamer/LED | verified |
| F7 | `30 30 4C 30` | Demand (power-limit) + econo — see §6.2 | econo, demand | verified |
| F8 | `30 32 30` | Protocol version: `byte1 & ~0x30` (=2 here) | gates v2 features | verified |
| F9 | `B0 B4 FF 30` | byte0 coarse room temp (firmware uses RH instead); byte2 humidity (`0x30+rh/5`, `0xFF`=no sensor) | humidity (if present) | verified |
| FA | reversed-nibble | **Energy register**, ÷10 kWh. =FM on FTKD; diverges on FTKC/FTKQ (see §9) | "Energy (FA)" cross-check sensor | verified |
| FC | `37 42 39 31` | Model id "7B91" (byte-reversed "19B7") — MCU/PCB firmware id, **not** capacity class (§8) | model→family lookup, logged on first parse | verified |
| FK | `71 73 35 31` | Capability bitmap "qs51" — see §6.3 | `fk_raw` + `demand_available` attrs | medium |
| FM | reversed-nibble | Energy meter, ÷10 kWh | Energy Meter sensor | verified |
| FU\<nn\> | see §6.4 | v2 extension sub-commands (payload-selected); FTKD-only | capability flags, FU04 diag | mixed (§6.4) |

### 3.2 Opaque / undecoded (exposed raw for graphing)
| Cmd | Payload (FTKD) | Notes |
|---|---|---|
| FB | `30 33 33 31` | Opaque profile byte; no known semantics |
| FG | `33 41 30 30` | Opaque profile byte |
| FN | `30 30 30 30` | Constant; reads `"0000"` on FTKD, `"1000"` on FTKQ |
| FP | `34 42 30 30` | Opaque profile byte |
| FQ | `36 35 30 30` | Opaque profile byte |
| FT | `31 30 30 30` | Constant `"1000"` |
| FR | `30 30 30 30` | Constant `"0000"` on all models tested |
| FV | `33 37 83 30` | byte2 `0x83` binary; **FTKD-only**, undecoded |

### 3.3 No-data / unsupported
| Command(s) | Behaviour | Exposed |
|---|---|---|
| `FL`, `FS` | constant zero (`0000`) on all models | no (dropped — useless) |
| `FN`, `FR` | constant (see §3.2) | yes (raw diag) |
| `FD`, `FH`, `FJ` | TIMEOUT | no |
| `FE`, `FI`, `FO`, `FW`, `FX`, `FY`, `FZ` | NAK | no |
| `FU05` | NAK on v2 (full ASCII model name, v3-only, e.g. `FTXA35C2V1BS`) | no |

## 4. R-class commands (reads, R→S)

| Cmd | Payload (FTKD) | Meaning / decode | Firmware | Confidence |
|---|---|---|---|---|
| RH | `542+` | Indoor/room temperature (÷10) | room temp | verified |
| RI | `001+` | Coil temperature (÷10) | coil temp | verified |
| RX | `042+` | Real target temperature (adjusted setpoint) | target temp sensor | verified |
| RN | `050+` | Measured louver angle (÷10°) | louver angle sensor | verified |
| Ra | `062+` | Outdoor-side / aux temperature (÷10, ≈26.0) | outside temp | verified |
| Rd | reverse-digit | Compressor frequency (Hz); some units always read 0 | compressor freq (if non-zero seen) | verified |
| Re | `050` | Humidity (reverse-ASCII); `050` = **no-sensor placeholder** | not exposed (placeholder) | verified |
| RG | single byte | Fan speed (alternative); differs per model (FTKD `5`) — left unpolled, redundant with F1 fan | not polled | verified |
| RK | `570`-form | Indoor fan **target** RPM ×10 (tracks fan setpoint) | Fan Target RPM sensor | verified |
| RL | `470`-form | Fan RPM (measured, ×10) | Fan RPM sensor | verified |
| Rb | `500` | Compressor load signal | Compressor Load sensor | low |
| RA | `1` | Single byte; mirrors F1 byte0 (power) | graphable diag sensor | likely-dup |
| RB | `3` | Single byte; mirrors F1 byte1 (mode) | graphable diag sensor | likely-dup |
| RF | `00` | Two bytes; meaning unknown | graphable diag sensor | unknown |
| Rg | `1` | Single byte; meaning unknown | graphable diag sensor | unknown |
| RW | `00` | Constant zero on all models; no handler | not exposed (still polled for query-index integrity) | unknown |
| RM | `611+` / `770+` | Present on FTKC/FTKQ (not observed on FTKD); likely timer/mode state | not exposed | unknown |

`RA`/`RB`/`RF`/`Rg` are exposed as numeric diagnostic sensors (`state_class: measurement`) so HA
long-term statistics can confirm/refute the F1 duplication over time.

## 5. D-class commands (writes, D)

| Cmd | Payload bytes | Sets | Notes |
|---|---|---|---|
| D2 | power / mode / temp / fan | Basic state | mirror of F1 |
| D3 | byte3 bit1 | Powerful (fallback when F6 unsupported) | |
| D6 | b0 powerful/comfort/quiet · b1 streamer · b3 LED brightness | F6 special modes + display brightness | brightness codes per §6.1 |
| D7 | b0 demand · b1 econo | F7 demand + econo | demand is write-only (not echoed back, §6.2) |

---

## 6. Decoded field maps

### 6.1 F6/D6 — special modes + display brightness
- byte0: powerful `& 0x02` · comfort (ceiling airflow) `& 0x40` · quiet (outdoor) `& 0x80`
- byte1: streamer `& 0x80`
- byte3: **display (LED) brightness, field `& 0x0C`**:

  | Level | field | on-wire byte (`'0'`+field) |
  |---|---|---|
  | High | `0x04` | `0x34` |
  | Low | `0x08` | `0x38` |
  | Off | `0x0C` | `0x3C` |

  `0x00` is **invalid** — the unit ignores it (so it does not act as "High"). `0x08` is the
  Low-brightness bit, not a sensor flag. Verified bidirectionally on FTKD (write + read-back + visual).

### 6.2 F7/D7 — demand (power limit) + econo
- byte0 demand: `100 − (byte0 − '0')`, treated as off when byte0==`'1'`. **Write-only / optimistic**:
  the unit never echoes the active cap (G7 byte0 always reads `100`/"off"; setting 40% changed
  neither byte0 nor byte2). Demand *does* take effect — compressor frequency responds.
- byte1 econo: `& 0x02`.
- byte2: `0x4C` (`'L'`) on FTKD — **FTKD-only static constant** (protocol default is `0x30`;
  FTKC/FTKQ read `0x30`). Does not track demand or econo; meaning unknown.

### 6.3 FK — capability bitmap (per-model, medium confidence)
FTKD `71 73 35 31` ("qs51"):
- byte0 `0x71`: acled(b2)=0, land(b3)=0, en_rtemp_a-disable(b6)=1
- byte1 `0x73`: elec(b0)=1 (energy reporting present), temp_rng(b2)=0, m_dtct(b3)=0 (no motion/eye sensor)
- byte2 `0x35`: jp(b0)=1, no-humidity-setting(b1)=0, fan-controls(b2)=1, disp_dry(b3)=0
- byte3 `0x31`: demand-mode available(b0)=1

**The bit decode is per-model and does not transfer** (FTKC reads `0:11`, FTKQ `0251`). The only
bit consistent across all three is **byte3 b0 = demand available** (`0x31` on all). Firmware
therefore exposes only `fk_raw` (the ASCII string) + `demand_available` as climate attributes, and
does not publish the other bits as labelled flags (would mislabel FTKC/FTKQ). Capability gating is
done empirically (feature NAKs on write), not derived from FK.

### 6.4 FU sub-commands (v2 extension; FTKD-only)
Sent as `FU` + a 2-char ASCII selector echoed back at the payload start. Only `00/02/04` are valid
on these v2 units (swept `00`–`0F+`; the rest NAK/TIMEOUT). **All three NAK on FTKC and FTKQ.**
- **FU00** `33 33 33 33 33 33 30…` — special-mode **availability** bitmap (`0x33`=available; 6 modes
  available): byte0 powerful, byte1 econo, byte5 streamer (others unmapped). Drives the firmware
  capability flags and the `special_modes` climate attribute, and gates special-mode entities (§7).
  *(verified)*
- **FU02** `A0 A0 30 30 30 FF…` — heat-mode temp limits: byte1 `atlmt=(b−0xA0)/2`=0.0,
  byte2 `hmlmt=(b−0x30)/2+10`=10.0. Not useful on a cooling-only unit; not exposed. *(decoded, low value)*
- **FU04** — per-device cumulative Wh; left raw (not decoded by the firmware). The value is
  non-monotonic, so it is **not** a lifetime-kWh counter. *(raw / unverified)*

---

## 7. Firmware capability gating (current behaviour)

Entities are exposed only when the unit supports the underlying command. The discriminator for the
**special-mode extension** is FU00: FTKD answers it (so `_fu00Seen` is set, and
`_hasPowerful/_hasEcono/_hasStreamer` come from its availability bitmap); FTKC/FTKQ NAK it, and
their D6 special-mode writes bounce.

Flags: `_fu00Seen` = FU00 parsed successfully (extension present). `_hasPowerful/_hasEcono/
_hasStreamer` = the corresponding FU00 availability bytes. `_ledBrightnessSeen` = G6 returned a
4-byte payload. `v` = protocol version (F8).

| Feature | Gate | FTKD | FTKC/FTKQ | legacy v0/v1 |
|---|---|---|---|---|
| powerful | `v<2 ? F6 : F6 && _hasPowerful` | show | hide | F6 (unchanged) |
| econo | `v<2 ? F7 : F7 && _hasEcono` | show | hide | F7 (unchanged) |
| comfort / quiet | `v2 && F6 && _fu00Seen` | show | hide | n/a (v2-only) |
| LED brightness | `v2 && F6 && _ledBrightnessSeen` | show | **show** | n/a |
| streamer | `v2 && F6` | show | **show** | n/a |
| demand / power-limit | `F7` | show | show | show |

- **LED brightness is not FU00-gated** — it's a display setting on F6/D6 byte3, present on every v2
  unit that returns a 4-byte G6, not part of the FU00 extension.
- **Streamer is left ungated**: FTKC physically has a streamer, and FU00-NAK can't distinguish FTKC
  from FTKQ, so a gate would wrong-hide it. FTKQ shows a cosmetic N/A entity until a model-aware
  gate is justified.
- **demand/power-limit** works on FTKQ (compressor frequency drops), so it stays on the F7 gate.
- Default `fan_speed_levels` is 5 (most v2 wall units are 5-speed); saved settings are preserved.

⚠️ Only FTKQ special-mode writes are directly tested; FTKC hiding is inferred from its FU00-NAK and
is not write-confirmed.

---

## 8. Not available over S21

Confirmed unavailable on these units (no firmware action possible):
- **Indoor humidity** — `F9` byte2 = `0xFF` (no sensor) and `Re` = `050` placeholder, on all three
  models. (Dashboard humidity comes from a separate sensor.)
- **Real-time power** — no register; only the FM/FA cumulative energy counters exist.
- **Mold Proof / self-clean** — IR-remote only. Toggling it on the remote produced zero change in
  F5/F6/F7/FU00; it is not on the S21 bus.
- **Model capacity class** — `FC` is an MCU/PCB firmware id, not a capacity code; no model→class
  mapping exists.
- **Power-limit readback** — demand is write-only/optimistic (§6.2).

## 9. Cross-model comparison

Three protocol-v2, vertical-swing-only (F2 vSwing=1, hSwing=0) wall units. Differences are
architectural (model traits), not runtime state.

| Trait | FTKD-ZV2S | FTKC-RV2S | FTKQ-UV2S |
|---|---|---|---|
| FC code | `7B91` | `A7D0` | `DD11` |
| FU00 / FU02 / FU04 | **supported** | all NAK | all NAK |
| FV | `33 37 83 30` | NAK | NAK |
| F7 byte2 | `0x4C` ('L') | `0x30` | `0x30` |
| FK bitmap | `71 73 35 31` "qs51" | `30 3A 31 31` "0:11" | `30 32 35 31` "0251" |
| F9 byte1 | `0xB4` | `0xFF` | `0xB4` |
| F9 byte2 (humidity) | `0xFF` (none) | `0xFF` (none) | `0xFF` (none) |
| FM | (not captured) | `42 41 34 41` "BA4A" | `31 42 45 31` "1BE1" |
| FA (energy register) | `01C5`→45.3 kWh | `39 33 32 31`→466.5 | `44 36 45 31`→778.9 |

- **FTKD-ZV2S is the outlier**, not the baseline: it is the only one that answers the FU
  sub-commands and FV. The firmware's FTKD extension handling NAK-degrades cleanly on the others
  (self-skip on first NAK).
- **`F7 byte2 = 0x4C` is FTKD-only** (FTKC/FTKQ read the `0x30` default).
- **No humidity sensor on any** of the three (`F9` byte2 = `0xFF` universally).
- **FA is energy-like but its register differs by model**: identical to FM on FTKD (verified:
  Guestroom 45.3 kWh, Bedroom 11.9, tracked over 8 h, both equal to `energyMeter`); but FTKC FA
  `466.5` ≠ FM `4215.5`, and FTKQ FA `778.9` ≈ FM `785.7`. Hence FA is exposed as a separate
  diagnostic cross-check, not as the meter.
- **FM** returns 4 undecoded data bytes on FTKC/FTKQ (`BA4A`, `1BE1`); FTKD's FM payload was not captured.

### FC model map
| FC code | Family | Manual |
|---|---|---|
| `7B91` | FTKD-ZV2S | `3P730851-16` |
| `A7D0` | FTKC-RV2S | `3P730851-17` |
| `DD11` | FTKQ-UV2S | `3P658678-13` |

## 10. Open unknowns

- **FK per-model bits** — the FTKD "qs51" decode does not transfer; FTKC `0:11` and FTKQ `0251`
  each need independent correlation. Only byte3 b0 (demand) is common.
- **FM 4-byte payload** on FTKC/FTKQ — meaning unknown; FTKD's FM payload was not captured.
- **FV** byte2 `0x83` — undecoded.
- **FB / FG / FP / FQ / FT** — opaque profile bytes; no known semantics.
- **RF / Rg / RM / RW** — reads with no assigned meaning.
- **RG** differs per model (FTKD `5`, FTKC `33`, FTKQ `41`) but carries no obvious info.
