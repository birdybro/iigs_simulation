# Clock Domain Crossing Analysis

## Clock Topology

**In simulation (`vsim/sim.v`):** Everything runs on a single clock. `CLK_28M`, `CLK_14M`, and `clk_vid` are all wired to the same `clk_sys` (line 157-159), and `ce_pix` is hardcoded to `1'b1` under `FASTSIM` (line 283). This means **CDC bugs are invisible in simulation**.

**On FPGA (`IIgs.sv`):** The PLL generates separate clocks from a common reference:

| Signal | Frequency | PLL Output |
|--------|-----------|------------|
| `clk_114` / `clk_mem` | 114.5 MHz | `outclk_0` |
| `clk_28` / `clk_vid` | 28.6 MHz | `outclk_3` |
| `clk_sys` / `CLK_14M` | 14.3 MHz | `outclk_4` |

So `CLK_14M` and `clk_vid` are **truly different clock domains on FPGA** — 14 MHz vs 28 MHz from separate PLL outputs. `CLK_28M` is also distinct at 28 MHz. All CDC issues between these domains are real on hardware.

---

## CRITICAL: `clk_vid` (28 MHz) ↔ `CLK_14M` (14 MHz) Crossings

These are the most consequential because both domains have substantial logic.

### 1. `vbl_irq` — VGC → iigs.sv interrupt manager
- **Source:** `vgc.v:980` — single-cycle pulse generated in `clk_vid` domain: `assign vbl_irq = v_blank & ~v_blank_d;`
- **Destination:** `iigs.sv:2339` — sampled in `CLK_14M` domain: `if (vgc_vbl_irq_pulse && INTEN[3])`
- **Sync:** Single-stage edge detect at `iigs.sv:2312-2313` (`vgc_vbl_irq_pulse_d`), but no double-flop synchronizer
- **Risk:** A single-cycle pulse at 28 MHz is ~35ns wide. The 14 MHz clock period is ~70ns. The pulse *could* be captured, but if it aligns with the 14 MHz setup/hold window, it causes metastability. Since `ce_pix` gates the pulse and isn't always `1'b1` on FPGA, the effective pulse may be only one `clk_vid` cycle — easily missed or metastable.

### 2. `mega2_vbl` — video_timing → iigs.sv CPU read ($C019)
- **Source:** `video_timing.v` — level signal generated in `clk_vid` domain
- **Destination:** `iigs.sv:1340` — combinational read in `CLK_14M` domain: `io_dout <= {mega2_vbl, key_keys}`
- **Sync:** None — direct combinational path
- **Risk:** Metastability on CPU read. This is the VBL status bit that software polls in tight loops.

### 3. `H` and `V` counters — video_timing → VGC and iigs.sv
- **Source:** `video_timing.v` — multi-bit counters in `clk_vid` domain
- **Destination:** `vgc.v` uses them for all address generation and mode detection; `iigs.sv` uses `V[8:0]` for `vbl_count` passed to ADB
- **Sync for VGC:** None needed — VGC also runs on `clk_vid`, so this is same-domain
- **Sync for iigs.sv/ADB:** `V[8:0]` is passed to `adb.v` (at `iigs.sv:2532`) — this is a **multi-bit crossing from `clk_vid` to `CLK_14M`** with no synchronization. Binary counter transitions can cause bit-tearing (e.g., 0xFF→0x100 could be sampled as 0x1FF).

### 4. CPU control signals → VGC
- **Source:** `iigs.sv` — mode flags (TEXTG, MIXG, HIRES_MODE, AN3, PAGE2, STORE80, EIGHTYCOL, etc.) written in `CLK_14M` domain
- **Destination:** `vgc.v` — sampled in `clk_vid` domain for rendering
- **Sync:** None
- **Risk:** Low for most flags (they change infrequently and are single-bit), but mode switches mid-scanline could cause a single glitched scanline. The `SHRG` flag at `vgc.v:627` controls major rendering path selection — a metastable sample there could cause a corrupted line.

### 5. `scanline_irq` — VGC → iigs.sv
- **Source:** `vgc.v` — generated in `clk_vid` domain
- **Destination:** `iigs.sv` — sampled in `CLK_14M` domain
- **Sync:** None visible
- **Risk:** Same class of issue as `vbl_irq`

### 6. VGC DPRAM — dual-port RAM at `iigs.sv:1984`
- **Source:** Port A in `CLK_14M` domain, Port B in `clk_vid` domain with `ce_pix` enable
- **Risk:** Dual-port RAM with different clocks on each port is architecturally valid on Cyclone V (true dual-clock BRAM), but the **address and write-enable signals** crossing into the BRAM must be stable relative to their respective clocks. The address generation in `iigs.sv` (CLK_14M side) writing while VGC reads (clk_vid side) needs no explicit CDC for the BRAM itself, but any handshaking signals around it do.

---

## HIGH: Clock Divider (`clock_divider.v`) — Speed Transition Hazards

### 7. `cyareg` — CPU speed register
- **Source:** Written by CPU in `CLK_14M` domain with `phi2` enable
- **Destination:** `clock_divider.v` uses it to generate `phi2_en`, `phi0_en`, `clk_7M_en`
- **Sync:** Single register (`cyareg_reg <= cyareg`) then combinational use in `slow_request`
- **Risk:** The clock divider generates all CPU timing enables. If `cyareg[7]` (speed bit) is metastable, it could produce a glitched `phi2_en`, causing a corrupted CPU cycle. This is mitigated by the fact that both are in `CLK_14M` domain — but only if the CPU write and the clock divider's sample are properly aligned to the same clock edge.

### 8. Motor detect flags
- Flags like `waitforC0C8` through `waitforC0F8` are set by address-match combinational logic and feed into `slow_request`
- These are within `CLK_14M` but the combinational depth could create timing violations on FPGA.

---

## HIGH: External Asynchronous Inputs (FPGA boundary signals)

These are signals from the MiSTer framework / host system that are asynchronous to all internal clocks.

### 9. `ps2_key[10:0]`, `ps2_mouse[24:0]` → ADB
- Multi-bit external inputs with no visible synchronizers in `adb.v`
- `keyboard.v:195` does edge detection directly on `PS2_Key[10]` — classic CDC violation

### 10. `img_mounted[5:0]`, `img_readonly`, `img_size` → sim.v / woz_floppy_controller
- `sim.v:474`: `if (~img_mounted5_d & img_mounted[5])` — edge detection on async input
- `woz_floppy_controller.sv` has the same pattern
- Multi-bit `img_size` used without synchronization when mount is detected

### 11. `sd_ack` → sim.v, woz_floppy_controller, woz_track
- Edge detection on async SD card acknowledgment signal throughout the disk subsystem
- `woz_floppy_controller.sv:172`: `if (old_ack && ~sd_ack)` — unsynchronized

### 12. `UART_RXD`, DCD → scc8530
- Serial receive data is inherently asynchronous
- `scc8530.v` line 53 comment explicitly acknowledges: inputs are not synchronized
- DCD inputs also unsynchronized

### 13. `timestamp[32:0]` → prtc.v
- 33-bit value compared against previous value for IRQ generation
- Multi-bit async crossing — susceptible to bit-tearing

---

## MEDIUM: Internal Enable-Domain Crossings

These are within `CLK_14M` but use different clock enables, which can create subtle issues.

### 14. `osc_en` — ES5503 → SoundGLU arbitration
- `soundglu.v:46`: `if (sound_cycle_state == ST_PENDING && osc_en)` — `osc_en` is derived from `clk_7M_en` in es5503, sampled in the `CLK_14M` domain
- Since both are enables on the same `CLK_14M` clock, this is **not a true CDC violation** — it's a combinational timing concern. The `osc_en` output is registered on `CLK_14M`, so it's safe as long as setup/hold is met.

### 15. `host_en` in ES5503
- CPU bus access gated by `phi2` enable, ES5503 oscillator cycles gated by `clk_7M_en`
- Both on `CLK_14M` — enable-domain arbitration, not true CDC. But register corruption is possible if `host_en` arrives on the same `CLK_14M` edge that an oscillator accesses the register file.

---

## Summary Table

| # | Signal(s) | Source Domain | Dest Domain | Sync Present | Severity |
|---|-----------|--------------|-------------|-------------|----------|
| 1 | `vbl_irq` | clk_vid (28M) | CLK_14M | 1-stage edge detect | **CRITICAL** |
| 2 | `mega2_vbl` | clk_vid (28M) | CLK_14M | None | **CRITICAL** |
| 3 | `V[8:0]` → ADB vbl_count | clk_vid (28M) | CLK_14M | None (multi-bit) | **CRITICAL** |
| 5 | `scanline_irq` | clk_vid (28M) | CLK_14M | None | **HIGH** |
| 4 | CPU mode flags → VGC | CLK_14M | clk_vid (28M) | None | **MEDIUM** |
| 6 | DPRAM port A/B | CLK_14M / clk_vid | (dual-clock BRAM) | Architectural | **LOW** |
| 7 | `cyareg` → clock_divider | CLK_14M | CLK_14M | 1-stage reg | **LOW** (same domain) |
| 9 | `ps2_key`, `ps2_mouse` | Async external | CLK_14M | None | **HIGH** |
| 10 | `img_mounted`, `img_size` | Async external | CLK_14M | None | **HIGH** |
| 11 | `sd_ack` | Async external | CLK_14M | None | **HIGH** |
| 12 | `UART_RXD`, DCD | Async serial | CLK_14M | None (acknowledged) | **HIGH** |
| 13 | `timestamp[32:0]` | Async external | CLK_14M | None (multi-bit) | **MEDIUM** |
| 14 | `osc_en` (soundglu) | CLK_14M (7M enable) | CLK_14M | Same clock | **LOW** |

## Key Takeaway

The **most dangerous** CDC issues are in the `clk_vid` ↔ `CLK_14M` boundary. In simulation these are invisible because `sim.v:157-159` ties all three clocks together. On FPGA, `clk_vid` is 28 MHz and `CLK_14M` is 14 MHz from separate PLL taps — every signal crossing between VGC/video_timing and iigs.sv's `CLK_14M` logic is a real CDC crossing. The `vbl_irq` pulse, `mega2_vbl` level, and `V` counter are the highest-risk signals since they directly affect interrupt delivery and software-visible state.

The external async inputs (PS/2, SD card, mount signals) are a secondary concern — they change infrequently so metastability windows are rare, but they have no protection at all.
