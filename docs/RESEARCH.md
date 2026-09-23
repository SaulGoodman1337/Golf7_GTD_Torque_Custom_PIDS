# VAG/MQB Research Notes

This document collects the current research state for the Golf 7 GTD project.

The goal is not to build a giant list of copied PIDs. The goal is to preserve enough context
that a future test session can efficiently decide:

- which controller to address
- which UDS DID to request
- whether `10 03` is required
- how to decode the payload
- how trustworthy the result is

## 1. Protocol model used by this repository

The existing implementation uses ISO 15765-4 CAN with UDS.

Typical module pairs relevant to this project:

| Module | Request | Response | Notes |
|---|---:|---:|---|
| Engine ECU | `7E0` | `7E8` | EA288/EDC, main DPF and engine telemetry |
| DSG / automatic transmission | `7E1` | `7E9` | transmission telemetry |
| Battery monitor / IBS | `710` | `77A` | battery state and current |
| ABS/ESC | `713` | `77D` | wheel/brake/indirect TPMS data |
| Instrument cluster | `714` | `77E` | cluster oil temp, odometer and related data |
| Climate | `746` | `7B0` | HVAC measurements |

The main service used for telemetry is:

```text
22 xx xx
```

which is UDS `ReadDataByIdentifier`.

Positive response:

```text
62 xx xx <payload...>
```

In formulas below, `A` is the first payload byte after the echoed DID, `B` the second,
and so on.

## 2. Existing vehicle-verified values

### Engine oil temperature

```text
Header:    7E0
Response:  7E8
Request:   22 11 BE
Formula:   (((A*256)+B)-2731)/10
Unit:      deg C
```

### DSG oil temperature

```text
Header:    7E1
Response:  7E9
Request:   22 21 04
Formula:   A
Unit:      deg C
```

The DSG formula is explicitly retained as vehicle-specific evidence. Public VAG datasets
also show `A-40` and signed 16-bit variants for `2104`.

## 3. DPF candidates

These are the highest-priority values for the next test session.

### 3.1 No session change explicitly required in current sources

| DID | Meaning | Decode | Unit | Confidence |
|---|---|---|---|---|
| `1ABE` | soot mass measured | signed16 / 100 | g | platform-candidate |
| `2609` | soot mass calculated | signed16 / 100 | g | platform-candidate |
| `1ABA` | distance since last regen | uint16 / 10 | km | platform-candidate |
| `1AD4` | regeneration status | uint16 | enum/raw | raw-only |
| `1AC3` | time since last regen | uint16 * 127.998 / 65535 | h | platform-candidate |
| `1AC0` | regeneration time counter | uint16 * 0.64 | s | platform-candidate |
| `1AC4` | service regeneration current duration | uint16 * 0.64 | s | platform-candidate |
| `1AC1` | fuel used since last regen | uint16 / 100 | l | platform-candidate |
| `1AC8` | regeneration blocked status | A | enum/raw | raw-only |
| `1AC9` | interrupted regenerations | A | count | platform-candidate |
| `11B2` | DPF inlet temperature | uint16 / 10 - 273.1 | deg C | platform-candidate |
| `10F9` | DPF outlet temperature | uint16 / 10 - 273.1 | deg C | platform-candidate |
| `1ABD` | DPF oil ash mass | uint32 * 0.0011921 / 10000 | g | platform-candidate |
| `1ABF` | DPF ash load limit | uint16 * 78.125 / 10000 | g | platform-candidate |
| `115C` | service regeneration status | A | enum/raw | raw-only |

### 3.2 Candidates reported with extended diagnostic session `10 03`

| DID | Meaning | Decode | Unit | Confidence |
|---|---|---|---|---|
| `114E` | soot mass measured, alternate | signed16 / 100 | g | session03-candidate |
| `114F` | soot mass calculated, alternate | uint16 / 100 | g | session03-candidate |
| `1044` | DPF surface temperature | uint16 / 10 - 273.1 | deg C | session03-candidate |
| `14F5` | differential pressure | signed16 | unknown/raw | session03-candidate, raw-only |
| `1156` | distance since last regeneration | uint32 / 1000 | km | session03-candidate |

The T6 community also reports `114E`, `114F`, `1156`, `11B2` and `10F9`, but VAG
software generations can remap DIDs. The presence of a DID on a T6 does not prove that the
Golf GTD uses the same DID.

## 4. DPF interpretation strategy

A useful dashboard should show more than a single "DPF %" estimate.

Recommended group:

```text
soot mass calculated
soot mass measured
distance since last regeneration
regeneration status
DPF inlet temperature
DPF outlet temperature
interrupted regeneration counter
```

Why both soot values matter:

- calculated soot is the ECU model
- measured soot is influenced by differential-pressure behaviour
- a persistent disagreement can be diagnostically useful
- both should be logged across multiple regeneration cycles before drawing conclusions

Do not hard-code a generic "100 % = 24 g" threshold for the Golf unless the exact ECU's
threshold has been confirmed. Community examples use different limits on different engines.

## 5. Injector correction candidates

Engine ECU `7E0 -> 7E8`:

| Cylinder | DID | Decode | Unit |
|---|---|---|---|
| 1 | `10FF` | signed16 / 100 | mg/str |
| 2 | `1105` | signed16 / 100 | mg/str |
| 3 | `1100` | signed16 / 100 | mg/str |
| 4 | `1104` | signed16 / 100 | mg/str |

Important: the DID numbers are not in cylinder-number order.

These values are useful as trends, especially at warm idle. They should not be used as a
standalone injector diagnosis without considering rail pressure, smooth running, compression,
fuel system condition and ECU strategy.

## 6. Boost / turbo candidates

Engine ECU:

| DID | Meaning | Decode | Unit |
|---|---|---|---|
| `1057` | boost pressure absolute | uint16 / 1000 | bar |
| `11CC` | boost regulator feedback | signed16 * 0.01 | % |
| `112C` | turbine actuator activation | signed16 * 0.01 | % |
| `1139` | mean injection quantity | signed16 / 100 | mg/str |

A useful log combines RPM, accelerator position, boost actual, boost command if available,
and actuator feedback.

## 7. DSG candidates

TCU `7E1 -> 7E9`:

| DID | Meaning | Decode | Unit | Notes |
|---|---|---|---|---|
| `2104` | transmission fluid temperature | vehicle uses `A` | deg C | variant-dependent |
| `18E3` | calculated clutch temperature | signed16 | deg C | platform-candidate |
| `381D` | hydraulic oil pressure actual | uint16 / 100 | bar | platform-candidate |
| `381E` | pressure control current | uint16 / 10 | mA | platform-candidate |
| `38B2` | centrifugal oil temperature | signed8 | deg C | platform-candidate |
| `210F` | current gear | A | enum | platform-candidate |

Do not assume the public `2104` equation applies to this vehicle; the repository's original
`A` equation already has stronger vehicle-specific evidence.

## 8. 12 V battery monitor candidates

Battery monitor `710 -> 77A`:

| DID | Meaning | Decode | Unit | Notes |
|---|---|---|---|---|
| `2A07` | battery voltage | uint16 / 1000 + 4 | V | platform-candidate |
| `2A0B` | battery temperature | A - 40 | deg C | platform-candidate |
| `2A0C` | battery state of charge | A | % | platform-candidate |
| `2A0E` | internal resistance | variant-dependent | mOhm | probe by plausibility |
| `2A0F` | usable battery charge | A | Ah | platform-candidate |
| `2A10` | battery voltage at rest | variant-dependent | V | platform-candidate |
| `2A09` | battery current | variant-dependent | A | multiple encodings documented |

For `2A09`, do not pick an equation solely because it produces a number. Validate sign and
magnitude during engine-off load, cranking and charging.

## 9. Cluster candidates

Instrument cluster `714 -> 77E`:

| DID | Meaning | Decode | Unit |
|---|---|---|---|
| `202F` | cluster/modelled oil temperature | A - 58 | deg C |
| `2203` | odometer | uint24 | km |
| `224B` | oil thermal wear | uint16 | km |
| `2299` | average fuel consumption | uint16 / 10 | l/100 km |

The cluster oil value can be modelled rather than a direct physical sensor value depending on
the vehicle.

## 10. Validation rules

A candidate becomes `vehicle-verified` only after all of the following are recorded:

- controller part number/software
- DID and diagnostic session
- at least one complete raw positive response
- decoded value
- independent plausibility check
- behaviour over time, not just one snapshot

Useful plausibility checks:

- temperatures converge near ambient after a long cold soak
- battery current changes sign/magnitude predictably under load/charging
- distance-since-regen resets after an observed regeneration
- soot mass drops during a confirmed regeneration
- DPF temperatures rise substantially while regeneration is active
- injector corrections are observed at stable warm idle

## 11. Negative responses worth recording

Do not discard failed requests. UDS negative responses contain useful information.

Common form:

```text
7F <service> <NRC>
```

Examples of useful distinctions:

- service not supported
- sub-function not supported
- request out of range
- conditions not correct
- security access denied
- response pending

Record the full response and current diagnostic session.

## 12. Open questions

Current unresolved items:

1. Exact engine ECU / ASAM dataset of the original Golf 7 GTD.
2. Which of the MQB DPF DIDs are accepted by that exact software.
3. Exact enum/bit meaning for `1AD4`, `1AC8`, `115C`.
4. Physical scaling for differential pressure `14F5` on the target ECU.
5. Whether `114E/114F` and `1ABE/2609` coexist or belong to different software families.
6. Exact UDS basic-setting sequence for service DPF regeneration.
7. Whether a security-access seed/key exchange is required for the target ECU.
8. Exact abort/stop routine for a manually started regeneration.
