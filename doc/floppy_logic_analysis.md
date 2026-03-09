# Floppy Controller Logic Analysis — Probable Issues

Analysis of `rtl/iwm_woz.v`, `rtl/flux_drive.v`, `rtl/iwm_flux.v`, and `rtl/woz_floppy_controller.sv` for incorrect logic.

## Summary — Ranked by Impact

| # | Severity | File | Issue |
|---|----------|------|-------|
| 11 | **Critical** | iwm_flux.v | `latch_hold_cnt` decrement inside `ifdef SIMULATION` — different FPGA behavior |
| 12 | **High** | flux_drive.v / iwm_woz.v | Chunk streaming ports unconnected — FLUX tracks >16KB broken on FPGA |
| 2 | **Medium** | iwm_woz.v | Mode register captured without Q6/Q7 check |
| 5 | **Medium** | woz_floppy_controller.sv | No `track_load_complete` after first 3.5" side load |
| 6 | **Medium** | flux_drive.v | `effective_bit_position` only handles 2x overflow |
| 9 | **Medium** | iwm_woz.v | 5.25" motor re-enable during spindown restarts full spinup |
| 7 | **Medium** | flux_drive.v | FLUX timing not normalized to rotation period |
| 8 | **Low** | iwm_flux.v | ASYNC_CLEAR_DELAY tuned to 2x MAME's value |
| 10 | **Low** | woz_floppy_controller.sv | Settling logic drops LSB for 5.25" track IDs |
| 3 | **Low** | iwm_flux.v | Duplicate prolog_last assignments in SIMULATION |
| 4 | **Low** | iwm_woz.v | Dead code in disk_mounted ternary |
| 1 | **Cosmetic** | flux_drive.v | Triple redundant assignment |

---

## Issue 1 — Triple Redundant Assignment (Cosmetic)

**File:** `rtl/flux_drive.v` lines 989-991

```verilog
bit_cell_cycles_reg <= bit_cell_base;
bit_cell_cycles_reg <= bit_cell_base;
bit_cell_cycles_reg <= bit_cell_base;
```

Copy-paste bug in the `track_load_reset` block. Only the last NBA wins in Verilog. Harmless but suggests this reset block was patched multiple times without cleanup.

---

## Issue 2 — Mode Register "Robust" Capture Without Q6/Q7 Check (Medium)

**File:** `rtl/iwm_woz.v` line 248

```verilog
if (cpu_access_edge && bus_wr && !iwm_active && (bus_addr == 4'hF)) begin
    mode_reg <= {3'b000, bus_din[4:0]};
```

This captures mode on **any** write to `$C0EF` while IWM is inactive, regardless of Q6 state. The proper mode register write (per MAME) requires Q6=1, Q7=1. The strict check at line 296 (`is_mode_write_access`) does verify Q6/Q7, but this "robust" fallback bypasses that.

**Impact:** If any software writes to `$C0EF` while Q6=0 and IWM is idle, this corrupts the mode register. The ROM likely always sets Q6 first, but this is a correctness bug that could break non-standard software.

**Fix:** Add `&& write_mode_q6` to the condition, or remove this fallback entirely and rely on the strict `is_mode_write_access` path.

---

## Issue 3 — Duplicate `prolog_last2`/`prolog_last1` Assignments (Low)

**File:** `rtl/iwm_flux.v` lines 805-806 vs 867-868 (EDGE_0 path), lines 1088-1089 vs 1150-1151 (EDGE_1 path)

In both the EDGE_0 and EDGE_1 byte completion paths, `prolog_last2`/`prolog_last1` are assigned **twice**: once in the non-ifdef synthesis path and again inside the `ifdef SIMULATION` block. In Verilog, the last NBA wins, so:

- **Simulation:** Second assignment (inside ifdef) takes effect — same values, no functional issue.
- **Synthesis:** Only the first assignment exists — correct.

Not a functional bug, but the duplication is confusing and risks divergence if someone changes only one copy.

**Fix:** Remove the duplicate assignments inside `ifdef SIMULATION`.

---

## Issue 4 — Dead Code in `disk_mounted` Wire (Low)

**File:** `rtl/iwm_woz.v` lines 875-877

```verilog
wire disk_mounted = DISK_READY[2] ? 1'b1 :
                   DISK_READY[0] ? 1'b1 :
                   (is_35_inch ? DISK_READY[2] : DISK_READY[0]);
```

The third branch is unreachable. If we reach it, both `DISK_READY[2]` and `DISK_READY[0]` are 0, so the expression always evaluates to 0. This simplifies to `DISK_READY[2] || DISK_READY[0]`. The dead fallback path may hide the original intent — perhaps it was meant to use `DISK_READY[1]` or `DISK_READY[3]` for drive 2.

**Fix:** Simplify to `wire disk_mounted = DISK_READY[2] || DISK_READY[0];` or add proper drive 2 support.

---

## Issue 5 — No `track_load_complete` After First 3.5" Side Load (Medium)

**File:** `rtl/woz_floppy_controller.sv` lines 997-1016

When loading both sides of a 3.5" track, `track_load_complete` only pulses after the **second** side finishes. The first side completes and immediately starts the second side's lookup without notifying `flux_drive`.

**Impact:** After the first side loads, `bit_position` continues from wherever it was. When the second side finishes and `track_load_complete` fires, `bit_position` resets to 0. If the ROM was reading data from the first side while the second side was loading, the sudden position reset desyncs byte alignment. The `track_data_valid` mechanism mitigates this, but there's a window where stale position data causes incorrect BRAM addressing.

**Fix:** Pulse `track_load_complete` after each side finishes, or at least after the side matching `stable_side` finishes.

---

## Issue 6 — `effective_bit_position` Only Handles 2x Overflow (Medium)

**File:** `rtl/flux_drive.v` lines 311-317

```verilog
wire pos_exceeds_1x = (bit_position >= track_bit_count_17) && (TRACK_BIT_COUNT > 0);
wire [16:0] pos_minus_1x = bit_position - track_bit_count_17;
wire pos_exceeds_2x = (pos_minus_1x >= track_bit_count_17) && (TRACK_BIT_COUNT > 0);
wire [16:0] pos_minus_2x = pos_minus_1x - track_bit_count_17;
wire [16:0] effective_bit_position = pos_exceeds_1x ?
                                     (pos_exceeds_2x ? pos_minus_2x : pos_minus_1x) :
                                     bit_position;
```

If `bit_position` exceeds 3x `track_bit_count` (e.g., a side switch halves the track size while position is near the end), `effective_bit_position` still overflows, producing an out-of-range BRAM address.

**Fix:** Add a 3x subtraction level, or clamp `effective_bit_position` to `track_bit_count - 1` as a safety net.

---

## Issue 7 — FLUX Timing Not Normalized to Rotation Period (Medium)

**File:** `rtl/flux_drive.v` lines 291-295

```verilog
wire flux_use_scaling = 1'b0;  // Disabled - use real 125ns timing
wire [31:0] flux_phase_inc = 32'd1000;
wire [31:0] flux_phase_mod = 32'd1790;
```

Scaling was disabled because it "caused timing mismatch with iwm_flux.v's fixed 28-cycle window timing." Without scaling, FLUX tracks whose total tick count doesn't equal exactly 200ms (one rotation at 300RPM) will either:
- **Finish early** (shorter track) — `flux_byte_addr` wraps to 0, replaying from the start
- **Finish late** (longer track) — the ROM never sees certain sectors

This works for most WOZ files but will fail for non-standard rotation speeds or tracks with unusual flux event density.

**Fix:** Either re-enable scaling with proper coordination between `flux_drive` and `iwm_flux` window timing, or handle the wrap/overrun case explicitly in `flux_drive`.

---

## Issue 8 — ASYNC_CLEAR_DELAY Tuned to 2x MAME's Value (Low)

**File:** `rtl/iwm_flux.v` line 155

```verilog
localparam [31:0] ASYNC_CLEAR_DELAY_14M = 32'd56;
```

MAME uses 14 cycles at 7MHz = 28 cycles at 14MHz. This was increased to 56 (4µs) because "the IIgs ROM has gaps in its read loop." The 2x increase may cause the opposite problem: stale bytes persisting too long, causing the CPU to read the same byte twice from different polling loops. This is a fragile tuning constant.

**Fix:** Investigate whether the ROM gap issue can be solved differently (e.g., through the `m_data_read` flag) rather than doubling the async clear delay. If the 56-cycle value is necessary, document which specific ROM routine requires it.

---

## Issue 9 — 5.25" Motor Re-Enable During Spindown Restarts Full Spinup (Medium)

**File:** `rtl/iwm_woz.v` lines 370-412

When `drive_on` goes low, `motor_spinup_done` is immediately cleared (line 404), but `motor_counter` starts counting down from `SPINDOWN_TIME`. If `drive_on` goes high again during spindown, the spinup logic sees `motor_spinup_done == 0` and starts the **full 300ms spinup delay** again — even though the motor never actually stopped spinning (it's still counting down from ~1 second spindown).

**Impact:** Real Disk II hardware maintains momentum; the motor should be instantly ready if re-enabled during spindown. The unnecessary 300ms delay after a brief motor-off toggle can cause ROM timeouts.

**Fix:** Don't clear `motor_spinup_done` when motor is disabled. Only clear it when `motor_counter` actually reaches 0 (motor fully stopped). Or: if `motor_spinning` is still true when `drive_on` re-asserts, skip the spinup delay.

---

## Issue 10 — Settling Logic Uses `track_id[7:1]` for 5.25" Drives (Low)

**File:** `rtl/woz_floppy_controller.sv` line 592

```verilog
if (track_id[7:1] != last_physical_track) begin
```

For 5.25" drives, `track_id` is a TMAP index (quarter-track value). Using `[7:1]` drops the LSB, meaning adjacent quarter-track steps (e.g., 20→21) don't trigger settle counter resets — the settle counter keeps incrementing from the prior change. This accidentally fast-tracks single quarter-track moves, which is fine.

However, for even-to-even quarter-track jumps (e.g., 20→22), `[7:1]` changes, resetting the counter and adding the full `SETTLE_THRESHOLD` of 50000 cycles (~3.5ms) delay. The settling logic was designed for 3.5" fast seeks during boot, not 5.25" single-track steps.

**Fix:** Use different settling logic for 5.25" vs 3.5" drives. 5.25" drives could use a shorter threshold or skip settling entirely (real Disk II hardware reads data immediately after stepping).

---

## Issue 11 — `latch_hold_cnt` Decrement Inside `ifdef SIMULATION` (Critical)

**File:** `rtl/iwm_flux.v` lines 577-589

```verilog
`ifdef SIMULATION
            if (!latch_mode) begin
                latch_hold_cnt <= 4'd0;
            end else if ((shift_edge0_now || shift_edge1_now) && latch_hold_cnt != 4'd0) begin
                latch_hold_cnt <= latch_hold_cnt - 1'd1;
                ...
`endif
```

The **latch hold countdown** logic is inside `ifdef SIMULATION`. In synthesis (FPGA), `latch_hold_cnt` is **never decremented** after being set to 8 (lines 967/1238). It can only be cleared by `rd_ack_take` (line 1421) or motor stop (line 1321).

**Impact:** In FPGA builds, `latch_hold_active` stays true for longer than intended, blocking the `async_clear` mechanism (line 631: `if (!latch_hold_active)`) and potentially keeping stale bytes in `m_data`. This causes **different behavior between simulation and synthesis** — the most dangerous class of bug.

**Fix:** Move the `latch_hold_cnt` decrement logic outside the `ifdef SIMULATION` block so it runs in both simulation and synthesis.

---

## Issue 12 — Chunk Streaming Ports Unconnected for FLUX Tracks >16KB (High)

**File:** `rtl/flux_drive.v` (port definitions) and `rtl/iwm_woz.v` (drive35 instantiation)

The chunk reload interface is defined in `flux_drive.v`:
- `CHUNK_RELOAD_REQ` (output) — requests next 16KB chunk
- `CHUNK_NEEDED` (output) — which chunk (0-3) is needed
- `CHUNK_LOADED` (input) — which chunk is currently in BRAM
- `CHUNK_LOADING` (input) — a chunk load is in progress

In the `drive35` instantiation in `iwm_woz.v` (line 458-500), these ports are **not connected**. The inputs `CHUNK_LOADED` and `CHUNK_LOADING` receive default values (0). For FLUX tracks larger than 16KB (common — FLUX tracks can be ~50KB), the chunk reload request fires but nothing responds. `CHUNK_LOADING` stays 0, so `flux_drive` reads garbage past the first 16KB chunk boundary.

**Impact:** All FLUX tracks >16KB produce corrupt flux data after the first 16KB, causing sector read failures for WOZ v3 flux-format disks.

**Fix:** Either connect the chunk ports to `woz_floppy_controller` (requires adding chunk reload support there), or increase the BRAM size to 64KB to hold the entire FLUX track without chunking. For simulation (where BRAM is not constrained), the latter is simpler.
