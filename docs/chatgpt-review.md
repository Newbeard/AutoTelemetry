# Telemetry v1 Task 4.5 review packet

## Task completed

Prepared and verified the development environment for future Telemetry v1 EDA, firmware, host testing, CAN/DBC, mechanical CAD, and manufacturing validation work. No Telemetry schematic, PCB, firmware, enclosure geometry, manufacturing output, procurement action, or copied third-party product design was created.

## Files changed

- Created `.gitignore` for project virtual environments and Python caches.
- Created `requirements-dev.txt` with the minimal host test and CAN/DBC dependencies.
- Created `docs/development-environment.md` with versions, activation, verified commands, reproduction steps, deferred tools, and known limitations.
- Updated `CODEX.md` with permanent KiCad/ESP-IDF version discipline, EDA source-of-truth and validation rules, symbol/footprint checks, mechanical source-of-truth and release gate, test/fixture principles, and manufacturing validation rules.
- Replaced this review packet, `docs/chatgpt-review.md`.

## Engineering findings

- Already present at inspection: Homebrew 4.6.12, Apple Git 2.50.1, Homebrew Python 3.13.3, and the official VS Code GUI. The required EDA, ESP-IDF, test, native-build, GitHub CLI, mechanical CAD, and formatting commands were absent.
- Installed or exposed during this task: KiCad 10.0.5 from its official signed DMG; CMake 4.4.2, Ninja 1.13.2, GitHub CLI 2.97.0, clang-format 22.1.8, Python 3.12.14, FreeCAD 1.1.3, and Miniforge through Homebrew; ESP-IDF v6.0.2 through official Espressif EIM; CadQuery 2.8.0 through conda-forge; and pytest/python-can/cantools in `.venv`.
- Environment changes were limited to vendor/user tool directories and clean PATH links for `code` and `kicad-cli`. The official `espressif.esp-idf-extension` 2.2.0 was added to VS Code; unrelated extensions and Git authentication were untouched.
- The verified host is macOS 26.5.2 build 25F84 on arm64 with Homebrew 4.6.12.
- KiCad 10.0.5 GUI and CLI are operational. Official symbols, footprints, 3D models, and templates are present. A disposable official template produced ERC, schematic PDF, DRC, Gerber, drill/map/report, CSV position, and board STEP outputs using locally verified KiCad 10 syntax.
- Official EIM 0.18.0 installed stable ESP-IDF v6.0.2. An official disposable `hello_world` project configured for ESP32-S3 and built successfully with GCC 15.2.0. No hardware was flashed.
- The project test environment is Python 3.12.14 with pytest 9.1.1, python-can 4.6.1, and cantools 42.0.3. The changing Homebrew `python3` default is 3.14.7 and is intentionally not the project runtime.
- FreeCAD 1.1.3 launched successfully. CadQuery 2.8.0 imported, created a disposable parametric box, exported STEP, and FreeCAD imported that STEP as one object.
- ESP-IDF does not support project paths containing spaces. The current path contains `my progect`; firmware work in this clone therefore needs a no-space path before it begins.

## Decisions made

- Froze KiCad major 10, currently 10.0.5, as the Telemetry v1 EDA source-of-truth version.
- Froze stable ESP-IDF v6.0.2 as the firmware major/minor baseline.
- Used native KiCad files as the only approved EDA authority and editable CadQuery/FreeCAD sources as mechanical authority.
- Chose an ignored repository `.venv` using explicit Python 3.12 for host tests and CAN tools; pinned only pytest, python-can, and cantools.
- Chose an isolated conda-forge CadQuery environment because the initial pip/OCCT environment did not import reliably on this workstation.
- Installed clang-format and retained ESP-IDF compiler diagnostics as the lightweight initial C/C++ checks. Deferred a larger analysis stack.
- Deferred SavvyCAN and final HIL equipment until a concrete interactive CAN or bench requirement exists.

## Assumptions

- Future contributors can use an administrator-managed official KiCad installation or reproduce the documented per-user installation while preserving KiCad major 10.
- The project will relocate or be re-cloned to a no-space path before firmware configuration becomes repository-local.
- Hardware serial access, port selection, vehicle fixtures, and HIL equipment will be addressed only after their interfaces and safety limits are approved.

## Uncertainties / unresolved questions

- The repository remains in a path unsupported by ESP-IDF because moving it was outside this task's authority.
- The VS Code extension is installed and the EIM environment is valid, but the user must perform the documented one-time **ESP-IDF: Configure ESP-IDF Extension** selection in the VS Code UI.
- The Homebrew KiCad cask's optional system-wide demos requested administrator access. No password was supplied; the official signed DMG was installed per-user instead.
- No physical ESP32-S3 board, serial port, USB-CAN adapter, vehicle data, GNSS simulator, or HIL equipment was available or required for this setup gate.
- FreeCAD reports the absent optional 3Dconnexion navigation framework. Standard GUI launch and STEP import work; no driver is needed unless a 3D mouse is later selected.

## Risks

- Running firmware builds from the current space-containing repository path can fail even though the isolated ESP32-S3 smoke build passed elsewhere.
- Silent KiCad major migration could rewrite authoritative design files; the new permanent review rule prevents this.
- CadQuery's large native geometry dependency stack is isolated, but it still requires controlled environment updates and a repeat STEP smoke test after upgrades.
- KiCad CLI validation reports violations from input designs; automation must preserve nonzero release gates and investigate rather than suppress them.
- Security-sensitive actions were avoided: no administrator password, Gatekeeper bypass, token exposure, Git identity/authentication change, untrusted binary, unknown package repository, random MCP server, or unofficial KiCad plugin was used.

## GPIO / peripheral changes

None. Task 4.5 changed tooling and policy only; the Task 4 frozen GPIO and peripheral map remains unchanged.

## Power / CAN / RF impact

- Power and RF design: no circuit, calculation, component, or layout changes.
- CAN: no vehicle profile, bus traffic, termination, or hardware change. `python-can` and `cantools` only prepare isolated future host tooling.
- ESP32: no firmware or target hardware change. ESP32-S3 build-tool availability was verified on an official disposable example.

## Datasheets / primary sources consulted

- Official Espressif ESP-IDF v6.0.2 installation documentation and official Espressif Installation Manager/Homebrew tap.
- Installed KiCad 10.0.5 local CLI help and official bundled KiCad template/libraries.
- Official CadQuery installation documentation and conda-forge package metadata.
- Official FreeCAD application distributed by the Homebrew cask.
- Local tool version output and the existing repository remote configuration.

No product component datasheet decisions were made in this tooling task.

## Recommended next step

Relocate or re-clone the repository to a path with no spaces, then repeat the documented environment smoke checks from that working copy.

## STOP condition

Task 4.5 is complete. Stop here and wait for engineering review. Do not begin Task 5, create the Telemetry v1 schematic, modify the PCB, implement firmware, create enclosure CAD, generate manufacturing files, or install additional unapproved tools.
