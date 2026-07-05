# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Cross-platform telemetry plugin for Euro Truck Simulator 2 / American Truck Simulator. Fork chain: this repo ← RenCloud/scs-sdk-plugin ← truckermudgeon/scs-sdk-plugin. Adds macOS/Linux support by translating the Windows shared-memory API (`OpenFileMapping`/`MapViewOfFile`/`UnmapViewOfFile`/`CloseHandle`) to POSIX equivalents (`shm_open`/`mmap`/`munmap`/`shm_unlink`).

## Structure

- `scs_sdk/` — vendored, unmodified copy of SCS's telemetry SDK headers (`readme.txt`, `sdk_license.txt` govern it). Do not edit.
- `scs-telemetry/` — the actual plugin: `inc/` headers, `src/*.cpp`, and `scs-telemetry.def` (Windows module-definition file exporting the two functions the game loads: `scs_telemetry_init`, `scs_telemetry_shutdown`).
- Single CMake target (`scs_telemetry_plugin`, output renamed to `scs-telemetry`), sources globbed from both directories above. No examples/, docs/, or test directories.

## Build

```sh
cmake -B build
cmake --build build --config Release
```

Binaries land in `build/bin/`. Requires CMake 3.14+ and a C++ compiler (VS Build Tools on Windows, Clang on macOS/Ubuntu). C++14 only, no C sources despite the SDK's C-style headers.

Platform-specific compile settings (in `CMakeLists.txt`, don't remove without understanding why):
- `APPLE`: `CMAKE_OSX_ARCHITECTURES` forced to `x86_64` — must match the architecture the game executable targets.
- `WIN32`: defines `_CRT_SECURE_NO_WARNINGS UNICODE`; compile flags `/W4 /wd4100 /permissive-`.
- `UNIX AND NOT APPLE` (Linux): links `rt` — required for `shm_open`.

No test suite exists. CI (`.github/workflows/cmake.yml`) only builds Release artifacts on `v*` tag pushes across macOS/Ubuntu/Windows — it does not run on every push/PR and has no test step.

## Gotcha: one-shot event flags must be toggled, not assigned

Telemetry booleans that represent one-shot gameplay events (`fined`, `tollgate`, `ferry`, `train`, `refuelPayed`, `jobCancelled`, `jobFinished`, `jobDelivered`, in `scs-telemetry/src/scs_telemetry.cpp`) must be flipped with `^= true`, never set with `= true`. Consumers detect the event by diffing the flag's value frame-to-frame; a plain assignment latches the flag `true` forever and any event after the first goes unreported (this exact bug shipped for `refuelPayed` — see the "Changes in this fork" section of README.md). Any new one-shot flag must follow the same toggle pattern.

## Code style

No `.clang-format`/`.editorconfig` is enforced. Existing source uses Allman brace style, 4-space indent, `#pragma region`/`#pragma endregion` blocks for logical sections, handler functions named `handleXxxYyy`, and SCS SDK struct fields grouped by type-suffix (`_b`, `_i`, `_f`, `_s`, `_ll`, `_ui`).
