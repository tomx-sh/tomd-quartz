---
publish: true
created: 2026-08-05
modified: 2026-08-07T09:58:53.666Z
---

# A code-first workflow for ZSA Voyager firmware

[Voyager Keyboard](https://github.com/tomx-sh/voyager-keyboard) is a custom firmware project for the ZSA Voyager split keyboard. It replaces a configuration-first workflow with a small, reproducible codebase built around QMK.

The repository contains the personal keymap, build scripts, and visual configuration—not a complete copy of QMK. This keeps the project focused while making the keyboard layout reviewable, version-controlled, and reproducible.

## Project overview

The firmware targets a French macOS layout and defines three layers: Base, Numbers, and Symbols. It also includes home-row modifiers, custom tap-and-hold behavior, and static per-layer RGB feedback.

The project produces two artifacts from the same source:

- a firmware binary for the keyboard
- a printable SVG diagram of every layer

Keeping both outputs in one pipeline avoids the common problem of a layout diagram drifting away from the firmware it documents.

## Stack and tools

| Technology | Role |
| --- | --- |
| QMK | Keyboard firmware framework and keymap compiler |
| ZSA's QMK fork | Voyager hardware support and Oryx compatibility |
| C | Layers, key behavior, and RGB feedback |
| Docker | Isolated and repeatable firmware builds |
| Make and shell scripts | Stable commands for setup, build, validation, and flashing |
| Python and uv | Pinned tooling for layout generation |
| keymap-drawer | Printable SVG rendering from the QMK keymap |
| ZSA Zapp | Flashing the compiled firmware to the keyboard |

## Architecture

The repository follows QMK's external userspace model. The custom code lives under the same keyboard and keymap path that QMK expects, while the larger ZSA firmware repository is downloaded into a disposable build directory at a pinned revision.

The keymap's responsibilities are separated across a few files:

- `keymap.c` defines layers, behavior, and lighting.
- `i18n.h` centralizes French macOS key aliases.
- `config.h` contains compile-time settings.
- `rules.mk` enables only the required QMK features.
- `qmk.json` declares the keyboard and keymap build target.

Generated files stay outside the source tree. Firmware compilation runs in Docker and copies the resulting binary into `artifacts/`. The documentation pipeline converts `keymap.c` through QMK's own parser, applies the Voyager geometry from the pinned ZSA checkout, and renders the result with keymap-drawer.

This creates one clear dependency chain: the C keymap is the source of truth, while the firmware binary and layout diagram are derived artifacts.

## Development workflow

The Makefile exposes the project through a small command surface:

```sh
make setup
make check
make flash
```

`make setup` fetches the pinned QMK revision and installs the rendering tools. `make check` compiles the firmware and regenerates the SVG, so functional and visual validation happen together. Once the diff and diagram look correct, `make flash` rebuilds and installs the known artifact with Zapp.

Pinning QMK and the Python dependencies makes the build easier to reproduce. Docker prevents the firmware toolchain from leaking into the host environment, while `uv` keeps the lighter documentation tools fast to install and run locally.

## Project outcome

The result is a compact firmware repository with an explicit architecture and a short feedback loop. Keyboard behavior can be reviewed as code, every change produces matching documentation, and the complete layout can be rebuilt without depending on an opaque editor state.

The broader lesson applies beyond keyboards: keep the smallest useful source of truth, pin external systems, generate documentation from the implementation, and hide the toolchain behind a few dependable commands.
