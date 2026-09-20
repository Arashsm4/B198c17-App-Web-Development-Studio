# Cinerion

> **CONFIDENTIAL — PROJECT INTERNAL(well it was supposed to be I had to make it public I'll remove it later)**  
> Copyright © 2026 IR. All rights reserved.

Cinerion is a secure operations, manufacturing, quality and inspection platform for an
automated production cell. This repository implements the `CH1` namespace against the frozen
**P03 / IFV** engineering baseline in `docs/baseline/P03_IFV/`.

**This build is Prototype 2, and it is a simulation.** Everything runs, but no real machine is
attached, and nothing in it can move one: there is no route from the portal to an actuator,
the firmware drives no output, and the pin map declares every actuator non-energisable.

## Install

| On | Do |
|---|---|
| **Windows** | Unzip, then double-click **`install.cmd`** |
| **Linux / macOS / WSL** | Unzip, then `sh install.sh` |

Needs Docker, Git and Node.js 22.13+ (the Windows installer offers to install whatever is
missing). The first install takes 10–20 minutes and needs the internet once; after that
everything runs offline. Then open <http://localhost:8081> and sign in as `ch1-portal-demo`.

If the package was made with a data snapshot (`BUILD_INFO.txt` says so), the new lab opens
with the same records as the machine it was packed on: telemetry history, alarms, audit
trail and everything else. Passwords and keys are never packed; the new PC makes its own.

**[INSTALL.md](INSTALL.md)** has the full guide: requirements, sign-in, day-to-day commands,
moving to another PC, offline use, troubleshooting and uninstalling.

## What's in the lab

| Part | What it does | Where |
|---|---|---|
| Operator portal | 22 screens: operations, control room, commands, procedures, work orders, genealogy, quality, calibration, devices, security, audit, resources, research and more | `apps/operator-portal` |
| Control Core | The API and every domain rule: commands, quality loop, audit chain, identity, devices | `services/control-core` |
| EdgeLink | The gateway to the cell: MQTT, OPC UA and Modbus adapters with a 24-hour buffer | `services/edge-link` |
| Identity provider | Keycloak with the controlled realm and 15 roles | `infrastructure/keycloak` |
| Database | PostgreSQL with row-level security and 15 controlled migrations | `services/control-core/.../Migrations` |
| Simulated cell | A stand-in controller that confirms prepared commands with a marked simulated button press | `simulators/cell-confirmer` |
| Firmware and PLC | ESP32-S3 controller, Arduino I/O node and Siemens S7-1200 logic, host-tested | `firmware`, `plc` |

## How the safety chain works

A person can only ever **prepare** an action. It happens only if, at the cell itself:

1. the controller checks scope, sequence, state, revision and expiry;
2. a cryptographic local credential proves someone is present, for that one action;
3. a *new* press of the physical confirmation button is seen;
4. the local permissives are sampled again;
5. only then does the controller issue `LOCAL COMMIT`.

Measurements, decisions, faults, nonconformities and releases are chained to the same part
and work order, in a hash-linked audit log that shows any tampering. The confirmation button
is a readiness gate, **not** an emergency stop.

## What works today

- All **22 screens** on live data, laid out and checked in a real browser at three widths.
- **52 of 54** API operations. The other 2 are the secure document vault, which stays switched
  off on purpose until a separate encryption-key boundary exists.
- The prepare → confirm → commit sequence, end to end, against the simulated cell.
- The quality loop: test run, failed measurement, hold, NCR, disposition, retest, release.
- Device certificates, mutual TLS between services, and a tamper-evident audit chain.
- All three industrial protocols against live stand-in controllers, with store-and-forward.
- **No AI service anywhere**, and it runs **fully offline** once installed. Both are checked on
  every build, not just stated.

## What remains

Nothing below is blocked on code in this repository; each needs a decision, a controlled
document, or hardware.

| Waiting on | What |
|---|---|
| **An identity decision** | Step-up actions (reserving, releasing, preparing a command, registering a credential, changing roles) need a multi-factor session, and a password sign-in is single-factor in this build: the realm neither asks for a second factor nor emits the `amr` claim Control Core reads. Adding Keycloak's OTP at sign-in and its AMR mapper would open the whole chain. It's a sign-in design change, so it waits on the project owner |
| Owner decisions | Recorded conflicts 34–37, and whether the realm file should declare `ProjectOwner` for the demo identity |
| Controlled documents | Conflicts 16, 17, 24, 25, 32, 33 and five obligations; the baseline is frozen and checksum-verified |
| Hardware | 4 cases need the real S7-1200 G2, 5 need a hardware-in-the-loop rig; no actuator may be energised before wiring freeze and independent review |

The full list, with the reasoning behind each, is in
[`docs/implementation/PROTOTYPE2_PROGRESS.md`](docs/implementation/PROTOTYPE2_PROGRESS.md).

## Verification

```bash
npm ci
./scripts/verify.sh --strict      # about 50 minutes; nothing may be skipped
```

The strict pipeline runs every gate the project has: the frozen baseline by SHA-256, 513 .NET
tests against a real PostgreSQL, firmware and PLC host builds, 61 portal render scenarios,
183 real-browser layouts, protocol edges, offline operation, the 24-hour buffer, the baseline
load, and 141 attacks across 9 suites that prove the gates actually refuse what they claim
to. Details of each gate: [`docs/implementation/DEVELOPMENT_GATES.md`](docs/implementation/DEVELOPMENT_GATES.md).

## Repository map

| Path | What lives there |
|---|---|
| `apps/operator-portal` | The portal (Expo / React Native for web) and its render harness |
| `services/control-core` | .NET API, domain, application and persistence layers, with tests |
| `services/edge-link` | The protocol gateway and its buffer, with tests |
| `simulators` | Stand-in controllers (OPC UA, Modbus, cell confirmer) and the state model |
| `firmware`, `plc` | Controller firmware, I/O node, PLC logic and their host tests |
| `infrastructure` | Compose models, Keycloak realm, nginx and broker configuration |
| `packages/contracts` | Shared contracts and the generated OpenAPI |
| `traceability` | Requirement-by-requirement implementation state and the verification map |
| `scripts` | Every gate, attack suite and lab script |
| `docs/baseline/P03_IFV` | The frozen controlled engineering baseline |
| `docs/implementation` | Engineering record, gates, architecture and runbooks |

## Documentation

- [INSTALL.md](INSTALL.md) — installing and running the lab
- [CHANGELOG.md](CHANGELOG.md) — what changed, newest first
- [docs/implementation/PROTOTYPE2_PROGRESS.md](docs/implementation/PROTOTYPE2_PROGRESS.md) — the full engineering record; read "Resuming work in a new session" before changing anything
- [docs/implementation/DEVELOPMENT_GATES.md](docs/implementation/DEVELOPMENT_GATES.md) — every gate and what it refuses
- `BUILD_INFO.txt` and `MANIFEST.sha256` (in a release archive) — which commit this is, and a checksum for every file

## Identifiers and identity

`Cinerion` is the project name and `CH1` the permanent identifier namespace; requirement IDs
appear directly in source and test names. Real sign-in is OIDC through Keycloak. A development
identity exists for tests only, and is refused unless the host is in Development **and**
`CINERION_ALLOW_DEVELOPMENT_IDENTITY=true`; the lab runs with it off.
