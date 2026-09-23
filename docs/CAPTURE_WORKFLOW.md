# CAN / UDS Capture Workflow

This is the preferred workflow for validating new PIDs and reverse engineering diagnostic
procedures on the target Golf 7 GTD.

## 1. Record controller identity first

Before collecting data, save identification from the relevant controller.

For the engine:

```text
Address 01 - Engine
Part No SW
Part No HW
Component
Revision
ASAM Dataset
ASAM Dataset Revision
Engine code
VIN model year
```

For DSG:

```text
Address 02 - Auto Trans
Part No SW
Part No HW
Component
Revision
ASAM Dataset
ASAM Dataset Revision
```

This prevents later captures from becoming anonymous data that cannot be tied to a software
version.

## 2. Read-only DID validation

For every candidate DID:

1. send only one candidate at a time
2. record diagnostic session
3. save the entire request
4. save the entire raw response
5. decode it using the proposed formula
6. record an independent reference value if available
7. repeat under at least two different conditions

Example log entry:

```text
Date:
Vehicle:
ECU SW:
Session:
Request header:
Response header:
Request:
Raw response:
Proposed formula:
Decoded value:
Reference value:
Conditions:
Result:
```

## 3. Cold-start validation

Cold soak is especially useful for temperature signals.

After the vehicle has been parked long enough for temperatures to equalize:

- engine oil temperature
- coolant temperature
- DSG temperature
- battery temperature
- ambient temperature

should be physically plausible relative to each other. This is an easy way to expose an
offset/scaling mistake.

## 4. DPF validation drive

For DPF candidates, create a synchronized log of:

- timestamp
- RPM
- vehicle speed
- coolant temperature
- soot calculated
- soot measured
- distance since regeneration
- regeneration status
- inlet temperature
- outlet temperature
- any available differential pressure

Keep logging through a complete naturally occurring regeneration when possible.

The strongest DID confirmation is seeing internally consistent state transitions across a
whole cycle.

## 5. UDS session testing

If a DID does not respond in the default session:

1. record the default-session negative response
2. enter `10 03`
3. verify the positive session response
4. retry the same DID
5. record the result
6. return to the default session / disconnect cleanly

Do not mix session changes with active RoutineControl experiments during basic PID discovery.

## 6. Capturing a service-tool procedure

For active basic settings, use a known diagnostic tool as the reference implementation.

Suitable sources include:

- VCDS
- ODIS
- OBDeleven

Capture the raw CAN traffic while carrying out the procedure normally.

For a DPF service regeneration, start logging before entering Basic Settings and keep the
capture running through either successful completion or an intentional normal abort.

## 7. Filtering the capture

Primary engine diagnostic pair:

```text
7E0 -> request
7E8 -> response
```

Do not assume every frame of interest uses only these two IDs; gateways and extended
addressing can exist. Start with the full capture and narrow it after the procedure is
understood.

## 8. ISO-TP reconstruction

UDS messages longer than one CAN frame use ISO-TP.

The useful object for this repository is the reconstructed diagnostic message, not only the
individual CAN frames.

Save both when possible:

```text
Raw CAN frames:
7E0 ...
7E8 ...
...

Reconstructed UDS:
10 03
50 03 ...
...
```

This makes later comparison much easier.

## 9. Services to flag automatically

When analysing a diagnostic capture, highlight these service bytes:

```text
10  DiagnosticSessionControl
11  ECUReset
22  ReadDataByIdentifier
27  SecurityAccess
2E  WriteDataByIdentifier
31  RoutineControl
3E  TesterPresent
7F  NegativeResponse
```

For active-function research, `27`, `2E` and `31` deserve particular scrutiny.

## 10. Preserve failures

A negative response is useful evidence.

Store:

- request
- response
- NRC
- current session
- engine state
- security state
- relevant preconditions

A DID that returns `requestOutOfRange` in default session but succeeds after `10 03` is a
valuable finding.

## 11. Naming captures

Suggested convention:

```text
captures/
  YYYY-MM-DD_ecuSW_topic/
    notes.md
    raw.log
    decoded.txt
```

Example:

```text
captures/
  2026-09-23_04L906xxx_dpf-natural-regen/
```

Avoid committing VINs or other unnecessary personal identifiers to a public repository.

## 12. Definition of done for a new Torque PID

Before adding a research candidate to the main Torque CSV:

- DID responds on the target vehicle
- correct payload bytes are identified
- equation is confirmed
- unit is confirmed
- value is plausible in multiple states
- header/response filtering is confirmed
- required diagnostic session is known
- no active side effects are observed
- a raw-response example is documented
