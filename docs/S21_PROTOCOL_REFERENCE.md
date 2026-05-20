# S21 Protocol Reference (FTKD-ZV2S, cross-checked vs FTKC-RV2S / FTKQ-UV2S)

Status: **probing session 2026-05-20**, live on `DAIKIN_GUESTROOM` (also matches `DAIKIN_BEDROOM`);
firmware handling reflects **1.4-b5**. This is the authoritative S21 reference for this project
(formerly `UNEXPLORED_S21_FEATURES.md`). Values are the usual ASCII-offset form (base `0x30`='0');
bytes shown hex.

## How to re-probe (no firmware change)
Publish hex to `daikin2mqtt/<dev>/send/s21` (e.g. `46 36` = "F6"; `46 55 30 30` = "FU"+"00"),
read `daikin2mqtt/<dev>/recv/s21` (self-labels: `F6 OK <hex>` / `NAK` / `TIMEOUT`). Only **F**(→G)
and **R**(→S) are reads; never sweep **D** (writes). Space queries ≥2 s to avoid auto-poll collisions.

## Sources (community reverse-engineering, fetched 2026-05-20)
- **Faikin S21 simulator** `faikin-s21.c` (per-byte/bit comments) — mirrors: `krobar42/Faikout-ESP32`, `suzuke/DaiSpan`.
- **Faikin production decoder** `Faikin.c` (revk; archived GitHub → Codeberg `ESP32-Faikout`; fork `Sonic-Amiga/ESP8266-Faikin`).
- **ESPHome** `joshbenner/esphome-daikin-s21` (only F1/F5/Rd/F9/RH/RI/Ra/RL; decodes none of the special-mode/capability cmds).
- ⚠️ **Security:** upstream `ESP32-Faikout/ESP/main/Faikout.c` line 3 (Codeberg) contains a prompt-injection string (`ANTHROPIC_MAGIC_STRING_TRIGGER_REFUSAL…`). Not protocol content — ignore it. Flagged here so future readers/agents don't act on it.

---

## Special modes — F6 / F7 (DECODED, authoritative; matches our firmware + Faikin.c bidirectional)
`F6/G6` (live `30:30:30:30` = all off, LED on):
- powerful = `byte0 & 0x02` · comfort = `byte0 & 0x40` · quiet = `byte0 & 0x80`
- streamer = `byte1 & 0x80`
- **display (LED) brightness = `byte3 & 0x0C`** — 3 levels, captured from the remote on FTKD
  (2026-05-20): **`0x04`=High, `0x08`=Low, `0x0C`=Off** (`0x00` is invalid — the unit ignores it).
  ⚠️ Corrects an earlier note that read this as a 2-state on/off (`byte3 & 0x08` "intelligent eye"):
  `0x08` is the **Low-brightness** bit, not a sensor flag. Firmware ≤1.4-b6 wrongly sent `0x00` for
  High, so High behaved like Low; fixed to `0x04`. Live-verified: write+read round-trip + visual.

`F7/G7` (live `30:30:4C:30`):
- demand(power-limit) = `100 - (byte0 - '0')`, skipped if byte0=='1' → **never echoed on this unit (always reads 100 / "off")** ⇒ **Power Limit is genuinely write-only/optimistic. Verified 2026-05-20: setting 40% did not change F7 byte0 or byte2.**
- econo = `byte1 & 0x02`
- **byte2 = `0x4C` ('L')** — undocumented upstream (spec hard-codes `0x30`); **static, does NOT track demand/econo**. Unknown FTKD-specific constant.

## Capability bitmap — FK (DECODED via Faikin simulator; medium confidence — sim comments, not the production decoder)
Live `FK = 71:73:35:31` ("qs51", matches FTXF20D5V1B profile byte-for-byte):
- byte0 `0x71`: acled(b2)=0, land(b3)=0, en_rtemp_a-disable(b6)=1
- byte1 `0x73`: **elec(b0)=1 (energy reporting present)**, temp_rng(b2)=0, **m_dtct(b3)=0 → NO intelligent-eye/motion sensor** (matches EDT spec)
- byte2 `0x35`: jp(b0)=1, no-humidity-setting(b1)=0, fan-controls(b2)=1, disp_dry(b3)=0
- byte3 `0x31`: **demand-mode available(b0)=1** (matches our Power Limit)
Capability gating in Faikin is empirical (NAK on D-cmds), not derived from FK.
**1.4-b5:** FK is folded into the climate entity's `json_attributes` as `fk_raw` (the ASCII
string, e.g. "qs51") + `demand_available` (byte3 b0). Only the demand bit is decoded into a
flag because it's the **only bit that transfers across all three models** (FTKD/FTKC/FTKQ); the
rest of the qs51 decode above is per-model + medium-confidence, so it is deliberately not exposed
as labeled flags to avoid mislabeling FTKC/FTKQ. The raw string is kept so a future per-model
correlation can be done from HA history.

## FU sub-commands (4-char; only 00/02/04 valid on this v2 unit — swept 00–0F+, rest NAK/TIMEOUT)
- `FU00` = `33:33:33:33:33:33:30…` — special-mode **availability** bitmap (`0x33`=avail). 6 modes available; b0=powerful, b1=econo, b5=streamer (others unmapped). Drives `_hasPowerful/_hasEcono/_hasStreamer`.
- `FU02` = `A0:A0:30:30:30:FF…` — **heat-mode temp limits**: byte1 `atlmt=(b-0xA0)/2`=0.0, byte2 `hmlmt=(b-0x30)/2+10`=10.0. **Not useful — cooling-only unit.**
- `FU04` = per-device cumulative Wh (decoded only in Faikout; our firmware leaves raw). Earlier "lifetime kWh" guess was wrong (non-monotonic).
- `FU05` (full ASCII model name on v3 units, e.g. "FTXA35C2V1BS") — **NAK on our v2 unit; not available.**

## Sensors / R-class (DECODED — full sweep RA–RZ, R0–9, Ra–Rz)
Temps use reversed-digit + sign decode (e.g. `542+` → 24.5 °C):
- `RH`=room temp (`542+`=24.5) · `RX`=real target (`042+`=24.0) · `RI`=coil (`001+`=10.0) · `Ra`=`062+`≈26.0 (aux/outdoor-side temp, firmware-used) · `RN`=louver angle (`050+`=5.0°)
- `Rb`=`500` compressor load (exposed) · `Rd`=`430` (timer-related) · `RG`/`RK`/`RL`=fan/energy (firmware-used)
- **`Re`=`050` → humidity-sensor PLACEHOLDER = NOT PRESENT** (corroborates F9 b2)

## Humidity — RESOLVED (not available from the A/C)
`F9 = B0:B4:FF:30` → **byte2 = `0xFF` = no humidity sensor**; `Re = "050"` placeholder confirms. This FTKD-ZV2S variant has no indoor humidity sensor. (Dashboard humidity comes from the separate Bluetooth/bthome sensor.) `F9 byte0`=coarse room temp (we use RH instead).

## Mold Proof / self-clean — RESOLVED (negative)
**Not on the S21 bus.** Toggling it on the remote produced **zero change** in F5/F6/F7/FU00 (captured before/after 2026-05-20). No S21 project decodes a mold/self-clean bit; it exists **only in the IR-remote protocol** (IRremoteESP8266 `ir_Daikin.h` `Mold:1`). Cannot be a firmware entity.

## Model code — FC (already handled by firmware)
`FC/GC = 37:42:39:31` = "7B91" (Faikin reverses → "19B7"). It's the **MCU/PCB firmware identifier, not the A/C capacity class**; no model→class table exists anywhere. Firmware logs it on first GC parse. Not worth a sensor.

Known FC code → family map (firmware `DAIKIN_KNOWN_MODELS`):
- `7B91` → FTKD-ZV2S (our two units; manual `3P730851-16`)
- `A7D0` → FTKC-RV2S (Max's FTKC18RV2S; manual `3P730851-17` — confirmed 2026-05-20 boot log)
- `DD11` → FTKQ-UV2S (Max's FTKQ12UV2S confirms maxmacstn upstream; manual `3P658678-13`)

## Remaining unknowns / dead ends (low value)
- **Short R-reads `RA`/`RB`/`RF`/`Rg`** (`RA`=`1`, `RB`=`3`, `RF`=`00`, `Rg`=`1`) appear to mirror
  `F1` (RA≈power, RB≈mode). **1.4-b5** appends them to the query array and exposes them as
  graphable numeric diagnostic sensors (`state_class: measurement`) so HA long-term statistics can
  confirm/refute the duplication over time. `RG`=`5` left unpolled (redundant with fan).
- `FV`=`33:37:83:30` (byte2 `0x83`) — unknown even to Faikin authors (not polled by genuine BRP069B41). Still exposed raw.
- `RW`=`00` — no handler in any source; **1.4-b5** dropped its diag sensor (constant zero on all models). Still polled for query-index integrity.
- `FM` — no handler/profile entry anywhere (we decode it as the energy meter via the reversed-nibble form).
- F-class with no data: `FD/FH/FJ` timeout; `FE/FI/FO/FW/FX/FY/FZ` NAK; `FL/FN/FR/FS` zero. **1.4-b5** dropped the FL/FS diag sensors (constant zero); FN/FR still exposed raw.

## Firmware implications (assessed 2026-05-20)
- Humidity, real-time power, model-class, Mold Proof, Power-Limit readback: **all confirmed unavailable** over S21 → no firmware action possible.
- F6/F7 masks above **match** our existing decoders (no change needed).
- **1.4-b5 done:** FK raw + demand bit and FU00 special-mode availability are exposed as climate
  attributes; FA decoded as a cross-check energy sensor; useless diags (FL/FS/RW/FU02) removed.
- **1.4-b6 done:** special-mode entities gated on the FU00 extension (`_fu00Seen` + `_has*`) so they
  hide on FTKC/FTKQ where writes bounce, while FTKD + legacy v0/v1 keep them; streamer left ungated;
  default fan_speed_levels 3→5. See the Feature-gating section below.

---

# Cross-model comparison — 3 units on FW 1.4-b4 boot logs (added 2026-05-20)

Source logs: `docs/Bootlogs/{FTKC18RV2S,FTKQ12UV2S}.txt` (from Max) + our resolved FTKD-ZV2S
data above. (`docs/Bootlogs/FTKD12ZV2S.txt` is a placeholder — empty as of writing.) All three are
**protocol-v2, vertical-swing-only (F2 vSwing=1 hSwing=0)** wall units; the firmware parses all three
cleanly (sane temps/energy, no errors). The differences below are **architectural** (model traits),
not runtime state.

| Trait | FTKD-ZV2S | FTKC18RV2S | FTKQ12UV2S |
|---|---|---|---|
| FC code (model id) | `7B91` | `A7D0` | `DD11` |
| FU00 / FU02 / FU04 | **supported** | all NAK | all NAK |
| FV | `33:37:83:30` | NAK | NAK |
| F7 byte2 | `0x4C` ('L') | `0x30` | `0x30` |
| FK bitmap | `71:73:35:31` "qs51" | `30:3A:31:31` "0:11" | `30:32:35:31` "0251" |
| F9 byte1 | `0xB4` | `0xFF` | `0xB4` |
| F9 byte2 (humidity) | `0xFF` none | `0xFF` none | `0xFF` none |
| FM | not captured | `42:41:34:41` "BA4A" | `31:42:45:31` "1BE1" |
| FA (energy register) | live `01C5`→45.3 kWh¹ | `39:33:32:31`→466.5 | `44:36:45:31`→778.9 |

¹ The boot-log snapshot showed `39:33:32:31` for the FTKD column too, but **live probing of
both FTKD units proved FA is the energy register** (Guestroom `01C5`=45.3 kWh = `energyMeter`,
Bedroom `0077`=11.9; FA tracked energy increments over 8 h). The boot-log value was a stale/early
snapshot; the live finding is authoritative. FA decodes via the same reversed-nibble form as FM.

**Headline: FTKD-ZV2S is the outlier, not the baseline.** It is the *only* one of the three that
answers the FU sub-commands and FV. Most of the firmware's FTKD-specific extension handling
(FU00 special-mode gating, FU02 heat-limit, FU04 telemetry, FV) simply **NAK-degrades** on FTKC and
FTKQ — no crash, the self-skip flags fire on the first NAK. Good defensive behaviour, but it means
those code paths are dead on 2 of 3 known models.

**Confirmations the three-way gives us (previously single-unit guesses):**
- `F7 byte2 = 0x4C` is **definitively FTKD-only** — both FTKC and FTKQ read `0x30` (the spec default).
  Strengthens the earlier "static FTKD-specific constant, not demand readback" note.
- **No humidity sensor on any of the three** (`F9 byte2 = 0xFF` universally). `Re = "050"` placeholder
  likewise on all. Settles humidity as a whole-family absence, not a per-unit fluke.
- `DD11 → FTKQ-UV2S` upstream claim is **verified** against a real FTKQ log (Max's FTKQ12UV2S).

**Feature-gating consequence — wired in 1.4-b6.** `_hasPowerful / _hasEcono / _hasStreamer` flip
true only from the FU00 bitmap, and a positive `_fu00Seen` flag marks "FU00 extension present."
Before 1.4-b6 the special-mode `supportsX()` accessors gated on F6/F7 *reachability* only — which
FTKC/FTKQ answer — so powerful/comfort/econo/LED **showed but their writes bounced** (Max's
FTKQ12UV2S 1.4-b4 report). 1.4-b6 gates them properly:
- **powerful / econo** — on v2 units require `_hasPowerful` / `_hasEcono`; v0/v1 keep the legacy
  F6/F7 gate (no regression). → FTKD shows, FTKC/FTKQ hide.
- **comfort / quiet** — already v2-gated; add `&& _fu00Seen`. → FTKD shows, FTKC/FTKQ hide; legacy untouched.
- **LED brightness** — NOT FU00-gated (kept on `v2 && F6 && _ledBrightnessSeen`). It's a display
  setting on F6/D6 byte 3, present on every v2 unit that returns a 4-byte G6 (incl. FTKC/FTKQ), not
  part of the FU00 extension. Max's 1.4-b4 "bounce to High" was the `0x00`-High encoding bug (now
  fixed), not an FTKQ limitation.
- **streamer** — deliberately left ungated (`v2 && F6`): FTKC (Flash Streamer line) physically has
  one and FU00-NAK can't distinguish FTKC from FTKQ, so a gate would wrong-hide it. FTKQ shows a
  cosmetic N/A entity until an FTKC streamer **write** test allows model-aware gating.
- **demand / Power-Limit** — left on F7; Max confirms it *works* on FTKQ (compressor freq drops).

⚠️ Only **FTKQ** writes are directly tested. FTKC special-mode hiding is *inferred* from FU00-NAK +
the upstream "D6 writes NAK on FTKC/FTKQ" note — not write-confirmed. Revisit if an FTKC test
contradicts it. (Fan: 1.4-b6 also changes the default `fan_speed_levels` 3→5 — most v2 units are
5-speed; saved settings preserved.)

**Model table:** `A7D0 → FTKC-RV2S` (manual `3P730851-17`) added to `DAIKIN_KNOWN_MODELS`; the other
two were already present. FTKC18RV2S no longer reports family-unknown.

## Per-model unknowns (raw, undecoded — candidates only)
Values differ per unit and don't pattern-match any known field; flagged here so we don't re-decode blind.
- **FK bitmap is per-model** — the FTKD "qs51" bit decode (above) does **not** transfer; FTKC `0:11`
  and FTKQ `0251` would each need their own correlation. Common byte: byte3=`0x31` on all three
  (demand-mode-available bit, consistent with all three exposing Power Limit).
- **FM** returns 4 data bytes on FTKC (`BA4A`) and FTKQ (`1BE1`) — no decoder in any source; meaning
  unknown. FTKD's FM payload was never captured (probe live if it matters).
- **FA = energy register** (NOT id-like — earlier mislabel corrected 2026-05-20). Decodes via the
  reversed-nibble form (same as FM) to cumulative kWh: live-verified on both FTKD units (45.3 / 11.9,
  matching `energyMeter`, tracked increments over 8 h). On FTKC/FTKQ it responds but **diverges from
  FM** (FTKC FA 466.5 ≠ FM 4215.5; FTKQ FA 778.9 ≈ FM 785.7) — so it's energy-like but its register
  differs by model. **1.4-b5** exposes it as the "Energy (FA)" diagnostic cross-check sensor.
- Short single-byte reads `RG` (FTKC `33`, FTKQ `41`) differ but carry no obvious info.
- `RM` present on both new units (`611+` / `770+`) — not yet assigned a meaning; likely timer/mode state, not a model trait.
