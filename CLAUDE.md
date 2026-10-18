# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is yabai

yabai is a tiling window manager for macOS that extends the built-in window manager. It uses binary space partitioning (BSP) for automatic window layout. Written in C and Objective-C with no external dependencies — only macOS system frameworks.

## Build Commands

```bash
make              # Debug build (clean + compile)
make install      # Production build (-O3)
make asan         # Build with AddressSanitizer
make tsan         # Build with ThreadSanitizer
make sign         # Code sign with "yabai-cert"
make clean        # Remove build artifacts (bin/ only)
make clean && make  # Full clean including OSAX generated sources

# Tests
cd tests && make all   # Build and run tests

# Documentation
make man          # Generate manpage from doc/yabai.asciidoc
```

Output binary goes to `./bin/yabai`.

## Build System Details

- Compiler: `xcrun clang` (universal binary: x86_64 + arm64)
- Standard: C11 (`-std=c11`), no ARC (`-fno-objc-arc`)
- Min deployment target: macOS 11.0
- All symbols hidden by default (`-fvisibility=hidden`)
- **Unity build**: `src/manifest.m` `#include`s all `.c` and `.m` files into a single compilation unit — there are no separate object files or linking steps
- **OSAX build step**: Before main compilation, `src/osax/payload.m` and `src/osax/loader.m` are compiled separately, then embedded as C byte arrays via `xxd` into `payload_bin.c` and `loader_bin.c`

## Architecture

### Core Components

- **`src/yabai.c`** — Entry point. CLI argument parsing, daemon initialization sequence, global state definitions
- **`src/manifest.m`** — Unity build orchestrator. All headers and source files are included here in dependency order
- **`src/event_loop.c/h`** — Thread-safe event queue with semaphore-based dispatch (pthread)
- **`src/message.c/h`** — IPC via Unix domain socket (`/tmp/yabai_<user>.socket`). Handles `yabai -m <command>` messages
- **`src/event_signal.c/h`** — User-defined signal/event subscriptions

### Manager Layer

- **`src/window_manager.c/h`** — Window state tracking, animation, opacity, layers, scratchpad
- **`src/space_manager.c/h`** — Workspace management, BSP tree views, layout application
- **`src/display_manager.c/h`** — Monitor arrangement, dock detection, display change handling
- **`src/process_manager.c/h`** — Application process lifecycle tracking

### Data Structures

- **`src/view.c/h`** — BSP tree (window_node) for layout. Supports BSP, stack, and float layouts
- **`src/window.c/h`** — Window state: frame, AXUIElementRef, flags (minimized, floating, sticky, fullscreen)
- **`src/space.c/h`** — Space/workspace with associated view and window list
- **`src/display.c/h`** — Display bounds and properties

### Scripting Addition (OSAX) — `src/osax/`

Injects code into Dock.app for privileged window server operations (requires partial SIP disable):
- **`payload.m`** — Runtime code injected into Dock.app
- **`loader.m`** — Handles injection into Dock.app
- **`common.h`** — Opcodes and socket protocol for yabai↔Dock.app communication via `/tmp/yabai-sa_<user>.socket`
- Architecture-specific: `x64_payload.m`, `arm64_payload.m`

### macOS Integration

- **`src/workspace.m`** — NSWorkspace notifications, macOS version detection
- **`src/sa.c/m`** — Client-side scripting addition interface
- **`src/mission_control.c`** — Mission Control event handling
- **`src/mouse_handler.c/h`** — Mouse event tracking for drag/resize
- **`src/misc/extern.h`** — Private SkyLight framework declarations (CGS* functions)

### Utilities (`src/misc/`)

Self-contained implementations: `hashtable.h`, `memory_pool.h`, `sbuffer.h` (string buffer), `ts.h` (temp storage), `helpers.h` (animation easing), `service.h` (launchd), `macho_dlsym.h` (dynamic symbol lookup).

## Key Frameworks

- **SkyLight** (private) — Window server control, space management (CGS* APIs)
- **Carbon** — Legacy window operations
- **Cocoa** — Foundation/AppKit for Obj-C integration
- **Accessibility API (AX)** — Window queries and manipulation

## IPC Protocol

Commands flow: `yabai -m <domain> <command>` → Unix socket → `message.c` parser → domain handler. Domains: `config`, `space`, `display`, `window`, `query`, `rule`, `signal`. Error responses are prefixed with `\x07`.

## macOS Version Handling

Version-specific code paths use `workspace_is_macos_*()` functions (monterey, ventura, sonoma, sequoia, tahoe). The OSAX payload has architecture-specific implementations (`x64_payload.m` / `arm64_payload.m`) because Mach-O patterns differ between Intel and Apple Silicon.

## Fork Workflow (Igor's checkout)

- `origin` = `idvorkin/yabai` (Igor's fork), `upstream` = `asmvik/yabai`.
- **Upstream doesn't accept PRs**: `asmvik/yabai` has pull requests disabled (`has_pull_requests: false`), so `gh pr create` fails with a `CreatePullRequest` permission error. Don't try. Pushing work to `idvorkin/yabai` is enough. The `upstream-pr/*` branches are clean cuts from `upstream/master`, in case the maintainer wants to apply them.
- Install a dev build with `~/.local/bin/yabai-dev-install [--no-build]`. It signs with Igor's "Apple Development" cert so the Accessibility grant survives rebuilds. An ad-hoc signed build gets a new hash every time and yabai aborts with `could not access accessibility features`. `/opt/homebrew/bin/yabai` is a symlink, and the script replaces its target with `rm` + `cp`. Copying over the existing file makes every launch get SIGKILLed (exit 137).
