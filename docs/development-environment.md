# Telemetry v1 development environment

## Approved baselines

This workstation was prepared and verified on 2026-08-14. Telemetry v1 uses KiCad major version 10 as its authoritative EDA format; the verified installation is KiCad 10.0.5. Future major-version migrations require an explicit engineering review. The firmware baseline is stable ESP-IDF v6.0.2. Do not silently change its major/minor line.

The host is macOS 26.5.2 (build 25F84), Apple silicon arm64. Homebrew 4.6.12 is the package manager. Only official vendor distributions, Homebrew, and conda-forge were used. No credentials, global Git identity changes, security bypasses, MCP servers, EDA plugins, or unrelated IDE extensions were introduced.

The pre-install inventory found Homebrew, Apple Git 2.50.1, Homebrew Python 3.13.3, and the official VS Code GUI already present. The VS Code shell command, pytest, KiCad CLI, ESP-IDF, CMake, Ninja, GitHub CLI, FreeCAD, CadQuery, and clang-format were absent. The later EIM/Homebrew dependency resolution changed the unpinned `python3` link, which is why this project uses explicit Python 3.12.

## Verified inventory

| Tool | Version / status | Installation and role |
|---|---|---|
| Git | 2.50.1 (Apple Git-155) | Existing Apple Git |
| Python default | 3.14.7 | Homebrew dependency; not the project runtime |
| Project Python | 3.12.14 | Homebrew `python@3.12`, isolated in `.venv` |
| pytest | 9.1.1 | `.venv`, pinned in `requirements-dev.txt` |
| python-can | 4.6.1 | `.venv`; near-term CAN capture/replay API |
| cantools | 42.0.3 | `.venv`; near-term DBC parsing/validation |
| KiCad | 10.0.5 | Official signed universal DMG, installed under `${HOME}/Applications/KiCad` |
| `kicad-cli` | 10.0.5 | Persistent `/opt/homebrew/bin/kicad-cli` symlink to the application bundle |
| ESP-IDF | v6.0.2 | Official Espressif Installation Manager (EIM) 0.18.0 and stable release |
| ESP32-S3 compiler | GCC 15.2.0, Espressif `esp-15.2.0_20251204` | ESP-IDF-managed `xtensa-esp32s3-elf-gcc` |
| System CMake / Ninja | 4.4.2 / 1.13.2 | Homebrew, general host use |
| ESP-IDF CMake / Ninja | 4.0.3 / 1.12.1 | Managed by the activated ESP-IDF environment |
| VS Code | 1.133.0 arm64 | Existing official application; `code` exposed on PATH |
| ESP-IDF VS Code extension | 2.2.0 | Official `espressif.esp-idf-extension` |
| GitHub CLI | 2.97.0 | Homebrew; useful for later review/CI inspection |
| clang-format | 22.1.8 | Homebrew; lightweight future C/C++ formatting |
| FreeCAD | 1.1.3, revision 20260725 | Homebrew official cask |
| CadQuery | 2.8.0 | Dedicated conda-forge Python 3.12 environment |

Git authentication and identity were not changed. The preserved remotes are:

```text
origin   git@github.com:Newbeard/AutoTelemetry.git
upstream https://github.com/MagnusThome/RejsaCAN-ESP32.git
```

## Python and CAN/DBC setup

Use an explicit Python 3.12 project environment; do not depend on the changing Homebrew `python3` default:

```sh
/opt/homebrew/bin/python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements-dev.txt
pytest --version
```

The environment itself is ignored. `python-can` and `cantools` are installed because CAN replay and reproducible DBC/profile fixtures have near-term value. SavvyCAN is deferred until interactive log inspection or hardware selection makes its GUI necessary. Do not add duplicate CAN, serial, or flashing utilities without a concrete requirement.

## KiCad setup and libraries

The Homebrew KiCad cask attempted an optional system-wide demo installation requiring administrator access. No password was supplied. The same official Homebrew-cached, signed KiCad 10.0.5 universal DMG was therefore installed per-user at `${HOME}/Applications/KiCad`. A normal administrator-managed workstation may instead use `brew install --cask kicad`. The project contains no dependency on a username-specific path.

Official bundled resources were verified under `KiCad.app/Contents/SharedSupport`: `symbols`, `footprints`, `3dmodels`, and `template`. The GUI launched successfully. The command-line link is:

```sh
ln -s "${HOME}/Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli" /opt/homebrew/bin/kicad-cli
kicad-cli version
```

The following KiCad 10.0.5 syntax was checked from local help and executed on a disposable official template outside this repository. ERC, PDF, DRC, Gerber, drill/map/report, CSV position, and STEP outputs were all produced. The template's own reported ERC/DRC violations and missing legacy `${KICAD9_3DMODEL_DIR}` component models are test-input findings, not tool failures.

```sh
kicad-cli sch erc --severity-all --exit-code-violations --output build/erc.rpt design.kicad_sch
kicad-cli sch export pdf --output build/schematic.pdf design.kicad_sch
kicad-cli pcb drc --severity-all --schematic-parity --exit-code-violations --output build/drc.rpt design.kicad_pcb
kicad-cli pcb export gerbers --board-plot-params --output build/gerbers design.kicad_pcb
kicad-cli pcb export drill --output build/drill --generate-map --map-format pdf --generate-report --report-path build/drill-report.rpt design.kicad_pcb
kicad-cli pcb export pos --format csv --units mm --output build/positions.csv design.kicad_pcb
kicad-cli pcb export step --force --output build/board.step design.kicad_pcb
```

Use `--exit-code-violations` in release validation. Do not omit or hide violations merely to make CI green. Production export layer and variant choices must be reviewed rather than copied blindly from the smoke-test example.

## ESP-IDF and VS Code

ESP-IDF was installed through the official Espressif Homebrew tap and EIM, then fixed to stable v6.0.2 at `${HOME}/.espressif/v6.0.2/esp-idf`. Activate it without committing a machine path:

```sh
source "${HOME}/.espressif/tools/activate_idf_v6.0.2.sh"
idf.py --version
```

EIM owns the isolated Python environment and compiler/build tools under `${HOME}/.espressif/tools`; do not replace the managed compiler with a system compiler. ESP32-S3 support was verified by copying the official `hello_world` example to a disposable no-space path, running `idf.py set-target esp32s3`, and completing `idf.py build`. No device was flashed.

ESP-IDF already supplies the required operations: `idf.py build`, `idf.py flash`, `idf.py monitor`, `idf.py flash monitor`, and `idf.py erase-flash`. Flashing resets through the supported serial interface; recovery/boot-mode behavior remains hardware-dependent. No duplicate global `esptool` installation was added.

The official VS Code extension is installed. On first use, run **ESP-IDF: Configure ESP-IDF Extension**, choose the existing EIM installation, and select v6.0.2. Keep user paths in VS Code user settings, not committed repository settings. Hardware serial-port selection also remains a user setting.

ESP-IDF documents that its own path and project path must not contain spaces. The working copy was relocated to `${HOME}/AutoTelemetry` and reverified there on 2026-08-17. From that repository working directory, EIM activation, `idf.py set-target esp32s3`, and a complete disposable `hello_world` build under `/private/tmp` succeeded. The original space-in-path blocker is resolved. Because Python virtual-environment launchers contain absolute interpreter paths, recreate `.venv` from `requirements-dev.txt` after any future repository relocation rather than copying the environment.

## Mechanical CAD

FreeCAD 1.1.3 GUI launch and headless operation were verified. CadQuery 2.8.0 is isolated from the project test environment at `${HOME}/.virtualenvs/telemetry-cadquery-conda`, created with Miniforge/mamba and conda-forge:

```sh
mamba create -y -p "${HOME}/.virtualenvs/telemetry-cadquery-conda" -c conda-forge python=3.12 cadquery=2.8.0
mamba run -p "${HOME}/.virtualenvs/telemetry-cadquery-conda" python -c 'import cadquery; print(cadquery.__version__)'
```

A disposable 20 mm x 10 mm x 5 mm box imported successfully in CadQuery, exported as STEP, and imported into FreeCAD as one object. The original pip/OCCT approach was rejected after runtime incompatibility; the conda-forge environment follows CadQuery's better-tested installation route. Production mechanical work must follow `CODEX.md`: editable parametric CAD is authoritative and STEP/STL/3MF are interchange or derived outputs.

FreeCAD reported that the optional 3Dconnexion navigation framework is absent. This did not prevent GUI launch, headless operation, or STEP import; install a vendor 3D-mouse driver later only if that hardware is selected.

The future mechanical workflow must support the core-device enclosure, display enclosure, and a separate vehicle-specific mount. It must also accommodate measurements, photographs, caliper data, scans or photogrammetry references, parametric revisions, prototype printing, fit tests, STEP delivery, and derived STL/3MF files. No product geometry was created here.

## Test and HIL preparation

Host tests will eventually cover normalized telemetry, vehicle profiles, CAN/diagnostic and GNSS replay, Generic OBD-II, ISO-TP, UDS, RaceChrono and first-party protocols, configuration, fault handling, and power-state logic. Hardware-dependent tests must use separate markers/directories from deterministic host tests.

A future HIL bench requires, but does not yet select or purchase: a workstation-controlled USB connection to Telemetry hardware, a supported USB-CAN interface, programmable CAN traffic/replay, GNSS/UART simulation or replay, controllable protected power and reset/boot access, current measurement, captured logs, deterministic fixtures, and automated pass/fail collection. HIL equipment selection is deferred until electrical interfaces and test limits are frozen.

## Lightweight firmware checks

Use repository formatting rules with the installed `clang-format` once source exists, compiler warnings from ESP-IDF, and ESP-IDF-native build diagnostics. Define an explicit warning policy before treating warnings as errors across third-party components. A large static-analysis stack is deferred until there is first-party firmware to analyze.

## Reproduction checklist

1. Install Homebrew from its official distribution if absent.
2. Install `python@3.12`, `cmake`, `ninja`, `gh`, `clang-format`, `dfu-util`, FreeCAD, and Miniforge through Homebrew.
3. Install official KiCad 10 and verify its bundled symbols, footprints, and 3D models; expose the bundled `kicad-cli` on PATH.
4. Install and trust the official Espressif EIM tap, use EIM to install stable ESP-IDF v6.0.2, then activate it with the generated script.
5. Install the official VS Code ESP-IDF extension and point it at the EIM-managed v6.0.2 installation.
6. Create `.venv` with Python 3.12 and install `requirements-dev.txt`.
7. Create the isolated CadQuery conda-forge environment and verify FreeCAD STEP import.
8. Re-run the KiCad disposable-project exports and ESP32-S3 `hello_world` build without modifying product design files.

Official setup references: [KiCad downloads](https://www.kicad.org/download/), [ESP-IDF stable ESP32-S3 macOS setup](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/get-started/linux-macos-setup.html), and [CadQuery installation](https://cadquery.readthedocs.io/en/stable/installation.html).
