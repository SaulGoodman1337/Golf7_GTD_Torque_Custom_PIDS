# DPF Regeneration Research

This document separates two very different topics:

1. observing a normal DPF regeneration
2. actively requesting a service/forced regeneration

The first is suitable for Torque. The second is an active diagnostic procedure and must be
reverse-engineered and implemented separately.

## 1. Read-only regeneration monitoring

The preferred way to understand the car's normal behaviour is to log:

```text
22 1A BE   soot mass measured
22 26 09   soot mass calculated
22 1A BA   distance since last regeneration
22 1A D4   regeneration status
22 1A C3   time since last regeneration
22 1A C9   interrupted regenerations
22 11 B2   DPF inlet temperature
22 10 F9   DPF outlet temperature
```

Optional/session-gated candidates:

```text
10 03
22 11 4E   soot mass measured, alternate DID
22 11 4F   soot mass calculated, alternate DID
22 10 44   DPF surface temperature
22 14 F5   DPF differential pressure
22 11 56   distance since last regeneration, alternate DID
```

The exact accepted set depends on the ECU software.

## 2. What a normal regeneration should reveal in logs

Without relying on a fixed threshold, a confirmed regeneration should show a combination of:

- regeneration state changing
- DPF/exhaust temperature increasing markedly
- calculated soot mass decreasing
- measured soot mass changing
- distance/time-since-regen resetting after completion
- regeneration duration/status counters changing

This is a stronger confirmation than inferring regeneration only from idle speed or radiator
fan behaviour.

## 3. Active service regeneration

Public VAG diagnostic tools expose a basic setting commonly named similar to:

```text
Service regeneration of particle filter
```

However, the human-readable VAG/ODX label is **not** the UDS RoutineControl identifier.

For example, a label such as `IDE00471` must not be interpreted as meaning that the ECU
expects:

```text
31 01 04 71
```

That would be an unsupported assumption.

## 4. Expected UDS building blocks

A captured service procedure may contain some of these services:

| UDS service | Purpose |
|---|---|
| `10 xx` | DiagnosticSessionControl |
| `27 xx` | SecurityAccess |
| `22 xxxx` | ReadDataByIdentifier |
| `31 xx xxxx ...` | RoutineControl |
| `2E xxxx ...` | WriteDataByIdentifier, if the implementation uses writable state |
| `3E 00` | TesterPresent |
| `7F .. ..` | Negative response |

Do not assume every ECU uses all of them.

## 5. General service prerequisites

Ross-Tech documents general UDS CR-TDI prerequisites for a standing service regeneration,
including:

- engine running
- sufficient fuel
- transmission in Neutral/Park
- parking brake applied
- coolant temperature above the required threshold
- hood closed
- following the on-screen pedal/brake instructions where applicable

These are general service-tool prerequisites, not a substitute for the target ECU's own
condition checks. The ECU may reject a routine with `conditionsNotCorrect` if any required
condition is missing.

A service regeneration can produce very high exhaust temperatures. Do not use it merely to
avoid allowing the vehicle's normal regeneration strategy to operate.

## 6. Why the active routine must not live in Torque

The current Torque CSV has a `startDiagnostic` sequence. This is appropriate for session
initialisation, but not for a service regeneration.

Reasons:

- Torque may reconnect to the adapter
- Torque may reinitialize a PID
- the dashboard may be opened accidentally
- multiple sensors can initialize independently
- a failed Bluetooth link can leave uncertain routine state
- an active routine needs explicit start, status and abort handling

The eventual regeneration controller should be a separate tool with a state machine.

## 7. Proposed active-regeneration state machine

No actual routine ID is specified yet because it has not been captured.

Proposed implementation shape:

```text
DISCONNECTED
    |
    v
CONNECT
    |
    v
READ ECU ID / SOFTWARE
    |
    v
CHECK PRECONDITIONS
    |
    v
ENTER REQUIRED SESSION
    |
    v
SECURITY ACCESS (only if captured/required)
    |
    v
ARMED -- explicit user confirmation --> START ROUTINE
    |                                      |
    |                                      v
    |                                  RUNNING
    |                                      |
    |                         +------------+------------+
    |                         |                         |
    |                         v                         v
    |                     COMPLETE                  ABORT/FAIL
    |                         |                         |
    +-------------------------+-------------------------+
                              |
                              v
                     RETURN DEFAULT SESSION
```

The implementation should continuously monitor ECU responses and use the ECU's own reported
status rather than a fixed timer.

## 8. Capture needed before implementation

A single successful VCDS/ODIS/OBDeleven session can answer most open questions.

Capture from before pressing Start until after completion/abort:

- tester -> engine ECU frames
- engine ECU -> tester frames
- ISO-TP multi-frame traffic
- exact timing/order
- DiagnosticSessionControl request/response
- SecurityAccess request/response if present
- RoutineControl request and complete option record
- status/result requests
- TesterPresent traffic
- stop/abort request
- all negative responses

Also record the ECU identification:

```text
Part No SW
Part No HW
Component
Revision
ASAM Dataset
ASAM Dataset Revision
```

## 9. Safety gate for future code

Before any future implementation sends an active regeneration request, it should at minimum
verify that:

- the connected ECU identity matches a known/captured configuration
- the service routine and parameters match that configuration
- the engine is already running as required
- basic preconditions reported by the ECU are satisfied
- the user explicitly arms and starts the procedure
- the program knows the captured stop/abort sequence
- status monitoring is working before start

If ECU identity is unknown, the tool should fall back to read-only monitoring.
