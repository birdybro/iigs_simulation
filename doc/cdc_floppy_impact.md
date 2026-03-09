# CDC Impact on Floppy Controller

## Directly Affecting Floppy Operation

### 1. `img_mounted[5]` / `img_mounted[4]` — async external → disk mount state machine
This is the entry point for mounting WOZ disk images. `sim.v:474` and `sim.v:576` do edge detection on these async MiSTer framework signals with no synchronizer. A metastable edge could cause the mount state machine in `woz_floppy_controller.sv` to see a false or double mount event. The same unsynchronized `img_mounted` is passed through to `woz_floppy_controller.sv` which does its own edge detection at line 175.

### 2. `sd_ack` — async external → woz_floppy_controller / woz_track
The SD card acknowledgment drives the track-loading state machine. `woz_floppy_controller.sv:172` does `if (old_ack && ~sd_ack)` — unsynchronized edge detection. Since track loading is a multi-step sequential read of WOZ track data from the SD interface, a glitched ack could cause a byte to be skipped or double-counted, corrupting track data in BRAM.

### 3. `DISK_READY[3:0]` — wired from woz_floppy_controller outputs → iwm_woz
At `sim.v:185`, `DISK_READY` is composed from `woz_ctrl_disk_mounted` and `woz_ctrl_525_disk_mounted`. These originate from the mount state machine (which itself depends on the unsynchronized `img_mounted` above). Inside `iwm_woz.v:441-444`, `DISK_READY[2]` gates whether the 3.5" drive is active and whether the motor spins — so a glitchy mount signal propagates all the way into the IWM's drive-active logic.

### 4. `track_load_complete` pulse — woz_floppy_controller → flux_drive
Generated in `woz_floppy_controller.sv:1020` as a single-cycle pulse, routed through `sim.v` to `iwm_woz`, then into `flux_drive.v`. Inside `flux_drive.v`, this pulse resets `bit_position` to 0 (line 1009) and reinitializes the flux decoding state. In simulation this is all one clock so it's fine, but the pulse originates from the `sd_ack`-driven state machine — so if the SD ack crossing is glitchy, the load-complete pulse could fire at the wrong time or be missed entirely.

### 5. Multi-bit track metadata — woz_floppy_controller → iwm_woz → flux_drive
Signals like `WOZ_TRACK3_BIT_COUNT` (32-bit), `WOZ_TRACK3_FLUX_SIZE` (32-bit), `WOZ_TRACK3_IS_FLUX` (1-bit) flow from woz_floppy_controller through sim.v into iwm_woz and flux_drive. These are outputs of the SD-loading state machine. If `sd_ack` glitches cause the state machine to advance prematurely, these multi-bit values could be sampled mid-update by the flux_drive, leading to incorrect bit counts or a mismatch between flux/bitstream mode and actual track data.

## Indirectly Affecting Floppy Operation

### 6. `cyareg` / motor detect → clock_divider → `phi2_en` timing
The floppy controller is extremely sensitive to CPU bus timing. `iwm_woz.v` samples the bus on PH2 edges (line 116: `wire bus_cen = PH2`). The clock_divider generates PH2 based on `slow_request`, which incorporates motor-detect flags and `cyareg[7]`. If a speed transition glitches `phi2_en`, the IWM could see a shortened or elongated bus cycle, missampling a soft-switch write. For copy-protected disks that depend on precise IWM timing, this could cause read failures.

### 7. `vbl_irq` / interrupt timing — indirect via CPU
Floppy disk I/O is often interleaved with VBL interrupt handling. If the `vbl_irq` CDC crossing (#1 in the main CDC analysis) causes a missed or delayed VBL interrupt, the CPU's interrupt service routine timing shifts, which can affect the tight loops that software uses for disk I/O polling. This is a second-order effect but relevant for copy-protected titles that rely on precise interrupt-relative timing.

## What Does NOT Affect Floppy

- `mega2_vbl` ($C019 read) — not used by the floppy subsystem
- `V[8:0]` → ADB — only affects keyboard repeat timing
- `scanline_irq` — not related to disk
- PS/2 keyboard/mouse — not related to disk
- SCC/UART — not related to disk
- PRTC timestamp — not related to disk
- Sound `osc_en` / `host_en` — not related to disk

## Bottom Line

The floppy controller's most direct CDC exposure is through the **SD card interface** (`sd_ack`, `img_mounted`) and the **clock divider's speed transition logic**. On FPGA, these unsynchronized external signals feed directly into the track-loading state machine and the mount detection, with corrupted state propagating through to `flux_drive`'s bit position and flux timing. In simulation none of this matters since everything is one clock, but on real hardware these are the likely sources of intermittent disk read failures.
