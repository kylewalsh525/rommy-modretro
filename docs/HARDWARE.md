# Hardware support and qualification

> This reference describes the access-controlled Rommy source kit. See the [starting guide](../README.md) for source access. File paths and build commands refer to that source checkout. Documentation snapshot: 2026-10-12.

Reviewed against primary ModRetro documentation on 2026-10-12. **Software simulation: passed. Physical qualification: not performed.** No handheld or cartridge is attached to the development environment. Hardware is not required to repeat the emulator validation below.

| Target | Documented capability | Rommy status |
| --- | --- | --- |
| Chromatic GB/GBC cartridge mode | ModRetro documents GB/GBC cartridge compatibility and a 160×144 display. | Native GB/GBC ROM and 8 KiB save format implemented; compiled and emulated with Gambatte. Exact cartridge/firmware combination still needs qualification. |
| Chromatic DevDay Edition | An activated computer can flash the included developer cartridge. The quickstart also describes streaming a computer emulator to the handheld. | Our original ROM can be supplied to a supported flashing workflow once mapper/save compatibility is confirmed. This package does not activate, flash or stream hardware. Streaming does not validate cartridge persistence. |
| Other GB/GBC flash cartridges | Cartridge-specific tooling and save behavior vary. ModRetro's Cart Clinic save manager explicitly excludes non-ModRetro cartridges. | Requires MBC5 compatibility and at least 8 KiB persistent SRAM/FRAM, plus a supported export of the cartridge's battery save. No individual flash cartridge is qualified here. |
| ModRetro M64 | A separate system for Nintendo64 cartridges with HDMI output and firmware updates. | Unsupported by this GB/GBC module. A native N64 build/storage adapter and hardware qualification would be separate work. Its update connectivity does not establish a browser or Rommy API runtime. |

Sources: [Chromatic specifications](https://modretro.com/products/chromatic-tetris-bundle), [DevDay quickstart](https://support.modretro.com/en_us/chromatic-devday-edition-quickstart-guid-By1iOlcMg), [Cart Clinic/save manager](https://support.modretro.com/en_us/chromatic-firmware-updater-ryhoYnzCx), [M64 specifications](https://modretro.com/products/m64).

The native path runs entirely offline: questions appear within the original game, answers enter the cartridge save, and a trusted computer later imports that save through the existing REST API. USB video output and developer streaming are distinct host functions; neither gives this ROM Wi-Fi, browser, HTTP or MCP access. This module integrates at source-level checkpoints in an original GBDK game. It is not a patcher for commercial games or a drop-in GB Studio event plugin.

ModRetro's current [Developer Features agreement](https://modretro.com/pages/eula) limits those features to personal, noncommercial development/testing unless otherwise agreed in writing. Confirm an appropriate development/distribution path before relying on its activated updater for a commercial rollout. This restriction is on those developer features; it does not establish a restriction on the separately built GBDK source itself.

## First-device qualification

This is a future checklist for anyone choosing to run the ROM on a cartridge; it is not a prerequisite for the delivered emulator demo.

Record the exact console model/firmware, cartridge mapper and persistent-memory type, flashing tool/version, save-export tool/version, ROM SHA256 and assignment pack hash with the result. Use the local fixture initially; it cannot upload student progress.

1. Confirm that the chosen cartridge accepts a 32 KiB MBC5+RAM+BATTERY ROM (header type `0x1B`) and exposes 8192 battery-save bytes. Back up existing cartridge data before writing the original demo ROM with the cartridge's supported tool.
2. Boot the demo from the cartridge, initialize its genuinely empty save, and verify prompt/choice readability, D-pad selection, A submission and B cancellation. Check the maximum-text boundary build as well as the small demo.
3. After the **Answer saved** screen appears, power off normally, disconnect USB/power, restart and check the saved-answer count. Export the battery save and run the importer with `--dry-run`; the assignment, question and selected choice must match. Emulator save-states and padded dumps are not accepted as 8192-byte saves.
4. With a disposable fixture and preserved backups, test power interruption during a write. A complete answer must survive; a torn save must block further writes and preserve its confirmed prefix. Timing-dependent real power-loss behavior cannot be established by software simulation alone.
5. Once the offline backend is deployed, use an isolated real assignment and restricted API key to import twice. Verify one learning effect per sequence, then replay a backup with a changed answer at the same sequence and verify conflict. Never use a new identity to hide an unexplained conflict.
6. At journal capacity, verify that all 338 answers export and the next answer is blocked. Keep the successful import receipt, ROM, manifest and original save. Issue a new assignment into a separate build directory before preparing an empty save for that new lesson with the cartridge's supported tool. Erasing the old save is a deliberate hardware operation; this ROM and importer never do it automatically.

For an interrupted final record, normal import stops. `--allow-incomplete-tail` imports only the verified prefix; it neither reconstructs the interrupted answer nor repairs the cartridge. Preserve the original bytes for investigation. A bad checksum, hidden records, wrong pack or sequence conflict needs reconciliation before any lesson replacement.

## Reproducible simulation evidence

The commands in [README.md](./BUILD.md#validation) use official GBDK 4.5.0 and the repository's Gambatte WASM core. The same 32 KiB ROM is run with explicit `gambatte_gb_hwmode` selections for GB and GBC; the harness verifies the core reports 8 KiB and 32 KiB system RAM respectively. This tests the two emulated Nintendo hardware modes, not a Chromatic firmware/FPGA image.

| Scenario | Evidence written to `build/` (and `build/boundary-32/` for the maximum pack) |
| --- | --- |
| Boot, readable prompt and D-pad/A selection | `emulator-menu.png`, `emulator-question.png`, `emulator-feedback.png` |
| SRAM and cold boot, B cancellation, second sequence | `emulator.sav`, `emulator-resumed.sav`, `emulator-resumed.png` |
| Final writable answer, then refusal to overwrite full journal | `emulator-last-answer.png`, `emulator-full.png` |
| Torn record, bad checksum and different assignment cannot be overwritten | `emulator-save-review.png`, `emulator-checksum.png`, `emulator-wrong-pack.png` |
| Same ROM in GB/DMG mode, followed by save/cold-boot checks | `emulator-dmg-question.png` |
| Native save dry-run and duplicate import through local HTTP mock | `emulator-import-report.json` (`networkRequests: 0`, two unique accepted identities) |

Portable C fault-injection tests additionally interrupt every byte position of an answer write and every header-write position during initialization. These assertions and the emulator's SRAM checks determine pass/fail; screenshots supply visual evidence. The hardware checklist above captures what simulation cannot establish.
