# Simulated Chromatic integration validation

> This reference describes the access-controlled Rommy source kit. See the [starting guide](../README.md) for source access. File paths and build commands refer to that source checkout. Documentation snapshot: 2026-10-12.

This kit is validated without a physical handheld. The original GB/GBC ROM runs in the repository's Gambatte emulator; the native save implementation also runs against a portable C storage fixture. These checks exercise the cartridge program and save format. They do not emulate the Chromatic FPGA, cartridge electronics, flash tool or USB save exporter.

## Reproduce the simulation

Use Node 22, a C99 compiler and the official GBDK 4.5.0 toolchain. Set `GBDK_HOME` to its extracted directory, then run from the repository root:

```sh
node --test integrations/chromatic/test/format.test.cjs
bash integrations/chromatic/build.sh
node integrations/chromatic/test/emulator.cjs
node integrations/chromatic/test/build-boundaries.cjs
```

The dedicated **Chromatic native simulation** CI workflow repeats these checks on Linux. It verifies the pinned toolchain archive's SHA256 before executing it. Core sources and binaries are the same repository Gambatte build used by the web application. No production key, learner or network API is used in the simulation.

## What the checks establish

| Layer | Checked behavior |
| --- | --- |
| Pack/export | Canonical assignment identity, text wrapping and format rejection; a different assignment cannot overwrite a build directory containing an earlier lesson. |
| Build | Original demo and the maximum 32-question pack compile to the declared 32 KiB GB/GBC ROM, with MBC5 and 8 KiB persistent save RAM. |
| Rendering/input | Nonblank 160×144 output, bounded maximum text, D-pad selection, A submission and B cancellation. Screenshots are saved from actual emulator frames. |
| Persistence | Answers survive a cold emulator restart and continue with the next sequence. The final journal slot commits; a full journal never wraps or overwrites. |
| Interrupted writes | Portable C tests interrupt initialization and record writes at byte boundaries. Corrupt, wrong-pack and torn saves remain unchanged by game input. |
| Companion import | The emulator's actual SRAM is parsed in dry-run mode, then imported twice through a synthetic HTTP response fixture. Both imports retain the same answer identities; malformed receipts fail closed. |
| Learning backend | The separate `scripts/test-offline-assignments.cjs` suite runs the real route, authorization and learning code against an isolated transaction fixture. It verifies scope/revocation, online/offline learner identity, atomic updates, repeated imports, conflicts and days-later recovery. |

The companion's synthetic receipt count is not a production learning update. The backend suite supplies independent coverage of grading and progress; neither test writes a real student's records.

## Inspect the artifacts

After running the checks in the source kit, `integrations/chromatic/build/` contains the original `rommy-quiz.gb`, its `assignment.json`, emulator screen captures, battery saves and `emulator-import-report.json`. The largest layout has its own `boundary-32/` directory. Keep each ROM, assignment and save together. The fixture flag prevents these demonstration lessons from being uploaded as real student progress.

Open the demo ROM in a compatible GB/GBC emulator, press A to begin, use Up/Down to choose, A to save and B to return before submission. This demonstration exercises the reusable quiz module. Integrate `rommy_quiz_run(question_index)` at a checkpoint in an original homebrew game to return to that game's code after the question; the module does not inject itself into an unrelated cartridge.

The private source repository’s CI attempts to retain build artifacts for seven days. Those workflow logs and artifacts require source-repository access. If the account's artifact quota is full, test outcomes and diagnostics remain in the workflow log; local reproduction still creates all files above.

The exact physical support boundary is documented in [HARDWARE.md](HARDWARE.md).
