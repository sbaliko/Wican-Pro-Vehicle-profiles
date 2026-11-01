# Fix Hyundai Ioniq Electric 38kWh profile — 33 parameters verified on-vehicle

## Summary

Complete overhaul of the Hyundai Ioniq Electric 38kWh (2020–2021) profile. Every formula has been rewritten and validated against live data from multiple drive sessions, AC charging sessions, and parked captures across March–September 2026.

## What was wrong

The original profile had fundamental issues that produced incorrect or zero values:

- **Speed**: `B20*3.97+2.3` — wrong formula entirely, always offset by +2.3 even when stationary
- **Current**: `[B17:B18]/10` — unsigned, regen braking showed as large positive values instead of negative
- **Temperatures**: `B22-40` etc. — applied -40°C offset when raw values are already in °C
- **Tire pressures**: `/5` — gave readings in PSI, not bar (schema unit)
- **Tire temp RR**: `B24-50` — B24 is a CF3 sequence byte (0x23), not data
- **Charging flags**: `(B15&32)/32` — wrong byte, B14 holds the charging status
- **Odometer**: `[B13:B14]` — 16-bit, overflows at 65,536 km
- **MCU PID**: `2101` — missing response line count, causing timeouts
- **TPMS PID**: `22C00B` — missing response line count (needs 4 frames)
- **CLU PID**: `22B002` — missing response line count (needs 3 frames)
- **Outside temp**: missing entirely
- **Motor RPM**: missing entirely

## Changes

### BMS 0x0101 (PID `2201019`)

| Parameter | Before | After | Fix |
|-----------|--------|-------|-----|
| CHARGING | `(B15&32)/32` | `(B14&32)/32` | Wrong byte — B14 changes during charging, B15 doesn't |
| CHARGING_DC | `(B15&64)/64` | `(B14&64)/64` | Same |
| HV_A | `[B17:B18]/10` | `(S17*256+B18)/10` | Signed current for regen braking |
| HV_W | `([B19:B20]/10)*([B17:B18]/10)` | `((S17*256+B18)/10)*([B19:B20]/10)` | Uses corrected signed current |
| HV_T_MAX | `B22-40` | `B22` | Raw = °C, no offset needed |
| HV_T_MIN | `B23-40` | `B23` | Same |
| HV_T_I | `B26-40` | `B26` | Same |

### MCU 0x2101 (PID `2101` → `21014`)

| Parameter | Before | After | Fix |
|-----------|--------|-------|-----|
| *(pid)* | `2101` | `21014` | Added 4-frame response count |
| SPEED | `B20*3.97+2.3` | `(S20*256+B19)*0.01609344` | Signed LE int16 in hundredths of mph → km/h. R=1.000 vs Car Scanner, R=0.98 vs GPS. |
| ENGINE_RPM | *(missing)* | `(S20*256+B19)*0.967711` | Fixed gear ratio derived from BMS motor speed correlation |
| THROTTLE | `B13` | `B12` | Corrected byte position |

### TPMS 0x22C00B (PID `22C00B` → `22C00B4`)

| Parameter | Before | After | Fix |
|-----------|--------|-------|-----|
| *(pid)* | `22C00B` | `22C00B4` | Added 4-frame response count |
| TYRE_P_* | `/5` | `/72.5` | Raw unit is 0.2 PSI; /72.5 converts to bar |
| TYRE_T_RR | `B24-50` | `B25-50` | B24 is CF3 sequence byte (0x23), data is at B25 |

### HVAC 0x0100 (PID `2201006`)

| Parameter | Before | After |
|-----------|--------|-------|
| TMP_A | *(missing)* | `B12/2-40` |

### CLU 0xB002 (PID `22B002` → `22B0023`)

| Parameter | Before | After | Fix |
|-----------|--------|-------|-----|
| *(pid)* | `22B002` | `22B0023` | Added 3-frame response count |
| ODOMETER | `[B13:B14]` | `B12*65536+[B13:B14]` | 24-bit value — 16-bit overflows at 65,536 km |

## Validation

Tested on a 2021 Hyundai Ioniq Electric 38kWh (AE, European market) using WiCAN Pro (firmware v4.50) across multiple sessions from March–September 2026.

**Conditions tested:**
- Driving (multiple sessions, 0–110 km/h, city and highway)
- Parked in ACC and READY states
- AC charging (7.2 kW on-board charger)
- Post-BMS-update (dealer recalibration June 2026)
- Odometer past 65,536 km overflow

**Verification methods:**
- GPS speed correlation: R = 0.98 over 502 driving samples
- Car Scanner Pro cross-reference: R = 1.000 over 922 synchronised speed samples
- OBC DC voltage/current vs BMS cross-validation during AC charging
- Raw ELM327 frame capture and byte-level verification for every parameter

All 33 parameters produce correct values across all vehicle states.

## Vehicle details

- **Model:** Hyundai Ioniq Electric (AE) 38.3 kWh, 2020–2021 facelift
- **ECUs:** BMS (7E4), MCU (7E2), HVAC (7B3), TPMS (7A0), CLU (7C6)
- **Adapter:** WiCAN Pro, firmware v4.50, ELM327 mode
