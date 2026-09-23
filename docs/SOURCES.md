# Sources and Attribution

This repository combines original vehicle reverse engineering with public community and
open-data research.

## Original project data

The following values originated from the original Golf 7 GTD testing in this repository:

- `22 11BE` engine oil temperature
- `22 2104` DSG oil temperature
- the associated Torque/ELM327 setup sequences

These are treated as the strongest evidence for the original vehicle.

## OBDb

OBDb is a community database for OBD parameters, scalings and diagnostic data.

- https://github.com/OBDb
- https://github.com/OBDb/Volkswagen-Golf
- https://github.com/OBDb/Audi-Q3

OBDb states that most text/data is licensed under CC BY-SA 4.0. When incorporating or
redistributing substantial OBDb-derived material, preserve attribution and check the source
repository's license.

The Audi Q3 dataset is especially useful because it contains extensive VAG diesel/MQB DPF
DIDs. The Volkswagen Golf dataset is useful for module addressing and Golf-specific signals.

## obd2-dashboard research compilation

A useful cross-reference and research compilation:

- https://github.com/miskibin/obd2-dashboard
- https://github.com/miskibin/obd2-dashboard/blob/main/docs/research-multibrand-extended-pids.md
- https://github.com/miskibin/obd2-dashboard/blob/main/app/src/main/java/com/miskibin/obd2dashboard/obd/VagPids.kt

It is valuable because it records formulas, module mappings, session requirements and
confidence caveats in one place. Values should still be traced back to primary captures where
possible.

## VW T6 community PID research

Community discussion with real CAN examples and Torque custom PID formulas:

- https://www.t6forum.com/threads/vw-t6-custom-pid-codes-for-dpf.33964/
- https://www.t6forum.com/threads/t6_measured-monitoring-dpf-regeneration-dpf-condition-egr-operation.39401/

This is useful corroboration for VAG diesel DPF concepts, but a T6 DID is not automatically a
Golf 7 GTD DID. Software families differ.

## Ross-Tech

General VAG/VCDS service procedure reference:

- https://wiki.ross-tech.com/wiki/index.php/Diesel_Particle_Filter_Emergency_Regeneration
- https://wiki.ross-tech.com/wiki/index.php/2.0L_CR_TDI

Ross-Tech is used here to document the existence and general prerequisites of service DPF
regeneration procedures. It does not provide the raw target-specific UDS RoutineControl
sequence needed by this project's future custom implementation.

## Source priority

When sources disagree, use this order:

1. raw capture from the target Golf 7 GTD
2. target ECU's ODX/ASAM/service-tool behaviour
3. raw capture from the same ECU/software family
4. OBDb vehicle dataset
5. corroborated VAG community research
6. single unsourced PID lists

A value returning a plausible number is not sufficient evidence on its own.
