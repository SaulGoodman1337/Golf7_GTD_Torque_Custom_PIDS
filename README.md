# Golf 7 GTD - Torque Custom PIDs / CAN & UDS Research

Custom Torque Pro PIDs and research notes for a Volkswagen Golf 7 GTD.

This repository started as a small collection of reverse-engineered values used to display
engine and DSG oil temperatures in an Android/Torque dashboard. It is now also intended as
a structured notebook for further VAG/MQB CAN and UDS research.

> [!IMPORTANT]
> The existing temperature PIDs were tested on the original vehicle. Most additional values
> in `research/` and `docs/` are **research candidates** collected from related VAG/MQB
> vehicles and public diagnostic datasets. Treat them as unverified until they have been
> tested on the exact ECU/TCU software version.

## Current working PIDs

The original Torque CSV is kept at:

- `Torque_customPID_Golf_7_GTD_DSG.csv`

### Engine oil temperature

| Property | Value |
|---|---|
| ECU request | `7E0` |
| ECU response | `7E8` |
| UDS request | `22 11 BE` |
| Torque Mode/PID | `0x2211BE` |
| Formula | `(((A*256)+B)-2731)/10` |
| Unit | deg C |
| Status | **Vehicle verified** |

### DSG oil temperature

| Property | Value |
|---|---|
| TCU request | `7E1` |
| TCU response | `7E9` |
| UDS request | `22 21 04` |
| Torque Mode/PID | `0x222104` |
| Formula used by this vehicle | `A` |
| Unit | deg C |
| Status | **Vehicle verified** |

The `22 2104` encoding is known to differ between some VAG transmission software versions.
Do not replace the working `A` equation only because another vehicle uses `A-40` or a
signed 16-bit value.

## Diagnostic session

The existing Torque configuration opens UDS extended diagnostic session `10 03`.

That matters because some useful VAG measuring values do not answer in the default session.
It also means this repository is not limited to generic SAE OBD-II PIDs: most of the useful
research here uses manufacturer-specific UDS `ReadDataByIdentifier (0x22)`.

## Interesting research areas

The current research backlog focuses on:

- DPF soot mass: measured and calculated
- distance/time since last DPF regeneration
- DPF regeneration state and interruption counters
- exhaust/DPF inlet, outlet and surface temperatures
- DPF ash load and differential pressure
- injector correction values per cylinder
- turbo/boost control feedback
- DSG clutch temperature and hydraulic pressure
- 12 V battery SOC, temperature, voltage and current
- cluster/ABS/climate measuring values
- reverse engineering the exact UDS sequence used for a service DPF regeneration

See:

- [Research notes](docs/RESEARCH.md)
- [DPF regeneration research](docs/DPF_REGENERATION.md)
- [CAN/UDS capture workflow](docs/CAPTURE_WORKFLOW.md)
- [Sources and attribution](docs/SOURCES.md)
- [Machine-readable DID candidate list](research/vag_mqb_did_candidates.csv)

## Repository layout

```text
.
├── README.md
├── Torque_customPID_Golf_7_GTD_DSG.csv
├── docs/
│   ├── CAPTURE_WORKFLOW.md
│   ├── DPF_REGENERATION.md
│   ├── RESEARCH.md
│   └── SOURCES.md
└── research/
    └── vag_mqb_did_candidates.csv
```

The original Torque CSV remains at the repository root to avoid breaking existing imports
or bookmarks. Experimental values are deliberately kept outside that file until they have
been validated.

## Confidence labels

Research entries use these labels:

| Label | Meaning |
|---|---|
| `vehicle-verified` | Tested on the original Golf 7 GTD |
| `platform-candidate` | Seen on a related VAG/MQB controller and worth probing |
| `session03-candidate` | Candidate expected to require `10 03` first |
| `variant-dependent` | Multiple incompatible encodings are documented |
| `raw-only` | Identifier is known, but physical scaling/unit is not trustworthy yet |
| `active-routine-unknown` | An actuator/basic-setting function exists, but its exact UDS routine is not yet captured |

## Safe research rule

Read-only requests and active diagnostic functions are intentionally separated.

Good candidates for normal Torque dashboards are UDS service `0x22` reads. Active functions
such as a forced DPF regeneration must **not** be placed in a Torque `startDiagnostic`
sequence. Torque may reconnect or reinitialize unexpectedly, which makes it a poor place for
a command that can raise engine speed and exhaust temperature.

The active regeneration path will only be documented after the exact sequence has been
captured from VCDS/ODIS/OBDeleven on the target ECU.

## Suggested next validation session

A useful next test on the Golf is to probe these read-only DPF values individually while
logging the raw responses:

```text
22 1A BE   DPF soot mass measured
22 26 09   DPF soot mass calculated
22 1A BA   distance since last regeneration
22 1A D4   regeneration status
22 11 B2   DPF inlet temperature
22 10 F9   DPF outlet temperature
22 1A C9   interrupted regenerations
```

Because the existing setup already enters `10 03`, also test the session-gated candidates:

```text
22 11 4E   soot mass measured, alternate DID
22 11 4F   soot mass calculated, alternate DID
22 10 44   DPF surface temperature
22 14 F5   DPF differential pressure, raw scaling currently uncertain
```

For every test, save both the decoded value and the complete raw ECU response. Raw captures
are more valuable than screenshots because formulas can be corrected later.

## Vehicle-specific information wanted for future work

For future reverse engineering, save the following from VCDS/ODIS/OBDeleven:

```text
Address 01 - Engine
Part No SW
Part No HW
Component
Revision
ASAM Dataset
ASAM Dataset Revision
VIN model year
Engine code
```

For DSG research, save the same information from Address 02 - Auto Trans.

## Contributions / research notes

When adding a new DID, include:

1. request header and response header
2. UDS service + DID
3. required diagnostic session
4. raw response example
5. proposed equation
6. physical unit
7. controller/vehicle/software where it was observed
8. source
9. confidence label

Do not mark a value as verified solely because it returns data. A plausible number with the
wrong byte order or scaling is still wrong.

## Disclaimer

Vehicle diagnostics can affect safety-critical systems. Read-only telemetry is the primary
scope of this repository. Active basic settings, adaptations and routines can create high
exhaust temperatures or alter controller state. Validate commands on the exact controller
and follow manufacturer service prerequisites.
