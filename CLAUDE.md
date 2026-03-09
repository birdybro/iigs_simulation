# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cycle-accurate Apple IIgs hardware simulation in Verilog/SystemVerilog. Runs under Verilator for development and targets Cyclone V FPGA (MiSTer platform). Emulates the complete system: 65C816 CPU, VGC, ES5503 sound, IWM floppy controller, HDD, ADB, SCC serial, and peripherals.

**FPGA constraint:** Don't modify clocks directly — use clock enables. Some signals labeled as clocks are actually enables. All RTL must remain synthesis-friendly for Quartus.

## Build Commands

**All commands must run from `vsim/` due to relative paths:**

```bash
cd vsim/
make                    # Build with ROM3 (default)
make ROM=rom1           # Build with ROM1
make SOUND=stub         # Stub out ES5503 sound system
make clean              # Clean build artifacts
```

## Running the Simulation

```bash
./obj_dir/Vemu                                          # Windowed simulation
./obj_dir/Vemu --disk totalreplay.hdv                   # Load HDD image (slot 7)
./obj_dir/Vemu --woz ArkanoidIIgs.woz                   # WOZ flux-level floppy
./obj_dir/Vemu --screenshot 245 --stop-at-frame 245     # Screenshot and exit
./obj_dir/Vemu --send-keys 100:hello\n                  # Inject keystrokes at frame 100
./obj_dir/Vemu --send-mouse 100:50,0,1,5                # Mouse: dx,dy,btn,duration
./obj_dir/Vemu --dump-vcd-after 400 --stop-at-frame 403 # VCD trace (keep <=3 frames)
./obj_dir/Vemu --enable-csv-trace                       # Memory trace CSV (~51% slower)
./obj_dir/Vemu -h                                       # Full option list
```

Disk options: `--disk`/`--disk2` (HDV, PO, 2MG), `--woz` (WOZ 1.x/2.x), `--floppy` (NIB)

## Regression Testing

After any RTL change, run regression from `vsim/`:

```bash
bash regression.sh
```

This runs 7 test cases (Total Replay, Pitch Dark, GS/OS, Arkanoid, Total Replay II, BASIC boot, MMU test) comparing screenshots frame-by-frame against `regression_images/`. If any test fails, stop and notify the user. You can analyze the PNG diffs to identify what changed.

## Core Architecture

### System Integration — `rtl/iigs.sv`
Top-level module wiring all subsystems. Handles memory controller (fast RAM banks 00-3F, slow RAM banks E0-E1, ROM banks FE-FF), I/O space mapping ($C000-$CFFF), shadow registers, and clock domain management (14MHz master, 2.8MHz fast, 1.024MHz slow).

### CPU — `rtl/65C816/`
Microcode-based 65C816 with 24-bit addressing. Key files: `P65C816.sv` (main), `mcode.sv` (microcode), `ALU.sv`, `AddrGen.sv`.

### Video — `rtl/vgc.v`
Dual-mode VGC: Super Hi-Res (IIgs native) and Apple II compatibility (Text 40/80, Lores 40/80, Hires 40/80, mixed). Uses `lineaddr()` for authentic Apple II non-linear memory addressing. Pixel buffer decouples fetch from output timing. AN3 signal ($C05E/$C05F) controls IIgs vs Apple II graphics mode selection.

Video mode priority when debugging: Text40/80 > Lores40/80 > Hires40/80 > Mixed modes.

### Disk Subsystem — WOZ Floppy
Flux-level floppy emulation for copy-protected software:
- `rtl/flux_drive.v` — Physical drive (motor, head, flux transitions)
- `rtl/iwm_woz.v` — IWM controller with WOZ/flux interface
- `rtl/iwm_flux.v` — Flux decoding and IWM register reads
- `rtl/woz_floppy_controller.sv` — WOZ format track management
- `vsim/sim.v` — Track data BRAM and C++ integration

### Other Key Modules
- `rtl/es5503.v` / `rtl/sound.v` / `rtl/soundglu.v` — Ensoniq 32-oscillator sound
- `rtl/adb.v` — Apple Desktop Bus (keyboard/mouse)
- `rtl/hdd.v` — Hard disk / SmartPort
- `rtl/scc8530.v` — Serial Communications Controller
- `rtl/clock_divider.v` — Speed switching (1.024MHz ↔ 2.8MHz)

### Simulation Harness — `vsim/`
- `sim.v` — Verilator top-level wrapper
- `sim_main.cpp` — C++ control, command-line parsing, frame loop
- `sim/sim_bus.cpp` — Bus simulation
- `sim/sim_blkdevice.cpp` — Disk image loading (HDV, WOZ, 2MG)
- `sim/sim_video.cpp` — Framebuffer and screenshot capture
- `sim/sim_input.cpp` — Keyboard/mouse injection

## Debug Macros

Uncomment `define` in source files to enable compile-time debug output:

| File | Macro | Notes |
|------|-------|-------|
| `rtl/iigs.sv` | `DEBUG_BANK`, `DEBUG_RESET`, `DEBUG_IO`, `DEBUG_IRQ` | Memory, reset, I/O, interrupts |
| `rtl/adb.v` | `DEBUG_ADB` | Very verbose (~20% speed) |
| `rtl/scc8530.v` | `DEBUG_SCC` | Serial controller |
| `rtl/clock_divider.v` | `DEBUG_CLKDIV` | Clock transitions |
| `vsim/sim.v` | `DEBUG_SIM` | Top-level sim events |

## Reference Materials

- `doc/` — Technical documentation (architecture, memory, video, HDD, SCC, etc.)
- `IIgsRomSource/` — Annotated IIgs ROM source
- `ref/` — Reference implementations for comparison
