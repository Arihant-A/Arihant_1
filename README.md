# VSD RTL Design & Synthesis Workshop — 12-Hour Lab Assessment

> **Tools Used:** Icarus Verilog · GTKWave · Yosys · SKY130 PDK
> **Platform:** GitHub Codespaces (Ubuntu)
> 
> **Date:** May 3, 2026

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [File Structure](#file-structure)
3. [Day 1 — RTL Simulation](#day-1--rtl-simulation-with-icarus-verilog--gtkwave)
   - [Lab 1: good_mux](#lab-1-good_mux--2-to-1-multiplexer)
   - [Lab 2: multiple_modules](#lab-2-multiple_modules--hierarchical-design)
   - [Lab 3: dff_asyncres — D Flip-Flop with Asynchronous Reset](#lab-3-dff_asyncres--d-flip-flop-with-asynchronous-reset)
   - [Lab 4: dff_async_set — D Flip-Flop with Asynchronous Set](#lab-4-dff_async_set--d-flip-flop-with-asynchronous-set)
   - [Lab 5: dff_syncres — D Flip-Flop with Synchronous Reset](#lab-5-dff_syncres--d-flip-flop-with-synchronous-reset)
4. [Day 2 — RTL Synthesis with Yosys + SKY130](#day-2--rtl-synthesis-with-yosys--sky130)
   - [Synthesis 1: good_mux](#synthesis-1-good_mux)
   - [Synthesis 2: multiple_modules (Hierarchical)](#synthesis-2-multiple_modules--hierarchical-synthesis)
   - [Synthesis 3: multiple_modules (Flat)](#synthesis-3-multiple_modules--flat-synthesis)
   - [Synthesis 4: sub_module1 (Partial Synthesis)](#synthesis-4-sub_module1--partial-synthesis)
   - [Synthesis 5: dff_asyncres](#synthesis-5-dff_asyncres--asynchronous-reset-flip-flop)
   - [Synthesis 6: dff_async_set](#synthesis-6-dff_async_set--asynchronous-set-flip-flop)
   - [Synthesis 7: dff_syncres](#synthesis-7-dff_syncres--synchronous-reset-flip-flop)
   - [Synthesis 8: mul2 — Multiplication Optimization](#synthesis-8-mul2--multiply-by-2-optimization)
   - [Synthesis 9: mult8 — Multiply by 8 Optimization](#synthesis-9-mult8--multiply-by-8-optimization)
5. [Key Learnings & Observations](#key-learnings--observations)
6. [Synthesis vs. Simulation Comparison](#synthesis-vs-simulation-comparison)
7. [SKY130 Cell Library — Reference](#sky130-cell-library--reference)

---

## Project Overview

This repository documents a **12-hour FPGA internship evaluation** covering end-to-end RTL design skills across two structured lab days:

| Day | Focus | Tools |
|-----|-------|-------|
| Day 1 | Behavioral RTL simulation, testbench authoring, waveform analysis | Icarus Verilog, GTKWave |
| Day 2 | Logic synthesis, technology mapping, netlist analysis, area estimation | Yosys, SKY130 PDK |

The modules evaluated span fundamental digital building blocks: combinational logic (multiplexers), hierarchical design, and sequential logic (D flip-flops with various reset styles), concluding with arithmetic optimization case studies.

---

## File Structure

```
vsd_rtl_lab/
├── verilog/
│   ├── good_mux.v
│   ├── tb_good_mux.v
│   ├── multiple_modules.v
│   ├── tb_multiple_modules.v
│   ├── dff_asyncres.v
│   ├── tb_dff_asyncres.v
│   ├── dff_async_set.v
│   ├── tb_dff_async_set.v
│   ├── dff_syncres.v
│   ├── tb_dff_syncres.v
│   ├── mul2.v
│   └── mult8.v
├── yosys_scripts/
│   ├── good_mux.ys
│   ├── mult_hier.ys
│   ├── mult_flat.ys
│   ├── asyn_res.ys
│   ├── asyn_set.ys
│   └── syn_res.ys
├── netlists/
│   ├── synth_mux.v
│   ├── synth_m_hier.v
│   ├── synth_m_flat.v
│   └── synth_asyn_res.v
├── images/         ← all waveform & schematic screenshots
└── README.md
```

| File | Description |
|------|-------------|
| `good_mux.v` | 2-to-1 MUX RTL design |
| `tb_good_mux.v` | Testbench: toggles sel every 75ns, i0 every 10ns, i1 every 55ns |
| `multiple_modules.v` | Top module instantiating AND (sub_module1) and OR (sub_module2) |
| `dff_asyncres.v` | DFF with active-high async reset |
| `dff_async_set.v` | DFF with active-high async set |
| `dff_syncres.v` | DFF with synchronous reset on clock edge |
| `mul2.v` | 3-bit input × 2 → 4-bit output |
| `mult8.v` | 3-bit input × 8 → 6-bit output |

---

## Day 1 — RTL Simulation with Icarus Verilog & GTKWave

### Simulation Workflow

```bash
# Compile design + testbench
iverilog -o sim.out <design>.v tb_<design>.v

# Run simulation (generates .vcd)
./sim.out

# Open waveform viewer
gtkwave tb_<design>.vcd
```

---

### Lab 1: `good_mux` — 2-to-1 Multiplexer

#### RTL Description

```verilog
module good_mux (input i0, input i1, input sel, output reg y);
always @ (*)
begin
    if (sel)
        y <= i1;
    else
        y <= i0;
end
endmodule
```

**Logic:** A classic 2-to-1 multiplexer. The `always @(*)` sensitivity list ensures the output `y` is recomputed for **any** change in inputs — making this fully combinational. When `sel = 1`, output follows `i1`; when `sel = 0`, output follows `i0`. Using `output reg` is correct practice for procedural assignments within `always` blocks.

#### Testbench

```verilog
`timescale 1ns / 1ps
module tb_good_mux;
    reg i0, i1, sel;
    wire y;

    good_mux uut (.sel(sel), .i0(i0), .i1(i1), .y(y));

    initial begin
        $dumpfile("tb_good_mux.vcd");
        $dumpvars(0, tb_good_mux);
        sel = 0; i0 = 0; i1 = 0;
        #300 $finish;
    end
    always #75  sel = ~sel;
    always #10  i0  = ~i0;
    always #55  i1  = ~i1;
endmodule
```

**Stimulus Design:** The three signals have deliberately chosen non-harmonic periods:
- `i0` toggles every **10 ns** (fastest) — stresses combinational pass-through
- `i1` toggles every **55 ns** — intermediate frequency
- `sel` toggles every **75 ns** — slowest, creating distinct windows to observe multiplexing

#### Simulation Waveform — Top-level View

![MUX Waveform (tb scope)](./images/image4.png)

**Waveform Analysis:**

The GTKWave output shows signals `i0`, `i1`, `sel`, and `y` over a 300 ns window.

- **Phase 1 (0–75 ns) — `sel = 0`:** Output `y` is identical to `i0`, toggling every 10 ns. The MUX correctly passes `i0`.
- **Phase 2 (75–150 ns) — `sel = 1`:** Output `y` now tracks `i1`, which has a 55 ns period. The clear transition at 75 ns confirms the MUX switches immediately (combinational, zero-latency).
- **Phase 3 (150–225 ns) — `sel = 0`:** `y` reverts to following `i0` again.
- **Conclusion:** The `always @(*)` block responds to all input changes without clock dependency. The output transitions **within the same simulation timestep** as the input change.

#### UUT Internal Signal View

![MUX Waveform (UUT scope)](./images/image5.png)

This expanded view (showing internal UUT signals `_0_` through `_3_` alongside port signals) confirms there are no internal latches or unexpected state elements — every signal transition is clean and combinational.

---

### Lab 2: `multiple_modules` — Hierarchical Design

#### RTL Description

```verilog
module sub_module2 (input a, input b, output y);
    assign y = a | b;   // OR gate
endmodule

module sub_module1 (input a, input b, output y);
    assign y = a & b;   // AND gate
endmodule

module multiple_modules (input a, input b, input c, output y);
    wire net1;
    sub_module1 u1 (.a(a), .b(b), .y(net1));  // net1 = a & b
    sub_module2 u2 (.a(net1), .b(c), .y(y));  // y = net1 | c = (a & b) | c
endmodule
```

**Logic:** Implements `y = (a AND b) OR c`. The design demonstrates **structural Verilog** — the top module `multiple_modules` is defined purely by instantiating sub-modules, not by writing behavioral logic at the top level. This mirrors how large SoCs are assembled from IP blocks.

#### Testbench

```verilog
`timescale 1ns / 1ps
module tb_multiple_modules;
    reg a, b, c;
    wire y;

    multiple_modules uut (.a(a), .b(b), .c(c), .y(y));

    initial begin
        $dumpfile("tb_multiple_modules.vcd");
        $dumpvars(0, tb_multiple_modules);
        a = 0; b = 0; c = 0;
        #300 $finish;
    end
    always #10  a = ~a;
    always #55  b = ~b;
    always #75  c = ~c;
endmodule
```

---

## Day 2 — RTL Synthesis with Yosys + SKY130

### Synthesis Workflow

```bash
# Launch Yosys
yosys

# Inside Yosys interactive shell:
> read_verilog <design>.v
> synth -top <module_name> -flatten   # or without -flatten for hierarchical
> dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
> abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
> show
> write_verilog -noattr synth_output.v
> stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

**Key Yosys Commands:**

| Command | Purpose |
|---------|---------|
| `read_verilog` | Parse RTL Verilog source |
| `synth -top <mod>` | Run full synthesis pass |
| `-flatten` | Merge all submodules into a single flat netlist |
| `dfflibmap` | Map flip-flop primitives to library cells |
| `abc -liberty` | Technology mapping using ABC engine against the SKY130 lib |
| `show` | Launch Dot Viewer to visualize synthesized schematic |
| `write_verilog -noattr` | Export gate-level netlist (no attributes for readability) |
| `stat -liberty` | Print area/cell statistics using actual cell areas from the library |

---

### Synthesis 1: `good_mux`

#### Yosys Script (`good_mux.ys`)

```tcl
read_verilog good_mux.v
synth -top good_mux -flatten
dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
write_verilog -noattr synth_mux.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![good_mux Yosys script in editor](./images/image3.png)

#### Synthesis Statistics

```
7. Printing statistics.

=== good_mux ===

   Number of wires:              8
   Number of wire bits:          8
   Number of public wires:       4
   Number of public wire bits:   4
   Number of memories:           0
   Number of memory bits:        0
   Number of processes:          0
   Number of cells:              1
     sky130_fd_sc_hd__mux2_1     1

   Chip area for module '\good_mux': 11.260800

Warnings: 26 unique messages, 234 total
End of script. Logfile hash: b945b11eab
CPU: user 0.42s system 0.04s, MEM: 41.45 MB total, 35.62 MB resident
Yosys 0.9 (git sha1 1979e0b)
Time spent: 50% 2x stat (0 sec), 38% 1x dfflibmap (0 sec), ...
```

![good_mux synthesis statistics terminal](./images/image1.png)

**Analysis:** Yosys synthesized the 2-to-1 MUX into exactly **1 cell** — `sky130_fd_sc_hd__mux2_1`. This is optimal. The ABC engine recognized that the RTL directly implements a MUX function and mapped it 1:1 to the library's native 2-input MUX cell. No decomposition into NANDs/NORs was required. Chip area: **11.26 μm²**.

#### Synthesized Schematic

![good_mux synthesized schematic](./images/image2.png)

**Schematic Analysis:**

The Yosys/Dot Viewer shows the synthesized netlist:
- Input ports `i0`, `i1`, and `sel` each pass through a **BUF** (buffer) cell before reaching the MUX inputs `A0`, `A1`, and `S` respectively.
- The core cell is `sky130_fd_sc_hd__mux2_1` (cell ID `$53`), mapping to the SKY130 library's standard 2-input multiplexer.
- The output `X` of the MUX passes through another BUF before reaching the output port `y`.
- The `$y` diamond represents the internal wire node.

The buffers are inserted by Yosys for signal integrity (drive strength normalization) and are a standard artifact of technology mapping. The critical path is `i0/i1 → BUF → MUX → BUF → y`.

---

### Synthesis 2: `multiple_modules` — Hierarchical Synthesis

#### Yosys Script (`mult_hier.ys`)

```tcl
read_verilog multiple_modules.v
synth -top multiple_modules
dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show multiple_modules
write_verilog -noattr synth_m_hier.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![multiple_modules hierarchical Yosys script](./images/image6.png)

**Note:** No `-flatten` flag is used here, so Yosys preserves the module hierarchy — `sub_module1` and `sub_module2` remain as distinct black boxes in the top-level netlist, synthesized independently.

#### Hierarchical Schematic

![multiple_modules hierarchical schematic](./images/image7.png)

**Schematic Analysis:** In hierarchical mode, the schematic shows exactly the RTL structure:
- Inputs `a` and `b` feed into `u1` (sub_module1 — AND gate), producing `net1` as intermediate signal.
- `net1` and `c` feed into `u2` (sub_module2 — OR gate), producing output `y`.
- The module boundary is clearly preserved: `sub_module1` and `sub_module2` appear as named sub-blocks, not as primitive gates.

#### Synthesis Statistics (Hierarchical — `multiple_modules`)

```
=== multiple_modules ===

   Number of wires:              5
   Number of wire bits:          5
   Number of public wires:       5
   Number of public wire bits:   5
   Number of memories:           0
   Number of memory bits:        0
   Number of processes:          0
   Number of cells:              2
     sub_module1                 1
     sub_module2                 1

   Area for cell type \sub_module2 is unknown!
   Area for cell type \sub_module1 is unknown!

=== sub_module1 ===

   Number of wires:              6
   Number of wire bits:          6
   Number of public wires:       3
   Number of public wire bits:   3
   Number of memories:           0
   Number of cells:              1
     sky130_fd_sc_hd__and2_0     1

   Chip area for module '\sub_module1': 6.256000

=== sub_module2 ===
```

![multiple_modules hierarchical stats part 1](./images/image8.png)

```
   Number of public wires:       3
   Number of public wire bits:   3
   Number of memories:           0
   Number of cells:              1
     sky130_fd_sc_hd__or2_0      1

   Chip area for module '\sub_module2': 6.256000

=== design hierarchy ===

   multiple_modules               1
     sub_module1                  1
     sub_module2                  1

   Number of wires:              17
   Number of wire bits:          17
   Number of public wires:       11
   Number of public wire bits:   11
   Number of memories:            0
   Number of memory bits:         0
   Number of processes:           0
   Number of cells:               2
     sky130_fd_sc_hd__and2_0      1
     sky130_fd_sc_hd__or2_0       1

   Chip area for top module '\multiple_modules': 12.512000
```

![multiple_modules hierarchical stats part 2](./images/image9.png)

**Analysis:**
- `sub_module1` (AND) → mapped to `sky130_fd_sc_hd__and2_0`, area **6.256 μm²**
- `sub_module2` (OR) → mapped to `sky130_fd_sc_hd__or2_0`, area **6.256 μm²**
- **Total design area: 12.512 μm²** — exactly 2× the AND2 cell, confirming the two equal-area cells.
- The "area unknown" warning for sub-modules at the top level is expected in hierarchical mode since the parent views them as black-box instances; the actual areas are resolved in the sub-module stats.

---

### Synthesis 3: `multiple_modules` — Flat Synthesis

#### Yosys Script (`mult_flat.ys`)

```tcl
read_verilog multiple_modules.v
synth -top multiple_modules
dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
flatten
show multiple_modules
write_verilog -noattr synth_m_flat.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![multiple_modules flat Yosys script](./images/image10.png)

#### Flat Schematic

![multiple_modules flat schematic](./images/image11.png)

**Schematic Analysis:** With `-flatten`, all module boundaries are dissolved and the full netlist is visible as primitive SKY130 cells in a single layer:
- Left side: inputs `b`, `a` → BUF → `u1.b`, `u1.a` → BUF → `sky130_fd_sc_hd__and2_0` (cell `$56`)
- AND output → `$y` → BUF → `u1.y` → BUF → `net1` → BUF → `u2.a`
- Right side: `c` → BUF → `u2.b`; `u2.a` and `u2.b` → BUF → `sky130_fd_sc_hd__or2_0` (cell `$58`)
- OR output → `$y` → BUF → `u2.y` → BUF → `y`

The flat netlist exposes all intermediate buffers and makes the AND-OR two-stage critical path explicit.

#### Flat Synthesis Statistics

```
=== multiple_modules ===

   Number of wires:              17
   Number of wire bits:          17
   Number of public wires:       11
   Number of public wire bits:   11
   Number of memories:            0
   Number of memory bits:         0
   Number of processes:           0
   Number of cells:               2
     sky130_fd_sc_hd__and2_0      1
     sky130_fd_sc_hd__or2_0       1

   Chip area for module '\multiple_modules': 12.512000
```

![multiple_modules flat stats](./images/image12.png)

**Hierarchical vs. Flat Comparison:**

| Metric | Hierarchical | Flat |
|--------|-------------|------|
| Top-level cells | 2 (sub_module1, sub_module2) | 2 (and2_0, or2_0) |
| Library primitives visible at top | No (black-box) | Yes |
| Total area | 12.512 μm² | 12.512 μm² |
| Optimization potential | Limited (per-module) | Higher (cross-module) |

The area is identical because this design is small and has no cross-module optimization opportunity. For larger designs, flattening enables the optimizer to eliminate redundant logic across hierarchy boundaries.

---

### Synthesis 4: `sub_module1` — Partial Synthesis

#### Yosys Script

```tcl
read_verilog multiple_modules.v
synth -top sub_module1
dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
write_verilog -noattr synth_m_hier.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![sub_module1 Yosys script](./images/image13.png)

**Purpose of Sub-Module Synthesis:** Synthesizing only `sub_module1` directly (with `-top sub_module1`) demonstrates how Yosys can target any module in a multi-module file. This is useful for:
- IP block validation before integration
- Faster iteration on individual modules in large designs
- Generating reusable synthesized netlists for modules used multiple times

#### Schematic

![sub_module1 schematic](./images/image14.png)

The schematic shows `a` and `b` as inputs, through BUF cells to ports `A` and `B` of `sky130_fd_sc_hd__and2_0` (cell `$51`), with output `X` buffered to `y`.

#### Statistics

```
=== sub_module1 ===

   Number of wires:              6
   Number of wire bits:          6
   Number of public wires:       3
   Number of public wire bits:   3
   Number of memories:           0
   Number of memory bits:        0
   Number of processes:          0
   Number of cells:              1
     sky130_fd_sc_hd__and2_0     1

   Chip area for module '\sub_module1': 6.256000
```

![sub_module1 stats](./images/image15.png)

---

### Synthesis 5: `dff_asyncres` — Asynchronous Reset Flip-Flop

#### RTL Description

```verilog
module dff_asyncres (input clk, input async_reset, input d, output reg q);
always @ (posedge clk, posedge async_reset)
begin
    if (async_reset)
        q <= 1'b0;
    else
        q <= d;
end
endmodule
```

**Logic:** D flip-flop with **active-high asynchronous reset**. The sensitivity list `posedge clk, posedge async_reset` means the process fires on either a rising clock edge OR a rising async_reset edge. When `async_reset` is asserted, `q` is immediately driven to 0 — no waiting for the next clock edge. This is critical for power-on initialization and emergency halts.

#### Testbench

```verilog
`timescale 1ns / 1ps
module tb_dff_asyncres;
    reg clk, async_reset, d;
    wire q;

    dff_asyncres uut (.clk(clk), .async_reset(async_reset), .d(d), .q(q));

    initial begin
        $dumpfile("tb_dff_asyncres.vcd");
        $dumpvars(0, tb_dff_asyncres);
        clk = 0; async_reset = 1; d = 0;
        #3000 $finish;
    end
    always #10  clk         = ~clk;
    always #23  d           = ~d;
    always #547 async_reset = ~async_reset;
endmodule
```

**Stimulus:** Clock period = 20 ns; reset asserted for ~547 ns cycles, then deasserted to allow normal DFF operation.

#### Simulation Waveforms

**Full View (0–3 µs):**

![dff_asyncres waveform full](./images/image16.png)

At t=0, `async_reset=1` forces `q=0` immediately. As long as `async_reset` is high, `q` stays at 0 regardless of `clk` or `d` transitions — confirming asynchronous behavior.

**Zoomed View (700 ns – 1.3 µs) — Reset Deassertion:**

![dff_asyncres waveform zoom](./images/image17.png)

At approximately **1030 ns**, `async_reset` goes low. From this point forward:
- `q` resumes following `d`, captured on the rising edge of `clk`.
- The transition of `q` at ~1030 ns + one clock edge demonstrates the flip-flop immediately resuming normal operation.
- The waveform confirms there is NO glitch or metastability — the reset release is clean.

**Critical Observation:** `q` goes to 0 instantaneously when `async_reset` rises, **without waiting for a clock edge** — the defining characteristic of asynchronous reset.

#### Yosys Script (`asyn_res.ys`)

```tcl
read_verilog dff_asyncres.v
synth -top dff_asyncres
dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
write_verilog -noattr synth_asyn_res.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![dff_asyncres Yosys script](./images/image22.png)

#### Synthesized Schematic

![dff_asyncres schematic](./images/image23.png)

**Schematic Analysis:**
- `async_reset` → BUF → `sky130_fd_sc_hd__clkinv_1` (inverter, cell `$52`). The SKY130 library's DFF with asynchronous reset (`dfrtp_1`) has an **active-low RESET_B** pin, so the active-high `async_reset` signal must be inverted before connecting to `RESET_B`.
- `clk`, `d`, and `q` connect directly to the `sky130_fd_sc_hd__dfrtp_1` cell (cell `$47`) at CLK, D, and Q ports.
- The internal signal `$50` (diamond shape) represents the inverted reset wire.
- BUF at the bottom drives the output buffer back to the module port.

**Key Insight:** The synthesis tool automatically inserts a `clkinv_1` (clock inverter) to adapt the active-high RTL reset polarity to the SKY130 library's active-low `RESET_B` pin. This is a critical example of technology mapping handling polarity conversion.

#### Synthesis Statistics

```
=== dff_asyncres ===

   Number of wires:              7
   Number of wire bits:          7
   Number of public wires:       4
   Number of public wire bits:   4
   Number of memories:           0
   Number of memory bits:        0
   Number of processes:          0
   Number of cells:              2
     sky130_fd_sc_hd__clkinv_1   1
     sky130_fd_sc_hd__dfrtp_1    1

   Chip area for module '\dff_asyncres': 28.777600
```

![dff_asyncres stats](./images/image24.png)

**Area:** 28.78 μm² — 2 cells (1 DFF + 1 inverter for polarity adaptation).

---

### Synthesis 6: `dff_async_set` — Asynchronous Set Flip-Flop

#### RTL Description

```verilog
module dff_async_set (input clk, input async_set, input d, output reg q);
always @ (posedge clk, posedge async_set)
begin
    if (async_set)
        q <= 1'b1;
    else
        q <= d;
end
endmodule
```

**Logic:** Identical structure to `dff_asyncres` but the asynchronous override **sets** `q` to `1` instead of resetting to `0`. When `async_set` pulses high, `q` is immediately forced to 1, regardless of clock.

#### Testbench

```verilog
`timescale 1ns / 1ps
module tb_dff_async_set;
    reg clk, async_set, d;
    wire q;

    dff_async_set uut (.clk(clk), .async_set(async_set), .d(d), .q(q));

    initial begin
        $dumpfile("tb_dff_async_set.vcd");
        $dumpvars(0, tb_dff_async_set);
        clk = 0; async_set = 1; d = 0;
        #3000 $finish;
    end
    always #10  clk       = ~clk;
    always #23  d         = ~d;
    always #547 async_set = ~async_set;
endmodule
```

#### Simulation Waveforms

**Full View (0–3 µs):**

![dff_async_set waveform full](./images/image18.png)

At t=0, `async_set=1` → `q=1` immediately. The `q` signal remains locked to 1 for the entire first phase where `async_set` is asserted, regardless of clock transitions or `d` values.

**Zoomed View (700 ns – 1.4 µs) — Set Deassertion:**

![dff_async_set waveform zoom](./images/image19.png)

At ~1090 ns, `async_set` falls low. Immediately after, `q` begins following `d` on clock edges, starting from a known `q=1` state. This confirms the asynchronous nature — the set release requires no clock edge to take effect.

**Contrast with `dff_asyncres`:** Reset forces `q=0`; Set forces `q=1`. Both respond outside the clock domain. Both are commonly used together in VLSI designs for full initialization control.

#### Yosys Script (`asyn_set.ys`)

```tcl
read_verilog dff_async_set.v
synth -top dff_async_set
dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
write_verilog -noattr synth_asyn_set.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![dff_async_set Yosys script](./images/image25.png)

#### Synthesized Schematic

![dff_async_set schematic](./images/image26.png)

**Schematic Analysis:**
- `async_set` → BUF → `sky130_fd_sc_hd__clkinv_1` (cell `$52`). Just as with the reset case, the active-high `async_set` is inverted.
- The target cell is `sky130_fd_sc_hd__dfstp_2` (cell `$47`) — this is the **set**-capable DFF variant in SKY130, with a `SET_B` (active-low set) pin, as opposed to `dfrtp_1` (reset-capable) used in the asyncres case.
- The structural similarity to `dff_asyncres` is evident — only the target flip-flop cell changes (`dfrtp_1` → `dfstp_2`), reflecting the different reset/set semantics.

#### Synthesis Statistics

```
=== dff_async_set ===

   Number of wires:              7
   Number of wire bits:          7
   Number of public wires:       4
   Number of public wire bits:   4
   Number of memories:           0
   Number of memory bits:        0
   Number of processes:          0
   Number of cells:              2
     sky130_fd_sc_hd__clkinv_1   1
     sky130_fd_sc_hd__dfstp_2    1

   Chip area for module '\dff_async_set': 30.028800
```

![dff_async_set stats](./images/image27.png)

**Area:** 30.03 μm² — slightly larger than `dff_asyncres` (28.78 μm²) because `dfstp_2` (drive strength 2) has a larger footprint than `dfrtp_1` (drive strength 1). This reflects real PDK cell library differences in drive strength options between reset and set variants.

---

### Synthesis 7: `dff_syncres` — Synchronous Reset Flip-Flop

#### RTL Description

```verilog
module dff_syncres (input clk, input async_reset, input sync_reset, input d, output reg q);
always @ (posedge clk)
begin
    if (sync_reset)
        q <= 1'b0;
    else
        q <= d;
end
endmodule
```

**Logic:** The sensitivity list is **only `posedge clk`** — no `async_reset` in the list. The `sync_reset` input is evaluated **inside** the clocked block, meaning the reset only takes effect at a rising clock edge. This is fundamentally different from async reset: the combinational reset path is gone. The `async_reset` input in the port list is unused in this implementation (a deliberate design choice in the lab to show the contrast).

#### Testbench

```verilog
`timescale 1ns / 1ps
module tb_dff_syncres;
    reg clk, sync_reset, d;
    wire q;

    dff_syncres uut (.clk(clk), .sync_reset(sync_reset), .d(d), .q(q));

    initial begin
        $dumpfile("tb_dff_syncres.vcd");
        $dumpvars(0, tb_dff_syncres);
        clk = 0; sync_reset = 0; d = 0;
        #3000 $finish;
    end
    always #10  clk        = ~clk;
    always #23  d          = ~d;
    always #113 sync_reset = ~sync_reset;
endmodule
```

#### Simulation Waveforms

**Full View:**

![dff_syncres waveform full](./images/image20.png)

**Zoomed View:**

![dff_syncres waveform zoom](./images/image21.png)

**Critical Observation:** When `sync_reset` goes high, `q` does **not** immediately go to 0. Instead, it waits until the **next rising edge of `clk`** before clearing. This is the fundamental behavioral difference:

| DFF Type | When does q respond to reset? |
|----------|-------------------------------|
| Async Reset | Immediately (clock-independent) |
| Sync Reset | Only on next posedge clk |

In the zoomed waveform, you can observe `sync_reset` asserting between clock edges — `q` holds its value until the next rising edge captures the `0` value, confirming synchronous behavior.

#### Yosys Script (`syn_res.ys`)

```tcl
read_verilog dff_syncres.v
synth -top dff_syncres
dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
write_verilog -noattr synth_syn_res.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![dff_syncres Yosys script](./images/image28.png)

#### Synthesis Statistics

```
=== dff_syncres ===

   Number of wires:              9
   Number of wire bits:          9
   Number of public wires:       5
   Number of public wire bits:   5
   Number of memories:           0
   Number of memory bits:        0
   Number of processes:          0
   Number of cells:              2
     sky130_fd_sc_hd__dftp_1     1
     sky130_fd_sc_hd__nor2b_1    1

   Chip area for module '\dff_syncres': 26.275200
```

![dff_syncres stats](./images/image29.png)

#### Synthesized Schematic

![dff_syncres schematic](./images/image30.png)

**Schematic Analysis — Most Interesting of the DFF Labs:**

The synchronous reset synthesis result is architecturally different from the async cases:
- **No separate reset pin on the DFF.** The library cell `sky130_fd_sc_hd__dftp_1` is a plain D flip-flop with **no reset input**.
- Instead, the reset logic is implemented in **combinational gates before the DFF's D input**:
  - `sync_reset` → BUF → port `A` of `sky130_fd_sc_hd__nor2b_1` (cell `$55`)
  - `d` → BUF → port `B_N` (inverted B input) of the NOR2B
  - The NOR2B computes `~(sync_reset NOR ~d)` = effectively a mux: if `sync_reset=1`, output=0; else output=d
  - The NOR2B output `Y` feeds into the `D` input of the DFF.
- The `sky130_fd_sc_hd__dftp_1` is a DFF with an active-low preset (`SET_B` not used here), clocked by `clk`.

**Key Insight:** The synthesizer correctly recognized that sync reset is a **combinational concern at the data input**, not a control signal for the flip-flop's dedicated reset pin. This is why SKY130's standard DFF (without built-in sync reset) plus a NOR gate achieves the correct behavior. This is an excellent demonstration of how the synthesis tool performs technology-aware mapping.

**Area:** 26.28 μm² — slightly less than the async variants because no polarity-inversion inverter is needed, just a NOR gate for the data mux logic.

---

### Synthesis 8: `mul2` — Multiply by 2 Optimization

#### RTL Description

```verilog
module mul2 (input [2:0] a, output [3:0] y);
    assign y = a * 2;
endmodule
```

**Logic:** A 3-bit input multiplied by 2, producing a 4-bit output. In binary arithmetic, multiplying by 2 is equivalent to a **1-bit left shift**: `y = {a, 1'b0}`. No actual multiplication hardware is needed.

#### Yosys Script

```tcl
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog mult_2.v
synth -top mul2
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
write_verilog -noattr synth_mult.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![mul2 Yosys script](./images/image31.png)

#### Synthesized Schematic

![mul2 synthesized schematic](./images/image32.png)

**Schematic Analysis:** The schematic shows a remarkable result — **no logic cells at all**. The entire `mul2` module is implemented as pure **wire connections**:
- Input bits `a[2:0]` (shown as `2:0`) are wired to output bits `y[3:1]` (shown as `3:1`)
- Output bit `y[0]` is hardwired to constant `0` (shown as `0 -> 0:0`)

This is the `{a, 1'b0}` shift implemented purely in wiring. The synthesis tool performed **constant propagation and bit-slice optimization** to eliminate all gates.

#### Synthesis Statistics

```
7. Printing statistics.

=== mul2 ===

   Number of wires:              2
   Number of wire bits:          7
   Number of public wires:       2
   Number of public wire bits:   7
   Number of memories:           0
   Number of memory bits:        0
   Number of processes:          0
   Number of cells:              0   ← ZERO CELLS
```

![mul2 stats](./images/image33.png)

**Result: 0 cells, 0 area.** The synthesis tool eliminated all hardware — this is a perfect example of **synthesis optimization** that RTL alone cannot express but Yosys/ABC recognizes.

---

### Synthesis 9: `mult8` — Multiply by 8 Optimization

#### RTL Description

```verilog
module mult8 (input [2:0] a, output [5:0] y);
    assign y = a * 8;
endmodule
```

**Logic:** 3-bit × 8 = 3-bit left shift by 3. Result: `y = {a, 3'b000}`.

#### Yosys Script

```tcl
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog mult_8.v
synth -top mult8
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib show
write_verilog -noattr synth_mult.v
stat -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![mult8 Yosys script](./images/image34.png)

#### Synthesized Schematic

![mult8 synthesized schematic](./images/image36.png)

**Schematic Analysis:** Again — no logic cells. Input bits `a[2:0]` wire to `y[5:3]`; output bits `y[2:0]` are grounded (constant 0). The label `2x 2:0 - 5:0` in the schematic represents the bit-extension mapping.

#### Synthesis Statistics

```
=== mult8 ===

   Number of wires:              2
   Number of wire bits:          9
   Number of public wires:       2
   Number of public wire bits:   9
   Number of memories:           0
   Number of memory bits:        0
   Number of processes:          0
   Number of cells:              0   ← ZERO CELLS
```

![mult8 stats](./images/image35.png)

**Result: 0 cells.** Both `mul2` and `mult8` confirm that **power-of-2 multiplications are synthesized as zero-cost wire connections** — no adders, no look-up tables.

---

## Key Learnings & Observations

### Synthesis vs. Simulation Comparison

| Aspect | Simulation (Icarus/GTKWave) | Synthesis (Yosys/SKY130) |
|--------|-----------------------------|--------------------------|
| **Purpose** | Verify behavioral correctness | Map RTL to real gates/cells |
| **Output** | VCD waveform file | Gate-level Verilog netlist |
| **Abstraction** | RTL (behavioral) | Structural (physical cells) |
| **Reset handling** | Simulates RTL intent directly | Must map to library primitives |
| **Timing** | Functional only (no real delays) | Cell-level area estimation |
| **Optimization** | None (simulates as-written) | Logic minimization, constant folding |
| **Tool** | `iverilog` + `vvp` | `yosys` + `abc` |

### Reset Style Impact on Synthesis

| Reset Type | Synthesis Result | SKY130 Cells Used |
|------------|-----------------|-------------------|
| Async Reset | DFF with RESET_B pin | `dfrtp_1` + `clkinv_1` |
| Async Set | DFF with SET_B pin | `dfstp_2` + `clkinv_1` |
| Sync Reset | Plain DFF + combinational mux | `dftp_1` + `nor2b_1` |

**The fundamental principle:** Asynchronous reset/set maps to dedicated flip-flop control pins (physical DFF reset). Synchronous reset is implemented as input data steering logic — the flip-flop itself has no knowledge of "reset"; it just captures a forced-0 value on its D input.

### Optimization Using SKY130

1. **Power-of-2 Multiplications → Zero Logic:** Multiplying by any power of 2 eliminates all cells. Yosys recognizes the shift pattern and generates pure wiring. This is a major area and power saving.

2. **Polarity Adaptation is Automatic:** When RTL uses active-high reset but the PDK cell has active-low `RESET_B`, Yosys automatically inserts `clkinv_1` cells. Engineers do not need to manually handle this polarity conversion in RTL.

3. **Hierarchical vs. Flat Synthesis:** For this lab, identical area. In real designs, flattening can reduce area 5–15% by enabling cross-boundary logic merging, but at the cost of longer synthesis runtime and harder design debugging.

4. **Single-MUX Mapping:** The `good_mux` design achieved perfect 1:1 RTL-to-cell mapping — one MUX cell for one MUX module. This shows the SKY130 library contains MUX primitives, not just NAND/NOR.

5. **NOR2B for Synchronous Mux Logic:** The `sky130_fd_sc_hd__nor2b_1` cell (NOR with an inverted B input) is used for the synchronous reset data-steering mux. This is more efficient than a NOR2 + INV because the inversion is built into the cell's internal topology.

---

## SKY130 Cell Library — Reference

Cells instantiated across this lab:

| SKY130 Cell | Function | Used In |
|-------------|----------|---------|
| `sky130_fd_sc_hd__mux2_1` | 2-to-1 Multiplexer (drive 1) | good_mux |
| `sky130_fd_sc_hd__and2_0` | 2-input AND (drive 0) | sub_module1 |
| `sky130_fd_sc_hd__or2_0` | 2-input OR (drive 0) | sub_module2 |
| `sky130_fd_sc_hd__dfrtp_1` | DFF with async Reset (active-low RESET_B, drive 1) | dff_asyncres |
| `sky130_fd_sc_hd__dfstp_2` | DFF with async Set (active-low SET_B, drive 2) | dff_async_set |
| `sky130_fd_sc_hd__dftp_1` | Plain DFF with Preset (drive 1) | dff_syncres |
| `sky130_fd_sc_hd__clkinv_1` | Clock Inverter (drive 1) | dff_asyncres, dff_async_set |
| `sky130_fd_sc_hd__nor2b_1` | NOR2 with inverted B input (drive 1) | dff_syncres |

**Library naming convention:** `sky130_fd_sc_hd__<cell_type>_<drive_strength>`
- `fd` = foundry design
- `sc` = standard cell
- `hd` = high density
- Drive strength: 0 = minimum, 1 = 1x, 2 = 2x, 4 = 4x

---

*README generated from VSD RTL Design & Synthesis Workshop lab documentation.*
*Environment: GitHub Codespaces · Yosys 0.9 · Icarus Verilog · GTKWave · SKY130 PDK (tt_025C_1v80)*
