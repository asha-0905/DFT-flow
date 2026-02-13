




















DFT-Complete-Flow/
│
├── README.md
│
├── 01_MBIST_Insertion/
│   ├── 01_wrapper_and_graybox.md
│   ├── 02_mbist_architecture_and_fault_models.md
│   ├── 03_mbist_insertion_flow.md
│   └── 04_ijtag_and_sib_architecture.md
│
├── 02_EDT_OCC_Insertion/
│   ├── 01_edt_architecture.md
│   ├── 02_occ_architecture.md
│   ├── 03_edt_insertion_flow.md
│
├── 03_Simulation/
│   ├── 01_timing_simulation_basics.md
│   ├── 02_simulation_flow.md
│   ├── 03_mismatch_debug_methodology.md
│
├── 04_ATPG/
│   ├── 01_fault_models.md
│   ├── 02_fault_classes.md
│   ├── 03_pattern_types.md
│   ├── 04_atpg_flow.md
│   ├── 05_coverage_improvement.md
│
├── 05_Scan_Insertion/
│   ├── 01_scan_basics.md
│   ├── 02_scan_drc_and_fixes.md
│   ├── 03_chain_balancing_and_merging.md
│
└── scripts/
    ├── mbist_flow.tcl
    ├── edt_flow.tcl
    ├── atpg_flow.tcl
    └── simulation.csh

------------------------------------------------------------------------------------------------


📁 01_MBIST_Insertion/
📄 01_wrapper_and_graybox.md
Wrapper Insertion and Graybox Generation
1️⃣ Why Wrapper Insertion?

ATPG process on very large and complex designs can often be unpredictable

While doing DFT at block level, we will not be checking the interconnects between 2 blocks

But the interconnects may have defects

We insert wrapper cells in order to test the interconnection between the blocks

2️⃣ Wrapper Chains

Wrapper chains are a series of scan cells connected to the boundary of the design

Wrapper chains are scan chains around the periphery of a block that connect to each input and output of the block to be tested

Wrapper cells may be of type dedicated cell or shared cell

Shared Wrapper Cells

When the PI of a block directly connects to a flop

Or connects to a flop through small combinational logic

The tool will reuse that flop as a wrapper flop (shared wrapper)

Similarly applicable for PO

Dedicated Wrapper Cells

Insert wrapper cells on input ports

Adding new wrapper cells

Dedicated wrapper chains are inserted into the scan chain configuration when you issue:

insert_test_logic


This approach will add area overhead

Why Shared Wrappers?

To overcome the area overhead increase problem

Re-using the existing flops as wrapper cells

3️⃣ Wrapper Operating Modes
INTEST Mode

All inputs to submodules are controllable using Input wrapper scan chains

All outputs are observable through Output wrapper scan chains

Input wrapper chains launch data into inside logic

Output wrapper chains capture data from inside logic

EXTEST Mode

All outputs from submodules are controllable using Output wrapper scan chains

All inputs are observable through Input wrapper scan chains

Output wrapper chains launch data into outside logic

Input wrapper chains capture data from outside logic

Tool uses input and output wrapper chains to provide test coverage of hierarchical designs during INTEST and EXTEST modes.

4️⃣ Wrapper Chain Commands
set_wrapper_analysis_options
set_wrapper_analysis_options


Sets parameters for wrapper cell analysis

Option:

##-exclude port port_spec → Excludes specified ports from wrapper analysis

##set_dedicated_wrapper_cell_options
##set_dedicated_wrapper_cell_options


Specifies how each port is handled during wrapper analysis

Default: auto

During wrapper analysis, tool checks if PI drives set/reset of flop

If yes → dedicated wrapper inferred

Lab usage:

set_dedicated_wrapper_cell_options on -ports rst


Adds dedicated wrapper for PIs driving reset ports

analyze_wrapper_chains
analyze_wrapper_chains


Identifies shared and dedicated wrapper cells for PI and PO

Works with set_dedicated_wrapper_cell_options

5️⃣ Scan Modes for Wrapped Core

For wrapped core:

add_scan_mode int_mode -edt_instances corea_rtl2_tessent_edt_c1_inst


Automatically takes configurations mentioned during EDT OCC run

add_scan_mode ext_mode -chain_length 32

analyze_scan_chains


Distributes scan elements into new scan chains

insert_test_logic


Inserts test structures and stitches scan chains

6️⃣ Graybox

Graybox is simplified representation of core module describing periphery logic

Used when only boundary logic is required

Inserted wrapper chains are included in graybox

Graybox Commands
get_config_elements Core(corea)/Scan/Mode -part tcd -silent

get_config_value

Import_scan_mode

set_attribute_value [get_ports*edt_channel*] -name ignore_for_graybox


Setting ignore_for_graybox on any pin has no effect on graybox analysis

analyze_graybox

write_design -tsdb -graybox -verbose


Mode Parameters:

mode_name: int_mode, ext_mode

mode_type: internal, external

📄 02_mbist_architecture_and_fault_models.md
MBIST Strategy and Fault Models
1️⃣ Plan MBIST Strategy
2️⃣ Memory Types

RAM → Read, Write

ROM → Read only

3️⃣ Fault Models
Single-Cell Faults
Stuck-at Fault

Cell always 0 (SA0)

Cell always 1 (SA1)

Transition Fault

Fails 0→1

Fails 1→0

Double-Cell Faults
Coupling Fault

Victim cell affected by aggressor cell.

State coupling → Aggressor state forces victim to 0/1

Inversion coupling → Aggressor transition complements victim

Idempotent coupling → Aggressor transition forces victim to 0/1

Address Decoder Faults

Possible faulty behaviors:

Given address → no cell accessed

Certain cell never accessed

Cell accessed by multiple addresses

Given address → multiple cells accessed

4️⃣ Memory Test Algorithms

Test algorithm = finite sequence of test elements.

Each test element contains:

Memory operations (Read/Write)

Data pattern (Zero/One)

Address sequence (Ascending/Descending)

📄 03_mbist_insertion_flow.md
Tessent MBIST Basic Flow
1️⃣ Set Context
set_context dft -rtl -design_id first_insertion


Creates separate directory in TSDB.

set_tsdb_output_directory ../tsdb_outdir

2️⃣ Provide Design & Library Files
read_cell_library ../../library/adk.tcelllib

read_verilog ../../library/mems/SYNC_1R1W_16x8.v -exclude_from_file_directory -verbose

read_verilog ../design/corea.v -verbose

3️⃣ Elaborate Design
Set_current_design corea

set_design_level physical_block

4️⃣ Define Clocks
add_clocks 0 clka -period 10ns

add_clocks 0 clkb -period 20ns

5️⃣ Create DFT Specification Requirements
set_dft_specification_requirements -memory_test on

6️⃣ Add DFT Signals
add_dft_signal <signal_name>


Inserts TDR logic for static signals

Static signals:

Tck_occ_en

Ltest_en

Memory_bypass_en

Dynamic signals:

Add_dft_signal <signal name> -ource node <top_level_port_name>

7️⃣ Run DRC
check_design_rules

DFT_C1 Violation

Clock not declared for memory

8️⃣ Create DFT Specification
Create_dft_specification


Uses:

set_design_level

Design netlist

9️⃣ Process DFT Specification
process_dft_specification


Performs:

Validates DFT spec

Generates MBIST controllers

Generates memory interfaces

Generates IJTAG network (RTL + ICL)

Inserts hardware

Generates SDC constraints

Writes files into TSDB

🔟 Pattern Specification
Create_patterns_speification

process_patterns_verification

📄 04_ijtag_and_sib_architecture.md
IJTAG and SIB Architecture
1️⃣ DFT Specification Contents

IJTAG network configuration

Memory BIST partitioning/configuration

Memory BIST Partitioning

Listing MBIST controllers

Clock domain per controller

Memories assigned per controller

2️⃣ SIB (Segment Insertion Bit)

A SIB is a special node in JTAG acting as a switch.

Types of SIBs
STI (Scan Tested Instrument)

Provides IJTAG access for MBIST controller

SRI (Scan Resource Instrument)

Provides IJTAG access for logic instruments (EDT, OCC)

3️⃣ Instrument Organization

Instruments active during scan (EDT/OCC) → under one SIB

Instruments scan tested (MBIST controller) → under another SIB

4️⃣ ICL Extraction

Automated generation of IJTAG interconnection info

ICL files created during process_dft_specification

Verifies connectivity of ICL modules

Must pass with no violations before generating patterns




---------------------------------------------------

📁 02_EDT_OCC_Insertion/
📄 01_edt_architecture.md
EDT Architecture (Embedded Deterministic Test)
1️⃣ Scan Compression Overview

Add compression hardware (EDT – Embedded Deterministic Test)

Reduces number of external scan channels

Improves test efficiency

2️⃣ Decompressor Architecture
LFSR (Linear Feedback Shift Register)

Given an initial value, the LFSR starts generating patterns

Disadvantage: Generated patterns have a diagonal relationship

Phase Shifter

Patterns from LFSR are given to Phase Shifter

Adds randomness to patterns

Reduces correlation between generated scan data

External Channels

Even after Phase Shifter, tool may not generate all combinations

Patterns are also provided from external channels

Increases randomness further

3️⃣ Compressor and Mask Logic
Compressor

Built using levels of XORs

Compresses values of all internal scan chains into a single value

If black boxes are encountered → propagates X

Mask Logic

Added to stop propagation of X

Masks X propagation using decoders

Decoders

Without decoders → size and area increase.

Types:

XOR Decoder

One-hot Decoder

4️⃣ Compression Relationship

Relationship between external channels and internal scan chains:

Compression Ratio = Number of internal chains / Number of external channels

📄 02_occ_architecture.md
OCC Architecture (On-Chip Controller)
1️⃣ Purpose of OCC

Circuit that decides:

Which clock propagates in Functional mode

Which clock propagates in DFT mode (Shift and Capture)

Number of clock pulses required during Capture phase

OCC controls clock behavior during:

Functional mode

Shift phase

Capture phase

📄 03_edt_insertion_flow.md
EDT/OCC Insertion Flow
1️⃣ Set Context to DFT
2️⃣ SETUP Phase
Read Design Files
read_verilog


(Scan inserted netlist)

read_cell_library


(Library file)

Elaboration

Elaborate the design

Dofile

Provide ATPG dofile

Internally testproc will be called

Specify Number of Channels
set_edt_options

3️⃣ ANALYSIS Phase
Check DRC Violations
analyze_drc_violations

Common EDT Violations

K13 → Information regarding newly inserted pins

E5 → X-propagation violation

Fix EDT-related violations.

4️⃣ INSERTION Phase
write_edt_files


Performs insertion

Writes output files

Auto-Generated Script

Complete process script automatically written by tool:

dc_script.scr

5️⃣ Library Concepts
Link Library

Used to link a design

Resolves instantiated references in RTL or gate-level netlist

Target Library

Used during compile command

Creates technology node specific gate-level netlist

DC optimization selects smallest gates meeting timing and functionality

6️⃣ Outputs

Compressed scan netlist:

edt_top_gate.v


EDT/OCC configuration


---------------------------------------------------------------------------------------

📂 03_Simulation/01_timing_simulation_basics.md
Timing Simulation Basics
1️⃣ Purpose of Simulation

Simulation is done:

To validate the patterns generated during ATPG

To ensure patterns work correctly before sending them to tester

2️⃣ Need for Timing Simulation

During ATPG:

Timing (cell delays and net delays) is not considered

Patterns are generated without delay information

Before giving patterns to the tester:

We must verify them with actual delays

Hence, timing simulation is required

3️⃣ PD Netlist and Timing Data

Flow:

PD Netlist → STA → Timing Reports + SDF


STA generates timing reports

SDF (Standard Delay Format) file is generated

4️⃣ SDF (Standard Delay Format)

SDF contains:

Delay of each and every cell

Delay of each and every net

In QuestaSim:

When SDF file is read

Tool maps delays to design

This mapping is called:

Annotation

Annotation = Mapping SDF delays to design cells and nets.

5️⃣ Simulation vs Tester

After manufacturing:

Tester (ATE) applies patterns on the design

During simulation:

Testbench is used

Testbench mimics the tester

If:

Simulated output = Expected output → Patterns are correct

Simulated output ≠ Expected output → Simulation mismatch

6️⃣ Serial vs Parallel Simulation
Serial Simulation

Patterns loaded serially into scan chain

Parallel Simulation

Patterns loaded parallelly into scan chain

📂 03_Simulation/02_simulation_flow.md
Simulation Flow
1️⃣ Compilation

Tool checks for syntax errors.

Command:

vlog pattern_serial_sample.v -f Verilog_files.list +nospecify -override 1ns/1ps -work work -l compile.log


Tool takes:

Library

Netlist

Testbench

2️⃣ Elaboration

Elaborate design with respect to top module.

Command:

vsim

3️⃣ Simulation

Apply stimuli to DUT.

Command:

vsim -c debugDB -voptargs=+acc DmaWr_pattern_serial_sample_v_ctl -do “add wave -r /*; run -all” -c +nospecify -l simulation.log -wlf wave.wlf

4️⃣ Pattern Files Used
Pattern File
Pattern_serial_sample.v.0.vec


Vector file

Contains pattern data

PO Name File
Pattern_serial_sample.v.po.name


Contains:

PO names

edt_channel_out names

Comparison between:

Simulated value

Expected value

is done at all these locations.

5️⃣ Testbench Role

The above two files are called inside:

pattern_serial_sample.v


Testbench:

Applies pattern to design

Mimics tester behavior

6️⃣ Running Simulation Script

To source simulation script:

source simulation.csh

7️⃣ Opening Waveform

Waveform file:

wave.wlf


Command:

vsim wave.wlf

8️⃣ Reasons for Simulation Mismatches

Internal cut points

Hold violation in Q-SI path not fixed

False path (SDC not read during ATPG TDF run)

Wrong database (pattern vs netlist release mismatch)

Library file mismatch

📂 03_Simulation/03_mismatch_debug_methodology.md
Simulation Mismatch Debug Methodology
1️⃣ Creating a Mismatch Scenario (For Practice)

To practice debugging:

Use:

case1_edt_top_gate.v


Generate patterns

Write all patterns

Do not use -begin and -end switch in write_pattern

Go to edt outputs directory

Create copy of:

case1_edt_top_gate.v


Rename it to:

simulation_mismatch.v


Modify design:

Ground A input of u8 AND gate

This creates a mismatch scenario.

2️⃣ Why Use Parallel Simulation for Debug?

For debugging mismatches:

Use parallel simulation

We can exactly identify which flop is failing

3️⃣ Steps for Simulation Mismatch Debug
Step 1: Check Simulation Log

In simulation log:

Identify failing flop

Identify failing pattern number

Identify timestamp of failure

Step 2: Open Waveform

Open QuestaSim waveform

Load failing flop waveform

Go to failure timestamp

Step 3: Open ATPG Session

In ATPG:

Open same flop instance

Load failure pattern

Menu:

Data → Pattern Index → Failing pattern number

Step 4: Trace Back D Input

In ATPG session:

Trace back D input of failing flop

In QuestaSim:

Open same instance

Compare values

Repeat tracing process until:

Root cause is found

Step 5: Fix Root Cause

After identifying:

Root cause of failure

Fix the issue

Debug Summary
Step	Action
1	Identify failing flop, pattern, time
2	Open waveform and inspect
3	Load failing pattern in ATPG
4	Trace D input backward
5	Identify and fix root cause

------------------------------------------------------------------------

📂 04_ATPG/01_fault_models.md
Fault Models

ATPG generates patterns based on fault models.

1️⃣ Stuck-at Fault Model (SA)

Detects nodes stuck at 0 or 1

Most basic and widely used model

Used for coverage analysis and improvement

2️⃣ Transition Delay Fault Model (TDF)

Used to detect whether logic transitions:

0 → 1

1 → 0

occur within the time period of fast frequency clock.

Why Fast Frequency Clock?

After manufacturing:

Chips operate at fast functional frequency

Need to verify rise/fall occurs within fast clock period

Capture Requirement

In capture phase:

2 clock pulses required

Launch

Capture

Ways to Detect TDF
LOC (Launch On Capture)

Launch on capture

Launch off capture

LOS (Launch On Shift)

Launch on shift

Launch off shift

LOC vs LOS

LOS has slightly more coverage than LOC

Tool has more controllability in LOS

Explanation:

In LOS, tool brings transition value through shift path

In LOC, tool must program combinational logic to create transition

If tool cannot generate proper value at combo output → fault not detected.

LOC requires more patterns than LOS

Limitation of LOS:

Controlling SE (timing closure on SE) is extremely difficult

Therefore:

Mostly LOC is used

Pipelined Scan Enable

In OCC:

When SEN = 1

Slow frequency clock is propagated for both SA and TDF

Using LOS:

Fast frequency clock propagated using pipelined scan enable

Why TDF Coverage < SA Coverage

Reason 1:

If one input of AND gate tied to ground

No transition faults detected

In SA, 2 faults detectable

Reason 2:

Timing exceptions

Timing Exceptions
False Path (AU.FP)

Faults untestable due to false path

Declared in SDC file

False path example:

Asynchronous clocks CLKA and CLKB

Declared false path

Timing not closed

For SA:

Tool handles this

Multicycle Path (AU.MCP)

Requires more than one clock cycle

Exception to default single-cycle timing

Example:

Designer wants setup check at cycle 4 instead of cycle 1

3️⃣ IDDQ

Fault model based on quiescent current

4️⃣ PDF

Path Delay Fault model

5️⃣ Fault Categories
Fault Dominance

Fault F dominates fault G if:

Detecting set of F contains that of G

Fault Equivalence

Faults F1 and F2 are equivalent if:

All tests detecting F1 detect F2

And vice versa

Fault Collapse

Removing equivalent faults from total fault list.

Fault Aliasing

A fault is aliased when:

Observed by even number of scan cells

Located in different scan chains

Compacted to same output channel

📂 04_ATPG/02_fault_classes.md
Fault Classes

Tool categorizes faults based on:

How detected

Why not detected

1️⃣ Untestable (UT)

No pattern can exist to detect them.

These faults:

Cannot cause functional failures

Excluded when calculating test coverage

Subclasses of UT
UU – Unused

Faults on circuitry unconnected to observation point

Faults on floating primary outputs

TI – Tied

Fault location tied to same value as stuck value

BL – Blocked

Tied logic blocks path to observation point

RE – Redundant

Proven undetectable after exhaustive analysis

2️⃣ Testable Faults

All faults that cannot be proven untestable.

Detected (DT)

Faults identified as detected.

Subgroups:
DS – det_simulation

Detected during fault simulation

DI – det_implication

Detected during learning analysis

Subclasses of DI:

scan_path

scan_enable

clock

set_reset

Posdet (PD)

Possible-detected faults:

Good machine = 0/1

Faulty machine = X

Hard-detected:

Binary difference

ATPG_Untestable (AU)

Testable faults not detected due to tool constraints.

AU Subclasses

AU.BB – Black Boxes

AU.PC – Pin Constraints

AU.TC – Tied Cells

AU.CC – Cell Constraints

AU.UDN – Undriven

AU.WIRE – Wire Contention

AU.SEQ – Sequential Depth

AU.EDT – EDT Blocks

AU.IJTAG – IJTAG instruments

AU.OCC – On Chip Clock Control

AU.MPO – Masked POs

Sequential depth can be increased using:

set_pattern_type -sequential

Undetected (UD)
UC – Uncontrolled
UO – Unobserved
UD.AAB – ATPG Abort

Increase abort limit:

set_abort_limit

UD.UNS – Unsuccess
UD.EAB – EDT Abort
Fault Coverage

Fault Coverage =
Detected faults / Total faults

Test Coverage =
Detectable faults / Testable faults

Test coverage > Fault coverage

We focus on improving test coverage.

📂 04_ATPG/03_pattern_types.md
Pattern Types
1️⃣ Serial Patterns

Loaded serially

Shift cycles = number of flops in chain (without EDT)

Serial without EDT:

Shift cycles =
Number of flops + masking bits + initialization cycles

Capture:

1 clock pulse

Procedure:

Force SI

Measure SO

Pulse clock

2️⃣ Parallel Patterns

Loaded parallelly

Only 1 shift clock required

Procedure:

Sense scan cell

Force parallel data

Pulse clock

Important:

Measure output first

Because ‘force’ overwrites previous flop value

Why Both Needed?
Parallel Patterns

Less runtime

Easy debug

Validate all patterns before manufacturing

Serial Patterns

Tester uses serial patterns

Validate sample before going to tester

📂 04_ATPG/04_atpg_flow.md
ATPG Flow
SETUP → ANALYSIS → Coverage Improvement → Finalize Patterns

1️⃣ SETUP Phase

Read input files and configure.

For EDT patterns:

edt_top_gate.v

edt.dofile

internally calls edt_testproc

For bypass patterns:

edt_top_gate.v

bypass.dofile

internally calls bypass.testproc

2️⃣ ANALYSIS Phase

Check DRC

Generate patterns

Get coverage

Write output patterns

📂 04_ATPG/05_coverage_improvement.md
Coverage Improvement (SA Model)
AU.PC

Step 1:

write_faults <file> -class AU.PC -replace


Step 2:

analyze_fault <pin hierarchy> -stuck 0/1 -D


Step 3:

Visualize and give data

Fix:

Add new flop

AU.SEQ

Step 1:

write_faults <file> -class AU.SEQ -replace


Step 2:

analyze_fault <pin hierarchy> -stuck 0/1 -D


Fix:

Increase sequential depth

AU.WIRE

Step 1:

write_faults <file> -class AU.WIRE -replace


Step 2:

analyze_fault <pin hierarchy> -stuck 0/1 -D


Fix:

Ensure wire driven by single logic

--------------------------------------------------------------------

📂 05_Scan_Insertion/01_scan_basics.md
Scan Insertion Basics
1️⃣ Invoking Scan Insertion Flow
Source Mentor Environment
source /home/cshrc

Invoke Tessent Shell
/home/TESSENT/bin/tessent -shell

2️⃣ Input Files Required

Design input file (Gate-level netlist)

Library file
(Contains functionality of each and every cell in design)

3️⃣ Files Created by Us

Dofile

Stores instructions/commands

Log file

Stores tool execution details

4️⃣ Scan Insertion Phases
SETUP → ANALYSIS → INSERTION

5️⃣ SETUP Phase

Set context (scan/edt)

Read design files

Read library files

Elaboration

Fix DRC violations after analysis

6️⃣ ANALYSIS Phase

Check DRC violations

Analyze violations

Declare number of scan chains

7️⃣ INSERTION Phase

Replace normal flops with scan flops

Insert scan structures

Add scan mux

Build scan chains

Generate reports and output files

8️⃣ Scan Structure Details
Replace Flops with Scan Cells

Normal flop → Scan flop

Add Mux to Flop

Mux inputs:

Functional input (0)

Scan input (1)

Mux output → Flop D input

Build Scan Chains

All scan flops stitched together

Form shift and capture paths

9️⃣ Test Mode and Scan Enable
Test Mode (TM)

TM = 0 → Functional mode

TM = 1 → Test mode

Scan Enable (SE)

SE = 1 → Shift phase

SE = 0 → Capture phase

🔟 Outputs Generated

Scan inserted netlist

ATPG Dofile

ATPG testproc

Scandef file

Scan cell report

Scan chain report

Scan DRC report

Scan Inserted Netlist

Normal flops replaced with scan flops

Scan chains stitched

New ports added:

scan_en

scan_in

scan_out

Scan Chain Report

Contains:

Number of scan chains

Information about each chain

Scan Cell Report

Contains:

Scan cell details

📂 05_Scan_Insertion/02_scan_drc_and_fixes.md
Scan DRC and Fixes
DRC (Design Rule Checks)

Rules that a flop must pass to convert into scan flop.

1️⃣ Clock Controllability DRC

Issue:

Clock definition missing

Clock connected to another flop

Fix:

add_clock <off state> <clock_name>


Or add mux.

2️⃣ Set/Reset Controllability DRC

Issue:

Set/Reset connected to flop/combo output

Tied to ground

Missing declaration

Fix:

add_pin_constraints <off state C0/C1> <set/reset name>


Or

add_clock <off state> <set/reset name>


Or add mux.

3️⃣ Bus Contention

Occurs when:

Two or more drivers force opposite values

Fix:

set_test_logic -tristate on

4️⃣ Feedback Loop DRC

Issue:

Oscillating feedback

Tool cannot determine value

Fix:

Add mux

5️⃣ X-Source DRC

Issue:

Output is X

Due to analog block or memories

Fix:

Bypass logic

6️⃣ Potential Race DRC
Clock Data Race

Clock drives both Clock and D port

Set Reset Race

Same signal drives Set and Reset

Fix:

Add mux

Specific Violation Codes
S1

Clock or set/reset missing or improperly connected.

Fix:

add_clock <off state> <clock_name>

S2

Clock tied to fixed value.

Fix:

Add mux

D5

S1/S2 violation

Scan equivalent not available

Latch present

Fix:

Declare clock

C6

Clock pin connected to D input and another flop’s clock.

Fix:

Add mux

Or:

set_test_logic -C6 on

C3

Positive edge flop connected to negative edge flop.

Fix:

Add lockup latch

C7

Flop cannot capture.

Fix:

Add mux

Manual and Autofix Options

Manual netlist editing using:

create_connection

delete_connection

create_instance

Autofix commands:

set_test_logic -clock on
set_test_logic -set on
set_test_logic -reset on
set_test_logic -tristate on
set_test_logic -C6 on

📂 05_Scan_Insertion/03_chain_balancing_and_merging.md
Scan Chain Balancing and Merging
1️⃣ Scan Chain Balancing

Goal:

Keep almost equal number of flops per chain

Reason:

Unbalanced chains increase test time

Fix:

Add dummy values to shorter chains

2️⃣ Edge and Clock Domain Mixing

Used for balancing.

Edge Mixing

Positive and negative edge triggered flops in same chain

Clock Domain Mixing

Flops from different clock domains in same chain

Command
insert_test_logic -number <num_chains> -edge merge -clock merge

3️⃣ Scan Chain Reordering

Done by PD team

Uses scandef file

Final Scan Outputs Summary

After complete scan insertion:

Scan inserted netlist

Scan cell report

Scan chain report

Scan DRC report

Scandef file


---------------------------------------------------------------------------------------------------
===================================================================================================

01_MBIST_Insertion/
01_wrapper_and_graybox.md
Wrapper Insertion and Graybox Generation
Why Wrapper Insertion?

ATPG process on very large and complex designs can often be unpredictable

While doing DFT at block level, we will not be checking the interconnects between 2 blocks

But the interconnects may have defects

We insert wrapper cells in order to test the interconnection between the blocks

Wrapper Chains

Wrapper chains are a series of scan cells connected to the boundary of the design

Wrapper chains are scan chains around the periphery of a block that connect to each input and output of the block to be tested

Wrapper cells may be of type dedicated cell or shared cell

Shared Wrapper Cells

When the PI of a block directly connects to a flop

Or connects to a flop through small combinational logic

The tool will reuse that flop as a wrapper flop

Similarly applicable for PO

Dedicated Wrapper Cells

Insert wrapper cells on input ports

Adding new wrapper cells

Dedicated wrapper chains are inserted into the scan chain configuration

insert_test_logic


This approach will add area overhead

Why Shared Wrappers?

To overcome the area overhead increase problem

Re-using the existing flops as wrapper cells

INTEST Mode

All inputs to submodules are controllable using Input wrapper scan chains

All outputs are observable through Output wrapper scan chains

Input wrapper chains launch data into inside logic

Output wrapper chains capture data from inside logic

EXTEST Mode

All outputs from submodules are controllable using Output wrapper scan chains

All inputs are observable through Input wrapper scan chains

Output wrapper chains launch data into outside logic

Input wrapper chains capture data from outside logic

Tool uses input and output wrapper chains to provide test coverage of hierarchical designs during INTEST and EXTEST modes.

Wrapper Chain Commands
set_wrapper_analysis_options


-exclude port port_spec

set_dedicated_wrapper_cell_options


Default: auto

set_dedicated_wrapper_cell_options on -ports rst

analyze_wrapper_chains

Scan Modes for Wrapped Core
add_scan_mode int_mode -edt_instances corea_rtl2_tessent_edt_c1_inst

add_scan_mode ext_mode -chain_length 32

analyze_scan_chains
insert_test_logic

Graybox

Graybox is simplified representation of core module describing periphery logic

Used when only boundary logic is required

Inserted wrapper chains of the block will be included in the graybox

get_config_elements Core(corea)/Scan/Mode -part tcd -silent
get_config_value
Import_scan_mode
set_attribute_value [get_portsedt_channel] -name ignore_for_graybox
analyze_graybox
write_design -tsdb -graybox -verbose

mode_name	mode_type
int_mode	internal
ext_mode	external
02_mbist_architecture_and_fault_models.md
Memory Types
Memory	Operation
RAM	Read, Write
ROM	Read only

