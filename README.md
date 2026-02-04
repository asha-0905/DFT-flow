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
