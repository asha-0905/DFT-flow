# DFT-flow
This repository documents the complete DFT flow including scan insertion, scan compression (EDT/OCC), JTAG and boundary scan, MBIST, IJTAG, ATPG pattern generation, simulation, and final DFT signoff.

---

# DFT FLOW

This repository documents the complete Design-for-Testability (DFT).

---

## 📌 Table of Contents

1. Pre-DFT Setup
2. Scan Insertion Overview
3. Scan compression - EDT/OCC Insertion
4. ATPG
5. Simulation
6. JTAG, Boundary scan, MBIST 

---

## 🔷 1. Pre-DFT Setup

### 🎯 Objective

Prepare the design and tool environment before inserting any DFT structures.

### 🔹 Steps

* RTL design and functional verification are completed.
* The RTL is synthesized into a **gate-level netlist**.
* Import:

  * Gate-level netlist
  * Standard-cell libraries
  * Memory models
* Set the tool environment and define the design context (RTL or gate-level).

This stage ensures the design is stable and ready for scan insertion and ATPG.

---

## 🔷 2. Scan Insertion — Overview

## 2.1.  Overview

### 🎯 Objective

Make all sequential elements controllable and observable by converting functional flip-flops into scan flip-flops and connecting them into scan chains.

Scan insertion enables:

* Shift-in of test patterns
* Capture of internal responses
* Shift-out of test results

This allows ATPG tools to detect manufacturing defects efficiently.

---

## 2.2. Tool Invocation (Mentor Tessent)

The Tessent environment is initialized using:

```bash
source /home/cshrc
/home/TESSENT/bin/tessent -shell
```

This launches the interactive shell where scan insertion commands are executed.

---

## 2.3. Input and Output Files

### 📥 Input Files

* **Design Input File:** Gate-level netlist
* **Library File:** Standard-cell library defining the functionality and timing of each cell

### 📄 User-Created Files

* **Dofile:** Stores all tool commands for automation and reproducibility
* **Log File:** Captures tool execution messages, warnings, and errors

These files help debug and reproduce the scan insertion flow.

---

## 2.4. Scan Insertion Phases

Scan insertion proceeds through three major phases:

```
SETUP → ANALYSIS → INSERTION
```

---

### 🔹 SETUP Phase

* Set the tool context (scan / EDT).
* Read the design and library files.
* Elaborate the design.
* Perform initial DRC analysis.
* Fix basic violations before insertion.

---

### 🔹 ANALYSIS Phase

* Run full scan DRC checks.
* Analyze violations.
* Declare:

  * Number of scan chains
  * Clock domains
  * Scan enable signals

---

### 🔹 INSERTION Phase

* Replace functional flip-flops with scan flip-flops.
* Insert scan multiplexers.
* Stitch scan chains.
* Generate:

  * Scan-inserted netlist
  * Scan cell report
  * Scan chain report
  * ATPG dofile
  * ATPG test procedure
  * SCANDEF file

---

## 2.5. Scan Structure Insertion

### 🔹 Scan Multiplexer

Each scan flip-flop includes a multiplexer at its D-input:

```
Functional input → 0
Scan input       → 1
Select line      → Scan Enable (SE)
```

* **SE = 0:** Functional mode
* **SE = 1:** Scan shift mode

---

### 🔹 Scan Chains

All scan flip-flops are stitched serially to form scan chains, enabling:

* Shift-in of test patterns
* Capture of responses
* Shift-out of test results

---

## 2.6. Test Mode vs Scan Enable

| Signal               | Purpose                                                      |
| -------------------- | ------------------------------------------------------------ |
| **Test Mode (TM)**   | Distinguishes functional mode (TM = 0) vs test mode (TM = 1) |
| **Scan Enable (SE)** | Distinguishes shift phase (SE = 1) vs capture phase (SE = 0) |

---

## 2.7. Scan DRC (Design Rule Checks)

Scan DRCs ensure that flip-flops can be safely converted into scan flip-flops and controlled during test.

---

### ✅ 1. Clock Controllability DRC

**Problem:**

* Clock definition missing
* Clock driven by another flop or combinational logic

**Fix:**

```tcl
add_clock <off_state> <clock_name>
```

or insert a multiplexer.

---

### ✅ 2. Set/Reset Controllability DRC

**Problem:**

* Set/Reset connected to flop output or combinational logic
* Tied to constant
* Missing definition

**Fix:**

```tcl
add_pin_constraints <off_state C0/C1> <set/reset_name>
add_clock <off_state> <set/reset_name>
```

or insert a multiplexer.

---

### ✅ 3. Bus Contention DRC

**Problem:** Two or more drivers drive opposite values onto a bus.

**Fix:**

```tcl
set_test_logic -tristate on
```

---

### ✅ 4. Feedback Loop DRC

**Problem:** Output feeds back into input causing oscillation.

**Fix:** Insert a multiplexer.

---

### ✅ 5. X-Source DRC

**Problem:** Logic output becomes X due to analog blocks or memories.

**Fix:** Bypass the logic during test.

---

### ✅ 6. Potential Race DRC

**Problem:**

* Clock signal goes to both clock and data pins
* Same signal controls both set and reset

**Fix:** Insert a multiplexer.

---

## 2.8. Violation Codes and Fixes

| Violation | Meaning                                            | Fix                             |
| --------- | -------------------------------------------------- | ------------------------------- |
| **S1**    | Clock or set/reset missing or incorrectly driven   | `add_clock <off_state> <clock>` |
| **S2**    | Clock tied to constant                             | Insert mux                      |
| **D5**    | No scan equivalent cell or latch                   | Declare clock / replace cell    |
| **C6**    | Clock pin connected to D pin or another clock      | Insert mux                      |
| **C3**    | Positive-edge flop connected to negative-edge flop | Insert lockup latch             |
| **C7**    | Flop cannot capture due to invalid clock           | Insert mux                      |

Violations can be fixed using:

* Manual netlist editing
* Tessent commands:

  * `create_connection`
  * `delete_connection`
  * `create_instance`
* Auto-fix commands:

```tcl
set_test_logic -clock on
set_test_logic -set on
set_test_logic -reset on
set_test_logic -tristate on
set_test_logic -C6 on
```

---

## 2.9. Scan Chain Optimization

### 🔹 Scan Chain Balancing

* Chains are balanced to have nearly equal numbers of flops.
* Unbalanced chains increase test time.
* Dummy flops are added if required.

---

### 🔹 Edge and Clock Domain Mixing

* **Edge mixing:** Positive-edge and negative-edge flops are placed in the same chain.
* **Clock domain mixing:** Flops from different clock domains are placed in the same chain.

Command:

```tcl
insert_test_logic -number <num_chains> -edge merge -clock merge
```

---

## 2.10. Scan Chain Reordering

* Performed by the Physical Design (PD) team.
* Uses the **SCANDEF file** generated during scan insertion.
* Improves routing and physical implementation quality.

---

## 2.11. Outputs of Scan Insertion

### 📄 Generated Artifacts

1. **Scan-Inserted Netlist**

   * Functional flops replaced by scan equivalents
   * New ports added: `scan_en`, `scan_in`, `scan_out`

2. **Scan Chain Report**

   * Number of scan chains
   * Length and composition of each chain

3. **Scan Cell Report**

   * List of all scan cells inserted

4. **Scan DRC Report**

   * Lists all violations and fixes

---
🔷 3. Scan Compression (EDT + OCC)

🎯 Objective

Scan compression reduces the amount of test data that must be supplied by the tester while still maintaining high fault coverage. This is achieved using EDT (Embedded Deterministic Test) for data compression and OCC (On-Chip Clock Controller) for safe clock generation during scan and at-speed testing.

This step improves:

Tester memory usage

Test application time

Pin count requirements

At-speed fault coverage

🔷 3.1 EDT (Embedded Deterministic Test)

EDT inserts compression hardware that allows a small number of external scan channels to drive a large number of internal scan chains.

🔹 3.1.1 Decompressor Logic

The decompressor expands compressed external patterns into internal scan chains.

📌 LFSR (Linear Feedback Shift Register)

Given an initial seed, the LFSR generates pseudo-random patterns.

However, LFSR-generated patterns show a diagonal correlation between bits, limiting randomness and pattern diversity.

📌 Phase Shifter

LFSR outputs are passed through a phase shifter to increase randomness and decorrelate patterns.

Even after phase shifting, not all deterministic combinations may be reachable.

📌 External Channel Injection

To further improve pattern diversity, deterministic values from external tester channels are also injected into the decompressor network.

This hybrid approach enables the tool to achieve higher fault coverage.

🔹 3.1.2 Compressor and Mask Logic

After capture, responses from internal scan chains must be compacted into fewer external outputs.

📌 Compressor

The compressor consists of multiple levels of XOR trees.

It combines outputs from many scan chains into a smaller number of response channels.

However, if any scan chain contains an unknown value (X), the XOR tree propagates X, making the result unusable.

📌 Mask Logic

To prevent X-propagation, masking logic is inserted.

Masking selectively blocks problematic scan chains during capture.

📌 Decoders

To avoid excessive area overhead from masking logic, decoders are used.

Two types:

XOR Decoder

One-Hot Decoder

Decoders reduce the size and complexity of the masking network while maintaining observability.

🔷 3.2 External Channels vs Internal Scan Chains

EDT creates a mapping between:

External scan channels (tester-visible)

Internal scan chains (inside the design)

📌 Compression Ratio
Compression Ratio = Number of Internal Scan Chains / Number of External Channels


Higher compression ratios lead to:

Lower test time

Reduced tester memory usage

🔷 3.3 OCC (On-Chip Clock Controller)

🎯 Objective

OCC is responsible for selecting and generating the appropriate clocks during:

Functional mode

Scan shift mode

At-speed capture mode

🔹 Functions of OCC:

Determines which clock source is active in functional vs DFT modes

Controls the number of capture pulses during at-speed testing

Enables:

Launch-on-capture

Launch-on-shift

Safe multi-clock scan testing

OCC ensures that scan testing occurs without violating clock domain or timing constraints.

🔷 3.4 EDT + OCC Insertion Flow
🔹 Step 1: Set Context

Set the design context to DFT mode.

🔹 Step 2: SETUP Phase

Read the scan-inserted netlist:

read_verilog <scan_inserted_netlist>


Read the cell library:

read_cell_library <library_file>


Elaborate the design.

Use ATPG dofile (internally calls testproc).

Specify the number of external channels.

Configure EDT options:

set_edt_options ...

🔹 Step 3: ANALYSIS Phase

Check for DRC violations:

analyze_drc_violations


Common EDT-related violations:

K13: Information regarding newly inserted pins

E5: X-propagation violations

Fix EDT-related violations before insertion.

🔹 Step 4: INSERTION Phase

Perform EDT/OCC insertion and generate outputs:

write_edt_files


The tool automatically generates the complete script:

dc_script.scr

🔷 3.5 Link Library vs Target Library
🔹 Link Library

Used to resolve references in RTL or gate-level netlists.

Ensures that all instantiated cells are mapped to valid library cells.

🔹 Target Library

Used during synthesis/compile.

Specifies the technology node and optimization targets.

Design Compiler selects the smallest cells that meet timing and functional constraints.

🔷 3.6 Outputs of EDT + OCC Insertion
📄 Generated Artifacts:

Compressed Scan Netlist

Example: edt_top_gate.v

Contains inserted decompressor, compressor, masking logic, and OCC circuitry.

EDT/OCC Configuration Files

Channel mapping

Compression ratio

Instrument connectivity

These outputs are used directly for ATPG and simulation.
























































📌 1️⃣ MBIST INSERTION (Including JTAG, Boundary Scan, Wrapper & Graybox)

This replaces the earlier simplified MBIST section and now includes:

JTAG & Boundary Scan (IEEE 1149.1)

TAP architecture & FSM

Wrapper insertion (Dedicated & Shared)

Graybox generation

MBIST planning & algorithms

IJTAG network architecture

Tessent implementation flow

📌 1️⃣ MBIST INSERTION
🔷 PART A — JTAG & BOUNDARY SCAN (IEEE 1149.1)

Before MBIST insertion, the first step in a hierarchical DFT flow is JTAG insertion compliant with IEEE 1149.1.

🎯 Why JTAG is Needed?

Modern SoCs contain many internal DFT control signals:

TM

edt_bypass_en

int_mode

ext_mode

int_ltest_en

ext_ltest_en

mbist_bypass_en

and many more...

If all these signals are exposed at the top level:

❌ Top-level port count increases drastically
❌ Routing complexity increases
❌ Package cost increases

✅ Solution: Use JTAG

With JTAG, we need only 5 top-level ports:

TDI

TDO

TCK

TMS

TRST (optional)

Using these 5 pins, we can internally control dozens of DFT signals.

🔹 Static vs Dynamic Signals
Type	Controlled By
Static signals (constant during test)	JTAG (via TDR)
Dynamic signals (toggle during test)	Top-level
Examples

Static (JTAG controlled):

edt_bypass_en

TM

memory_bypass_en

ltest_en

Dynamic (Top-level):

edt_update

SE (Scan Enable)

🔷 JTAG ARCHITECTURE

JTAG consists of:

TAP (Test Access Port)

TAP Controller (16-state FSM)

Instruction Register (IR)

Data Registers (DRs)

Boundary Scan Register

🔹 TAP Signals

Defined by IEEE 1149.1:

Signal	Description
TDI	Serial test data input
TDO	Serial test data output
TCK	Test clock
TMS	Controls FSM state transitions
TRST	Optional asynchronous reset
🔹 Internal Architecture

JTAG includes:

MUX A

Selects between:

Instruction Register (IR)

Data Register (DR)

MUX B

Selects one Data Register based on decoded instruction.

Why Decoder?

To reduce IR width while supporting multiple DRs.

🔷 TAP CONTROLLER (16-State FSM)

The TAP Controller is a 16-state finite state machine.

🔹 Major States
1️⃣ Test-Logic-Reset

Entered when:

TRST_N = 0

TMS = 1 for 5 TCK cycles

Power-up

In this state:

Test logic disabled

Chip works normally

2️⃣ Run-Test/Idle

Example:
When performing core logic test, signals like:

int_mode

ext_mode

int_ltest_en

ext_ltest_en

are loaded via TDR.

FSM remains in Run-Test/Idle until operation completes.

3️⃣ IR Path States

Select-IR-Scan

Capture-IR
→ Loads fixed pattern (LSBs = 01)
→ Detects SA0/SA1 faults on TDI/TDO

Shift-IR
→ Serial loading via TDI/TDO

Exit-1 IR

Pause-IR

Exit-2 IR

Update-IR
→ Parallel load into hold register

🔹 IR Width Explanation

Mandatory Instructions:

EXTEST

SAMPLE/PRELOAD

BYPASS

Minimum IR width = 2 bits
Can be increased depending on number of DRs.

4️⃣ DR Path States

Select-DR-Scan

Capture-DR

Shift-DR

Exit-1 DR

Pause-DR

Exit-2 DR

Update-DR

🔷 JTAG INSTRUCTIONS
🔹 Mandatory Instructions
1️⃣ EXTEST

Tests PCB interconnections

Selects Boundary Scan Register

Used for board-level testing

2️⃣ SAMPLE/PRELOAD

Sample functional data

Preload test data before EXTEST

3️⃣ BYPASS

Skips device in JTAG chain

Uses 1-bit bypass register

🔹 Optional Instruction
IDCODE

Each chip has unique ID.

During Capture-DR:

IDCODE loaded

Shifted out via TDO

Used to:

Identify chips on PCB

Verify connection order

🔷 BOUNDARY SCAN (BSCAN)

Even if Chip1, Chip2, Chip3 pass ATE testing:

Board may fail due to:

Interconnect defects

Short/Open between chips

Solution → Boundary Scan
🔹 BSCAN Cell Structure

Modes:

ShiftDR	Mode	Operation
1	X	Shift data between cells
0	1	Apply test data to outputs
0	0	Normal operation
X	Capture	Capture response
🔹 BSCAN Chain

All BSCAN cells stitched into separate chain

Controlled via JTAG

Shifted through TDI → TDO

Each chip has its own JTAG controlling its BSCAN cells.

🔷 PART B — WRAPPER INSERTION
🎯 Why Wrappers?

In hierarchical DFT:

Block-level ATPG does NOT test inter-block interconnects.

Interconnect defects may remain undetected.

Solution → Wrapper Cells
🔹 Wrapper Chains

Scan chains around block boundary.

Connected to:

All PIs

All POs

🔹 Wrapper Types
1️⃣ Dedicated Wrappers

New wrapper cell inserted

Area overhead increases

Example command:

set_dedicated_wrapper_cell_options on -ports rst

2️⃣ Shared Wrappers

Reuse existing flop

No area overhead

Tool checks:
If PI drives flop → reuse flop as wrapper.

🔹 Wrapper Modes
INTEST

Inputs controllable

Outputs observable

Used for core internal logic testing

EXTEST

Outputs controllable

Inputs observable

Used for interconnect testing

🔹 Wrapper Commands
set_wrapper_analysis_options
set_dedicated_wrapper_cell_options
analyze_wrapper_chains

🔷 PART C — GRAYBOX GENERATION
🎯 What is Graybox?

A simplified representation of the core.

Contains:

Boundary logic

Wrapper chains

No internal core logic

Used when:

Only boundary logic is needed

Reduces complexity

🔹 Graybox Commands
get_config_elements
get_config_value
import_scan_mode
set_attribute_value [get_ports *edt_channel*] -name ignore_for_graybox
analyze_graybox
write_design -tsdb -graybox -verbose


Modes:

int_mode → internal

ext_mode → external

🔷 PART D — MBIST PLANNING
🔹 Memory Types
Type	Description
RAM	Read/Write
ROM	Read Only
🔹 Memory Fault Models
Single Cell Faults

**Stuck-at 0/1

**Transition fault

Double Cell Faults

**Coupling fault

**State coupling

**Inversion coupling

**Idempotent coupling

Address Decoder Faults

**No cell accessed

**Cell never accessed

**Multiple addresses → one cell

One address → multiple cells

🔷 Memory Test Algorithms

Test algorithm consists of:

Read/Write operations

Data pattern (0/1)

Address order (Ascending/Descending)

Example: March algorithms

🔷 PART E — TESSENT MBIST IMPLEMENTATION FLOW

Using Siemens EDA Tessent MBIST

🔹 Step 1 — Set Context
set_context dft -rtl -design_id first_insertion
set_tsdb_output_directory ../tsdb_outdir

🔹 Step 2 — Provide Files
read_cell_library ../../library/adk.tcelllib
read_verilog ../../library/mems/SYNC_1R1W_16x8.v
read_verilog ../design/corea.v

🔹 Step 3 — Elaborate
set_current_design corea
set_design_level physical_block

🔹 Step 4 — Define Clocks
add_clocks 0 clka -period 10ns
add_clocks 0 clkb -period 20ns


If missing → DFT_C1 violation.

🔷 DFT SPECIFICATION & IJTAG NETWORK
🔹 Add DFT Signals
add_dft_signal <signal_name>


Only for static signals.

Dynamic signals:

add_dft_signal <signal_name> -source node <top_port>

🔹 Create DFT Spec
set_dft_specification_requirements -memory_test on
create_dft_specification


Spec describes:

IJTAG network

MBIST partitioning

Memory grouping

Clock domains

🔹 SIB Types
SIB Type	Purpose
STI	MBIST instruments
SRI	Scan/EDT/OCC instruments

SIB acts as a switch in IJTAG.

🔹 Process DFT Spec
process_dft_specification


Performs:

Validation

Hardware generation

MBIST controller insertion

Memory interface insertion

IJTAG RTL + ICL generation

SDC generation

Writes outputs to TSDB

📤 MBIST FINAL OUTPUTS

✔ MBIST Controllers
✔ Memory Interfaces
✔ IJTAG Network (RTL + ICL)
✔ Wrapper Chains
✔ Graybox Netlist
✔ SDC Constraints
✔ Updated Netlist in TSDB
