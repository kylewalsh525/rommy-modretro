# Rommy for GB/GBC homebrew and Chromatic

> This reference describes the access-controlled Rommy source kit. See the [starting guide](../README.md) for source access. File paths and build commands refer to that source checkout. Documentation snapshot: 2026-10-12.

New to the integration? Start with the [ModRetro developer guide](../README.md) for a guided demo, API-key setup, and a copyable coding-agent prompt. This document is the detailed build and storage reference.

This is an original, native Game Boy / Game Boy Color question module, a playable demonstration ROM, and a computer-based save importer. It targets ModRetro Chromatic through its documented GB/GBC cartridge compatibility. The ROM, screen and save journal are tested in Gambatte; no physical handheld or cartridge has been qualified. See [hardware support and qualification](./HARDWARE.md) for the exact boundary, including DevDay cartridges and M64.

The native screen is 160×144. D-pad Up/Down selects an answer, A saves it, and B cancels before submission. Feedback appears only after the save has been read back successfully. The demo returns to its menu after feedback; a homebrew game can call the same function at a checkpoint and then resume its own gameplay.

## Build the demonstration

Requirements: Node.js 22+ and the official [GBDK 4.5.0 release](https://github.com/gbdk-2020/gbdk-2020/releases/tag/4.5.0). GBDK is a separate build dependency and is not installed automatically or committed here. Select the release archive for your OS/architecture and verify its release digest. The macOS Apple Silicon archive used for local validation has SHA256:

```text
289ee60e46c5a2785a21e35533f84a5131ed4a063b21b0dbdedc9a10af15bf78
```

From the repository root:

```sh
GBDK_HOME=/absolute/path/to/gbdk bash integrations/chromatic/build.sh
```

This produces `integrations/chromatic/build/rommy-quiz.gb`, generated C pack files and `assignment.json`. Build outputs are ignored. The ROM uses MBC5+RAM+BATTERY (header 0x1B), 32 KiB ROM and 8 KiB save RAM. Use a compatible cartridge with persistent SRAM/FRAM; a cartridge without persistent save storage cannot preserve progress after power loss.

`fixtures/demo-assignment.json` contains three clearly synthetic questions and no real learner assignment. Its `fixture:true` flag is retained by the exporter, and the importer refuses to upload it. It is suitable for emulator and local codec tests only.

To compile a real assignment response saved by a trusted computer application:

```sh
GBDK_HOME=/absolute/path/to/gbdk bash integrations/chromatic/build.sh assignment.json integrations/chromatic/build/lesson-one
```

Use a separate output directory per assignment, with one build/export process per directory at a time. The exporter permits rebuilding the same immutable pack but refuses to overwrite another lesson, change its fixture flag or replace a damaged/missing manifest alongside existing build files. Preserve the ROM, `assignment.json` and each original save backup together. Custom output directories outside `build/` need their own ignore/access rules; learner assignments and saves do not belong in Git.

Never put an API key, client token, email, or student name into a ROM or save. The exporter retains only assignment ID, pack hash and the question pack; authentication remains on the companion computer. Local correct-answer indexes are included for offline feedback and can be extracted from a ROM; this is a practice-learning integration, not tamper-proof assessment.

## Create a lesson and sync its save

The trusted computer calls `POST /api/public/v1/offline/assignments` with its API key, an `Idempotency-Key`, the external learner ID and an explicit list of up to 32 question IDs. Save the returned JSON envelope. Server-side creation rejects unsupported question formats and stores the immutable questions used for grading.

Pack preparation uses `offline:assignments:create` and `offline:assignments:read` key scopes; the importer requires `offline:attempts:write`. Provision these explicitly for a restricted key. The ordinary `sessions:create` scope alone does not authorize cartridge operations. An example creation body is `{"externalLearnerId":"learner-one","questionIds":["your-text-question-id"]}`. Use the same learner ID and project as the Linux bridge when their progress should be combined.

The pack contains printable ASCII text only. Prompts are at most 90 characters and five greedily wrapped 18-column lines. Two to four choices each fit 36 characters and two 18-column lines. Leading/trailing/repeated spaces, words wider than 18 columns, Unicode, and text that would overflow are rejected rather than truncated. HTML, LaTeX, diagrams and audio are not rendered by this module; the server selects only suitable text questions.

Export the cartridge's **battery save** as an 8192-byte `.sav`, not an emulator save-state. [Cart Clinic](https://support.modretro.com/en_us/chromatic-firmware-updater-ryhoYnzCx) documents backup/restore for ModRetro cartridges only; other flash cartridges need their own supported save-export tool. The exact cartridge and game-build save compatibility must be checked on actual hardware. Preserve the original backup.

Inspect without any network request:

```sh
node integrations/chromatic/bin/import-save.cjs assignment.json game.sav --dry-run
```

For a real assignment, set `ROMMY_API_BASE_URL=https://rommygames.netlify.app` to use the HTTPS REST origin without interactive browser challenges, and provide `ROMMY_API_KEY` through your local environment/secret manager, then run:

```sh
node integrations/chromatic/bin/import-save.cjs assignment.json game.sav
```

The importer sends ordered batches of at most 8 attempts to `POST /api/public/v1/offline/assignments/{assignmentId}/attempts`. This is the ordinary REST API; do not use `https://mcp.rommygames.com/mcp` as its base URL. Real uploads require the active Rommy project and offline permissions described in the starting guide. It uses no browser session and never changes the save. HTTPS is required except for loopback development; redirects are rejected so the credential cannot be forwarded to another origin. Each request times out after 20 seconds, and response bodies are limited to 64 KiB. Receipts must match assignment, pack, answer identity, response time, grading and a valid processing timestamp before a batch is reported as imported. On a network failure, unexpected response, 429 or 5xx, keep the same files and rerun later. Already committed attempts replay safely. A 409 conflict needs review: do not generate a new identity merely to bypass it.

A torn final record blocks ordinary import. `--allow-incomplete-tail` explicitly imports only the verified committed prefix; the incomplete record remains in the untouched file. The cartridge will continue to stop on that save. Back up and inspect it before any manual repair or move to a newly issued assignment. There is no reset/erase action in the ROM or importer.

## Reuse in an original game

Compile `native/save.c`, `native/gb_storage.c`, `native/quiz.c` and the generated `rommy_pack.c`/`rommy_questions.c` with your own game. Include `native/rommy_quiz.h` and call:

```c
rommy_quiz_display_init();
/* Only offer rommy_save_initialize() after explicit confirmation of an empty save. */
result = rommy_quiz_run(question_index);
/* Restore your game's graphics/audio and resume after ROMMY_QUIZ_SAVED or CANCELLED. */
```

The reference module reserves **all SRAM bank 0 (8 KiB)** and ROM bank 1 for question text. It replaces the background/font tiles and text palette, uses the standard VBlank timing, and temporarily switches ROM bank 1. It restores the prior ROM bank when returning. Your integration must allocate these resources without overlapping its own assets, pause its own timers/audio, and restore its own display afterwards. To put the journal in another SRAM bank, implement the four `rommy_storage_*` functions for your cartridge layout; retain the exact byte format and 8 KiB region. Do not point it at existing game-save data.

The journal holds 338 answers. It never wraps or silently overwrites. When full, export/import the save and issue a new assignment for further play; this minimal module does not implement an acknowledgement/compaction protocol. A new assignment has a new identity. Rebuilding a different pack while retaining the old save is detected and blocked. Restoring an older save can replay known events, but changed answers at an already-used sequence produce a server conflict rather than extra credit.

## Validation

See [SIMULATION.md](./SIMULATION.md) for the reproducible simulation report, CI coverage and artifact guide.

```sh
# JS pack/save/import tests plus a portable C journal test; requires a C99 compiler (CC or cc).
node --test integrations/chromatic/test/format.test.cjs

# Build first, then boot the ROM with this repository's existing Gambatte WASM core.
node integrations/chromatic/test/emulator.cjs

# A different build directory can be supplied explicitly.
node integrations/chromatic/test/emulator.cjs /absolute/path/to/build

# Compile and emulate the maximum32-question/90-character/4x36-character pack.
GBDK_HOME=/absolute/path/to/gbdk node integrations/chromatic/test/build-boundaries.cjs
```

The native test interrupts initial clearing/header writes, cuts storage writes at every byte boundary of a record, checks checksum failures, refuses reset of damaged data and verifies journal-full behavior. The emulator test explicitly selects both GB and GBC hardware modes and verifies their distinct system-RAM sizes. It checks a nonblank 160×144 screen, D-pad/A/B, response-time bounds, SRAM contents, cold reboot, continuing sequence numbers, the final writable record, full-journal blocking and preservation of torn, corrupted and wrong-pack saves. The resulting native save passes through the actual importer in dry-run mode and twice through an in-process HTTP mock; incomplete-tail recovery imports only its committed prefix. No external network call or production write occurs. These simulated receipts verify companion behavior; backend grading/Firestore atomicity has separate server tests.

The optional boundary build also checks that 32 maximally sized questions fit the declared 32 KiB ROM, then repeats the same emulator/import checks. Screenshots, `.sav` files and `emulator-import-report.json` are written into the ignored build directory. The playable fixture is `build/rommy-quiz.gb`; open it in a GB/GBC emulator with battery-save support. This completes the repeatable software simulation; it does not establish exact Chromatic FPGA behavior, electrical cartridge compatibility, real power-loss timing, physical legibility or save extraction on a particular handheld/cart.

ModRetro's [DevDay quickstart](https://support.modretro.com/en_us/chromatic-devday-edition-quickstart-guid-By1iOlcMg) separately documents activated developer-cartridge flashing and computer-emulator streaming to Chromatic. Its current [Developer Features agreement](https://modretro.com/pages/eula) limits those features to personal, noncommercial development/testing unless otherwise agreed in writing. This integration uses GBDK and does not automate the ModRetro updater or activate/flash hardware.

The exact interchange contract is in [PROTOCOL.md](./PROTOCOL.md).
