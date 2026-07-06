# Building a Custom FPGA Fabric with OpenFPGA — Theory & Reference Guide

This document explains **why** each tool and file exists in the OpenFPGA fabric-generation flow, not just how to run the commands. It's written to be handed to someone else as teaching notes.

---

## 1. The big picture — why four different tools?

Building an FPGA fabric from scratch isn't one program — it's a chain of four tools, each solving a completely different problem:

| Stage | Tool | What it actually does |
|---|---|---|
| 1 | **Yosys** | Converts a user's RTL Verilog into a technology-mapped netlist (a list of LUTs/gates) — decides *which logic primitives* implement a circuit |
| 2 | **VPR** (Versatile Place and Route) | Takes that netlist plus an *abstract* description of an FPGA's internal structure, and decides *where* each piece of logic sits on the chip and *how the wires connect them* |
| 3 | **OpenFPGA** | Takes VPR's abstract placement/routing result and a *physical* circuit description, then generates real, synthesizable **Verilog** for the actual fabric — LUTs, muxes, flip-flops, configuration memory, routing switches, all as real gates |
| 4 | **OpenLane** (built on OpenROAD) | Takes that fabric Verilog and produces actual chip geometry (GDS) — the file sent to a fab for real silicon |

**The one idea that explains almost everything else in this guide:**
VPR was built by academia (the VTR project, University of Toronto) purely to *study* FPGA architectures in simulation. It never had to produce real hardware, so its architecture description is abstract — "there's a LUT here" — with no notion of what a LUT is actually built from. OpenFPGA exists specifically to bridge that gap: it re-describes every abstract element in VPR's world as a real physical circuit, then stitches out real Verilog. This is why you need **two separate architecture files**, not one — they describe two different layers of the same device.

---

## 2. The three core input files — what each one is *for*

### 2.1 VPR architecture XML — "the shape of the device"

This file answers: how many logic blocks exist, how big is each one, how are they arranged in a grid, how much wiring exists between them, what does the I/O ring look like. Key concepts inside it:

- **CLB (Configurable Logic Block)** — the repeating logic tile. Contains one or more BLEs.
- **BLE (Basic Logic Element)** — one LUT + one flip-flop, the smallest unit of programmable logic.
- **LUT (Look-Up Table)** — a tiny memory that implements *any* logic function of N inputs by storing its truth table. An N-input LUT can realize any function of N variables just by pre-loading the right bit pattern.
- **FF (Flip-Flop)** — a 1-bit register, for holding state/pipeline stages.
- **Routing resources** — wire segments plus **switch blocks** (junctions where wires can turn/connect to other wires) and **connection blocks** (junctions where wires connect into a logic block's pins).
- **`<layout>`** — the physical floorplan. `tileable="true"` tells OpenFPGA to build switch/connection blocks as uniformly as possible, so an arbitrary array size only needs ~9 unique tile types instead of one-off tiles for every grid position — this matters enormously once you're targeting real silicon, since it keeps netlist size and place-and-route sane.
- **`<fixed_layout name="..." width="W" height="H">`** — a specific named device size. In every example we tested, `width = height = N + 2`, where N is the CLB grid size and the extra 2 is the one-tile-deep I/O ring wrapping the core on all sides.

This file alone is enough for VPR to run its own pack/place/route — but everything in it is abstract. It never says what a LUT is made of, physically.

### 2.2 OpenFPGA architecture XML — "how the device is physically built"

This is where abstraction becomes real hardware. It answers: what is a LUT actually made of (transistors, gates), what is a routing switch actually made of, how does the chip get configured (programmed) once fabricated. Its major parts:

- **`<technology_library>`** — process-level info (transistor models). Can be a placeholder if you're not running SPICE-level analysis.
- **`<circuit_library>`** — a catalog of circuit models, built in two layers:
  - **Atoms**: inverters, buffers, transmission gates (pass-gates), wire segments — the raw building blocks.
  - **Primitives**: real LUTs, multiplexers, flip-flops, I/O pads, built by composing atoms together. E.g. a 4-input LUT primitive is built from 16 SRAM bits + a tree of pass-gate multiplexers.
- **`<configuration_protocol>`** — how a bitstream gets loaded into the fabric after fabrication. The most common choice is `scan_chain`: every configurable bit in the chip is a flip-flop, and all of them are wired together into one long shift register, so a bitstream is shifted in serially, one bit at a time, through a single pin.
- **Bindings** (`<connection_block>`, `<switch_block>`, `<routing_segment>`, `<pb_type_annotations>`) — this is the "glue" layer. It maps every *named* element in the VPR architecture XML onto one of the physical circuit models defined above. The names must match exactly between the two files, and pb_type annotations must give the *full hierarchy path* (e.g. `clb.fle[n1_lut4].ble4.lut4`) to say which circuit model implements which specific primitive. **Name mismatches here are the single most common cause of a "from scratch" build failing.**

Without this file, OpenFPGA has an abstract routing result but no idea what to actually generate as Verilog.

### 2.3 Simulation settings XML — "how to test it"

Even if your only goal is the Verilog netlist, OpenFPGA's flow still requires this file, because it also generates verification testbenches alongside the fabric. It defines:
- The **operating clock** frequency (how fast the fabric runs the user's logic) — often set to `auto`, meaning it derives the frequency from VPR's own timing/routing results, with a safety margin (`slack`).
- The **programming clock** — a separate, usually slower clock used only while shifting in the configuration bitstream.
- Simulator-level settings: timing thresholds for measuring delay/slew, stimulus slew rates, accuracy settings.

---

## 3. The supporting files

| File | What it is | Why it exists |
|---|---|---|
| `design_variables.yml` | Plain key/value substitutions | Lets one architecture template support multiple parameter sets (e.g. device size) without hand-editing XML each time |
| `generate_fabric.openfpga` | A literal **script**, written in OpenFPGA Shell's own command language | This is the actual sequence of operations executed: read the two architecture files + sim settings, run VPR, build the fabric, repack, write bitstream, write Verilog, write testbenches, write timing constraints. `task.conf` just tells the runner which script to run and which files to feed it. |
| `and.blif` / `and.v` / `and.act` (or whatever benchmark you use) | A disposable test circuit | OpenFPGA can't place-and-route "nothing" — it needs *some* design routed through the fabric to produce a concrete netlist. The benchmark has nothing to do with the fabric's size or shape; a trivial 2-input AND gate works exactly as well as a real design for proving the fabric itself is correct. `.blif` (Berkeley Logic Interchange Format) is the synthesized-gates form VPR actually consumes; `.v` and `.act` (switching activity, for power estimates) just ride along for reference. |
| `fabric_key.xml` | A fixed pin/bitstream mapping | Generated by an earlier run and fed back in on later runs, to keep pin assignments and bitstream bit-ordering consistent across regenerations — important once you start doing physical verification, since you don't want the mapping to silently shuffle between runs. |
| `task.conf` | The orchestrator | Doesn't define the fabric itself — it just tells `run_fpga_task.py` which files to feed to which stage, in one place, so the whole run is reproducible from a single command. Think of it as closer to a Makefile than an architecture description. |

### Anatomy of `task.conf`

```
[GENERAL]           → run mode, output type (Verilog/SPICE/power), global timeout,
                       variable substitution file, which flow mode (e.g. vpr_blif
                       skips synthesis and expects a pre-made .blif; yosys_vpr
                       synthesizes RTL Verilog itself)
[OpenFPGA_SHELL]     → which .openfpga script to run, PLUS the actual OpenFPGA
                       architecture XML and sim settings XML paths, PLUS fixed
                       layout name / channel width overrides
[ARCHITECTURES]      → the VPR architecture XML
[BENCHMARKS]         → the disposable test design(s) routed through the fabric
[SYNTHESIS_PARAM]    → per-benchmark metadata: top module name, activity file,
                       original RTL source (for testbench reference)
[SCRIPT_PARAM_*]     → named variants of a run (e.g. different channel widths);
                       a task.conf can define several of these blocks
```

---

## 4. What `generate_fabric.openfpga` actually does, command by command

This script is the real "engine" of the flow. A typical version runs, in order:

1. **`vpr ...`** — invokes VPR directly: pack the benchmark netlist into CLBs, place them on the grid, route the connections, using the fixed device layout and channel width you specified.
2. **`read_openfpga_arch`** — loads the OpenFPGA architecture XML (circuit library, config protocol, bindings).
3. **`read_openfpga_simulation_setting`** — loads the sim settings XML.
4. **`link_openfpga_arch`** — cross-references the OpenFPGA architecture against VPR's own data, so every abstract element VPR used now has a physical circuit model attached to it.
5. **`build_fabric`** — actually constructs the module graph: every CLB, switch block, and connection block as real, physical circuit structures.
6. **`repack`** — re-packs the design considering only *physical* modes (the real circuit structure), rather than VPR's abstract operating modes. **This must run before any bitstream generation** — it's the step that reconciles "what VPR decided" with "what the real hardware looks like."
7. **`build_architecture_bitstream`** — computes the bitstream at the abstract architecture level (independent of physical wiring details) — useful mainly for correctness cross-checking.
8. **`build_fabric_bitstream`** — converts that into the actual physical bit ordering (scan-chain position, or memory address, depending on your configuration protocol).
9. **`write_fabric_bitstream`** (called with different `--format` flags) — writes the real, loadable bitstream file: `plain_text` (raw bits, meant for real hardware) and/or `xml` (annotated, easier to debug).
10. **`write_fabric_verilog`** — writes the actual synthesizable fabric netlist (`fpga_top.v` + supporting modules) — this is what feeds OpenLane.
11. **`write_full_testbench`** / **`write_preconfigured_testbench`** — generate simulation testbenches: one that models the full configuration-then-operation sequence (slower, more realistic), one that starts pre-configured (faster, for quick functional checks).
12. **`write_pnr_sdc`** / **`write_analysis_sdc`** / **`write_sdc_disable_timing_configure_ports`** — write timing constraint files (SDC) for the physical-design backend (OpenLane/OpenROAD), including special handling to exclude the slow configuration path from the main timing analysis (since it runs at a much lower frequency than normal operation).

**Key theoretical takeaway:** bitstream generation and Verilog generation happen in the *same* run, from the *same* command sequence — they are not sequential separate jobs you trigger independently. The bitstream configures the fabric to implement whatever benchmark was fed in; the Verilog describes the fabric itself, independent of any specific design.

---

## 5. Core concepts worth understanding deeply

- **Abstract vs. physical modeling** — the entire reason two architecture files exist. VPR's world is abstract (logical structure only); OpenFPGA's world is physical (real circuits). Nearly every confusing error in this flow traces back to a mismatch between these two layers (a name that exists in one file but not the other, a hierarchy path that doesn't match).
- **Tileable architecture** — building switch/connection blocks so they repeat identically across the array, rather than needing custom logic per grid position. Practically: this is *why* adding a new device size is "just one new `<fixed_layout>` block" rather than a redesign.
- **Channel width** — how many parallel routing wires exist in each routing channel. A width that comfortably routes a small array can become insufficient once the array grows, because more tiles compete for the same wiring resources. This is a completely separate failure mode from a missing layout — one is "VPR doesn't know this device size exists," the other is "VPR knows the size, but can't fit the routing."
- **Scan-chain configuration** — the most common way OpenFPGA-generated fabrics get programmed: every configuration bit is a flip-flop, all chained together, and the bitstream is shifted in serially through one pin, one clock cycle at a time.
- **Repack** — the translation step between "what VPR decided using abstract modes" and "what the real physical circuit structure requires." Always happens after `build_fabric`, always before bitstream generation.
- **The benchmark is not the fabric** — one of the most common conceptual trip-ups. The test design routed through the fabric during generation (`and2` in most examples) has nothing to do with the fabric's size, shape, or capability — it only exists because the tools need *something* concrete to place and route.

---

## 6. Common failure patterns (from real debugging)

| Symptom | Real cause | Why the error looked confusing |
|---|---|---|
| Task runner throws a generic `CalledProcessError`, `exit status 0` | The runner scans log output for the word "error" and re-raises its own wrapper exception — the real cause is buried in a log file, not in this traceback | The wrapper hides the actual VPR/OpenFPGA error message |
| Failure right after "Created total 1 jobs", nothing routes | A `--device <name>` value was requested that doesn't exist as a `<fixed_layout name="...">` in the architecture XML | Looks like an environment/toolchain problem; it's actually just a missing layout block |
| VPR reports it can't route at a given channel width | The fixed channel width that worked for a smaller device is too narrow once the array is scaled up | Easy to mistake for a layout problem instead of a capacity problem — check both independently |
| OpenFPGA build fails referencing a switch/segment/pb_type name | A name in `openfpga_arch.xml`'s bindings doesn't exactly match the corresponding name in `vpr_arch.xml` | These two files are edited somewhat independently, so drift between them is easy to introduce by hand |

---

## 7. The end-to-end pipeline, start to finish

1. Decide on your architecture parameters on paper first: LUT size, CLB structure, array size, routing style.
2. Start from a known-working baseline VPR architecture XML (don't write one blind) — copy a VTR reference template or an existing custom one.
3. Add physical I/O modeling and `tileable="true"` if not already present.
4. Add a `<fixed_layout>` block for your target size (`width = height = N + 2`).
5. Author the OpenFPGA architecture XML: circuit library (atoms → primitives), configuration protocol, and bindings that match your VPR arch XML's names exactly.
6. Author the simulation settings XML (clock/timing for testbenches).
7. Assemble a task folder: the three architecture files + a disposable benchmark + `generate_fabric.openfpga` + `task.conf` wiring it all together.
8. Run `run_fpga_task.py <task_name>` — this single command drives VPR, then OpenFPGA's build/repack/bitstream/Verilog/testbench/SDC generation, all in one pass.
9. Verify: check the fabric Verilog compiles, check the bitstream + testbench actually simulate correctly for the placeholder benchmark.
10. Feed the fabric Verilog into OpenLane, targeting `fpga_top` as the design top, to produce a real GDS.

---

## 8. What happens after fabrication (the ecosystem)

A fabricated chip is not, by itself, a finished deliverable — it's only useful paired with the toolchain that lets someone load new logic into it. The realistic post-fabrication pipeline:

1. **Fabricated chip** — packaged silicon. If this is an eFPGA tile (embedded inside a larger SoC, e.g. a Caravel-style harness with a management processor), it communicates with the rest of the chip over an internal bus, with a handful of exposed I/O pins.
2. **Package & bring-up** — power-on tests, clock stability checks, and critically: loading the exact placeholder-benchmark bitstream that was already verified in simulation, to confirm real silicon matches simulated behavior.
3. **Toolchain as a deliverable** — the three architecture files, packaged so an external user can go from their own RTL to a working bitstream without needing to understand OpenFPGA's internals directly.
4. **Loading the bitstream onto real silicon** — requires an actual physical mechanism (firmware on a management processor, or an external programmer) that shifts the bitstream into the real scan chain according to the timing the configuration protocol defined. This does not exist automatically — it has to be written specifically for the chosen configuration protocol.
5. **Running an application** — once configured, the fabric behaves like fixed logic implementing whatever the bitstream describes, indistinguishable from hardened logic until reconfigured.

---

*Compiled as a reference/teaching document based on hands-on OpenFPGA fabric bring-up (custom device sizing, from-scratch architecture authoring, and bitstream/Verilog generation).*
