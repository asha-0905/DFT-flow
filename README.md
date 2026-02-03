# DFT-flow
This repository documents the complete DFT flow including scan insertion, scan compression (EDT/OCC), JTAG and boundary scan, MBIST, IJTAG, ATPG pattern generation, simulation, and final DFT signoff.

---

# DFT FLOW

This repository documents the complete Design-for-Testability (DFT) scan insertion flow used in industrial ASIC/SoC designs. The content is written for beginners while maintaining professional accuracy, and it follows Mentor Graphics (Siemens Tessent) tool methodology.

---

## 📌 Table of Contents

1. Pre-DFT Setup
2. Scan Insertion Overview
3. EDT/OCC Insertion
4. ATPG
5. Scan Insertion Phases
6. Scan Structure Insertion
7. Test Mode and Scan Enable
8. Scan DRC (Design Rule Checks)
9. Violation Types and Fixes
10. Scan Chain Optimization
11. Scan Chain Reordering
12. Outputs of Scan Insertion

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



