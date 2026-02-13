

1. MBIST planning and insertion
•	Wrapper insertion and Graybox generation
•	Wrapper insertion 
	ATPG process on very large and complex designs can often be unpredictable 
	While doing DFT at block level, we will not be checking the interconnects between 2 blocks
	But the interconnects may have defects
	We insert wrapper cells in order to test the interconnection between the blocks
•	Wrapper chains
	Wrapper chains are a series of scan cells connected to the boundary of the design
	Wrapper chains are scan chains around the periphery of a block that connect to each input and output of the block to be tested
	Wrapper cells may be of type dedicated cell or shared cell
	When the PI of a block directly connects to a flop or if it connects to a flop through a small combo logic, then the tool will reuse that flop as a wrapper flop (shared wrapper). Similarly for PO
	Dedicated Wrappers: 
	Insert wrapper cells on input ports 
	Adding new wrapper cells
	Dedicated wrapper chains are inserted into the scan chain configuration when you issue the “insert_test_logic” command
	This approach will add area overhead
	Shared Wrappers: 
	To overcome the area overhead increase problem, we go for shared wrappers
	Re-using the existing flops as wrapper cells 
	INTEST mode: In INTEST mode, all inputs to submodules are controllable using the Input wrapper scan chains and all outputs are observable through the Output wrapper scan chains
	Input wrapper chains launch data into the inside logic and output wrapper chains capture data from inside logic
	EXTEXT mode: In EXTEST mode, all outputs from submodules are controllable using the output wrapper scan chains and all inputs are observable through the Input wrapper scan chains
	Output wrapper chains launch data into the outside logic and input wrapper chains capture data from outside logic
	Tools uses the input and Output wrapper chains to provide test coverage of hierarchical designs during INTEST and EXTEST modes
	Wrapper chain Commands:
	Command: set_wrapper_analysis_options
o	This command sets up parameters for the wrapper cells analysis
o	One of the parameters is -exclude port port_spec which specifies that all ports in the port_spec will be excluded from wrapper analysis
	Command: set_dedicated_wrapper_cell_options
o	This command specifies how each port in the current design is to be handled during wrapper analysis
o	By default, this command is set to auto (dedicated wrappers cells are inferred and determined by the tool)
o	During wrapper analysis, the tool will check whether a PI is driving the set/reset ports of a flop. If yes, it will infer a dedicated wrapper on this PI
o	In my lab, we gave set this command to on and we are asking the tool to add a dedicated wrapper for the PIs that are driving reset ports of a flop
o	set_dedicated_wrapper_cell_options on -ports rst
	Command: analyze_wrapper_chains 
o	This command identifies shared and dedicated wrapper cells for PI and PO
o	When used in conjunction with set_dedicated_wrapper_cell_options command, the analysis will be resolved and the registration of the PI and PO ports with dedicated wrappers that have failed the shared wrapper identification will be done
	Scan Insertion:
	For a wrapped core, we can use internal mode and external mode
	add_scan_mode int_mode -edt_instances corea_rtl2_tessent_edt_c1_inst (from EDT inserted netlist)
	it will automatically take the configurations which we have mentioned during the EDT OCC run
	add_scan_mode ext_mode -chain_length 32
	analyze_scan_chains: distributes scan elements into new scan chains
	insert_test_logic: inserts the test structures and stitches up the scan chain to increase the design’s testability
	Graybox:
	Graybox is a simplified representation of the core module describing its periphery logic
	The graybox model is used in place of full core netlist in any situation in which only the logic at the boundary of the core is needed
	The inserted wrapper chains of the block will be included in the graybox
	Graybox generation commands
o	get_config_elements Core(corea)/Scan/Mode -part tcd -silent
get_config_elements: returns a collection of configuration elements. We will be getting it from our TCD file for scan
o	get_config_value: Returns a value associated with a configuration element based on the specified option
o	Import_scan_mode: Imports configuration settings from tcd_scan
o	set_attribute_value [get_ports*edt_channel*] -name ignore_for_graybox: sets an attribute’s value
ignore_for_graybox: Setting this attribute on any pin of a module has no effect on graybox analysis
o	analyze_graybox: Identifies the instances and nets to be included in the graybox netlist
o	write_design -tsdb -graybox -verbose: Write the graybox netlist into tsdb output directory
mode_name : int_mode, ext_mode
mode_type : internal, external


•	Plan MBIST strategy
•	Memory types
•	RAM: Read, Write
•	ROM: Read only
•	Fault models (single cell, double cell)
•	Single-cell 
	Stuck-at fault: A cell is always 0 (Stuck at 0) or it is always 1 (Stuck at 1)
	Transition fault: A cell fails to go from 0 to 1 transition or 1 to 0 when it is written
•	Double-cell
	Coupling fault: Victim cell is affected by the aggressor cell. The value in the aggressor cell can affect the value in the victim cell.
o	State coupling: If aggressor cell is in given state, the victim is forced to 0 or 1
o	Inversion coupling: If aggressor cell rise/fall, the victim is complemented
o	Idempotent coupling: If aggressor cell rise/fall, the victim is forced to 0 or 1
	Address decoder faults: It is caused by AND or OR types of faults. It can lead to four faulty behaviours
o	Given a certain address, no cell will be accessed
o	A certain cell is never accessed by any address
o	A certain cell can be accessed by multiple address
o	Given a certain address, multiple cells are accessed
•	Memory test algorithms
•	Test algorithm is a finite sequence of test elements. Test elements contains number of
	Memory operations [Read/Write]
	Data pattern [Zero/One]
	Adress sequence [Ascending/Descending]
•	Tessent MBIST basic flow
•	Load design and MBIST requirements
•	Provide memory, library, and design files
•	Set context, define clocks for memories
•	Set memory test requirements and run design rule checks
•	Context “dft rtl” tell the tool to enter into RTL insertion mode, design_id tells to create a separate directory in tsdb for this particular below insertion
set_context dft -rtl -design_id first_insertion
•	Specify the output directory where you want to dump the outputs of first insertion
set_tsdb_output_directory ../tsdb_outdir
•	Provide the design files and library files
read_cell_library ../../library/adk.tcelllib
read_verilog ../../library/mems/SYNC_1R1W_16x8.v -exclude_from_file_directory -verbose
read_verilog ../design/corea.v -verbose
•	Elaborate the design
Set_current_design corea
•	Set the design level to either chip, physical_block or sub_block
set_design_level physical_block
add_clocks 0 clka -period 10ns

add_clocks 0 clkb -period 20ns
1. DFT specification and IJTAG network
•	Create DFT specification
•	Describe MBIST, EDT/OCC
•	Define IJTAG network and MBIST partitioning
•	Specifies the requirements to be checked during check_design_rules
•	set_dft_specification_requirements -memory_test on (“-memory_test on” option is needed when implementing memory test)
•	adding DFT signal
•	add_dft_signal <signal_name> (It will insert TDR logic for specified signal)
•	TDR can be inserted only for static signals (like TE)
•	Tck_occ_en: A global DFT signal that is used to enable the mini-OCC present inside the sib(sti) node
•	Ltest_en: A logic test control signal that is used to enable the logic test mode. This signal is force high during all logic test modes
•	Memory_bypass_en: To bypass memories. This signal is set to 1 by default during logic test
•	For dynamic signals, we can’t insert TDR. So, it has to be controlled from the top level
•	Add_dft_signal <signal name> -ource node <top_level_port_name>
•	DRCs Check: Once everything is defined check_design_rules command will run a DRC check on the current design
•	DFT_C1 violation: When we have not declared the clock that is going to the memory
•	DFT Specification: The DFT specification describes the test hardware that will be added to your design
	IJTAG Network configuration
	Memory BIST portioning/configuration
o	Memory BIST portioning:
	Listing of Memory BIST controllers to be generated 
	Clock domain associated with each memory BIST controller
	Memories assigned to each controller
•	Creating a New DFT Specification: A new DFT Specification can be created using the command 
•	Create_dft_specification
•	This command uses information from prior settings:
	Set_design_level
	Design netlist etc
•	A SIB is a special node in JTAG that acts as a switch
•	Instruments that need to be active during scan (EDT OCC) are inserted under 1 SIB and the ones that are scan tested such as MBIST controller are inserted under another SIB
•	Types of SIBS 
	Scan Tested Instrument (STI): The SIB STI provides access to the IJTAG network for MBIST controller
	Scan Resource Instrument (SRI): The SIB SRI provides access to the IJTAG network for logic instruments (EDT, OCC) 
•	Generating and Inserting the Hardware 
	process_dft_specification
	Validates the DFT Specification
	Generates hardware: MBIST related controllers, memory interfaces, IJTAG network (RTL and ICL descriptions)
	Edits design and inserts generated hardware
	Generates SDC constraints
	Generated files are written into TSDB directory
•	Add DFT signals
•	Static test signals
•	Dynamic signals sourced from top level ports
•	Generate and validate DFT spec
•	View configuration data
•	Define instruments
•	SIB (STI): MBIST instruments
•	SIB (SRI): Scan/EDT/OCC instruments
•	Process DFT spec
•	Insert MBIST controllers and memory interfaces
•	Generate IJTAG (ICL + RTL)
•	

4. Pattern specification and ATPG
•	Extract ICL for instrument network
•	ICL Extraction Process: 
	Automated generation of the interconnection information of the various IJTAG building blocks (SIBs, TDRs etc)
	Tessent Instrument ICL files are created during process_dft_specification
	Extract ICL process verifies the proper connectivity of the ICL modules that were inserted during the process_dft_specification command
	ICL extraction must pass with no violations in order to generate the test patterns
•	Create pattern specification
•	Default pattern spec object
•	Validate pattern spec
Create_patterns_speification is used to generate a pattern specification
Validation and processing of the specification is done using the process_patterns_verification
•	ATPG pattern generation
•	Fault Classes: The tool categorizes faults into fault classes, based on how the faults were detected or why they could not be detected
	Untestable (UT): faults for which no pattern can exist to either detect or possible-detect them. Untestable faults cannot cause functional failures, so the tool exclude them when calculating test coverage
	Unused (UU): The unused fault class includes all faults on circuitry unconnected to any circuit observation point and faults on floating primary outputs
	Tied (TI): The tied fault class includes faults on gates where the point of the fault is tied to a value identical to the fault stuck value.
	Blocked (BL): The blocked fault class includes faults on circuitry for which tied logic blocks all paths to an observable point
	Redundant (RE): The redundant fault class includes faults in the test generator consider undetectable. After the test pattern generator exhausts all patterns, it performs a special analysis to verify that the fault is undetectable under any condition
	Untestable (UT): Testable faults are all those faults that cannot be proven untestable 
	Detected (DT): The detected fault class includes all faults that the ATPG process identifies as detected. The detected fault class contains two groups:
	det_simulation (DS): faults detected when the tool performs fault simulation
	det_implication (DI): faults detected when the tool performs learning analysis. Sub classes of DI faults:
scan_path: DI faults that are directly in the scan path
scan_enable: DI faults that can propagate to and disrupt scan shifting
clock: DI faults that are in clock cone of state elements
set_reset: DI faults that are in the set/reset cone of the state elements
	Posdet (PD): The posdet, or possible-detected fault class includes all faults that fault simulation identifies as possible-detected but not hard detected. A possible-detected fault results from a good-machine simulation observing 0 or 1 and the faulty machine observing X. a hard-detected fault results from binary (not X) differences between the good and faulty machine simulations
	ATPG_untestable (AU): Testable faults become ATPG_untestable faults because of constraints, or limitations, placed on the ATPG tool (such as a pin constraint or an insufficient sequential depth). These faults may be possible-detectable, or detectable, if you remove some constraint, or change some limitation, on the test generator (such as removing pin constraint or changing the sequential depth). Sub classes of ATPG untestable (AU)
	AU.BB – BLACK_BOXES: These are faults that are untestable due to a black box, which includes faults that need to be propagated through a black box to reach an observation point, as well as faults whose control or observation requires values from the output(s) of a black box 
	AU.PC – PIN_CONSTRAINTS: These are faults that are uncontrollable or that cannot be propagated to an observation point, in the presence of a constraint value. That is, because the tool cannot toggle the pin, the tool cannot test the fanout. The only possible solution is to evaluate whether you really need the input constraint, and if not, to remove the constraint
	AU.TC – TIED_CELLS: These are faults associated with cells that are always 0 (TIE 0) or 1 (TIE 1) or X (TIE X) during capture. One example is test data register that are loaded during test_setup and then constrained so as to preserve that value during the rest of scan test
	AU.CC – CELL_CONSTRAINTS: If some faults are not getting detected because some person has added some cell constraints, we need to discuss it with the person and check whether it is a necessary cell constraint or not
	AU.UDN – UNDRIVEN: These are faults that cannot be tested due to undriven input pins. For the purpose of this analysis, undriven input pins include inputs driven by X values
	AU.WIRE – WIRE: This normally occurs when there is contention (2  or more inputs are given to a single wire)
	AU.SEQ – SEQUENTIAL_DEPTH: Sequential depth refers to the number of clock cycles required in order to capture the fault. Increasing the sequential depth means increasing the number of capture pulses. 
set_pattern_type -sequential (enables the tool to create clock sequential patterns with up to two clock pulses between scan loads)
	AU.EDT – EDT_BLOCKS: These are faults inside an instance identified as an EDT instance
	AU.IJTAG – These are faults located within the IJTAG instruments
	AU.OCC – ON_CHIP_CLOCK_CONTROL: These are faults located within the On-Chip Clock Controller (OCC). These faults (faults in the test logic) will be inherently detected by the tool.
	AU.MPO – These are faults that exist in the fanin cone of the masked POs and have no path to other observation points. PO will be masked because after manufacturing, chips will be tested parallelly on the tester and the number of pins on the tester will be limited
	Undetected (UD):
	Uncontrolled (UC): Undetected faults, which during pattern simulation, never achieve the value the value at the point of the fault detection – that is, they are uncontrollable
	Unobserved (UO): Faults whose effects do not propagate to an observable point
	UD.AAB – ATPG_ABORT: these are faults that are undetected because the tool reached its abort limit. You can raise the abort limit to increase coverage using the set_abort_limit command
	UD.UNS – UNSUCCESS: These are faults that are undetected for unknown reasons. There is nothing you can do about faults in this sub-class
	UD.EAB – EDT_ABORT: If your design’s chain:channel ratio is insufficient to detect the fault
	Fault Coverage: It is a measure of percentage of faults that can be detected by the generated test patterns out of the total number of faults that could exist in a circuit
	Fault coverage = total number of faults detected / total faults in a circuit (Where total faults includes testable faults and untestable faults)
	Test coverage = total number of faults detectable / total faults testable
	Test coverage will be more compared to fault coverage as in test coverage untestable faults are not included
	We will mainly try to improve the test coverage because untestable faults will not aaffect the functionality of the design
•	Fault models: SA, TDF, IDDQ, PDF
•	Fault categories: dominance, equivalence, collapse
	Fault Dominance: Fault F dominates fault G if detecting set of F contains that of G
	Fault Equivalence: Two faults F1 and F2 are equivalent if all tests that detect F1 also detect F2 and vice versa
	Fault Collapse: Removing equivalent faults from the total fault list is called fault collapsing
Fault Aliasing: A fault is aliased when it is observed by an even number of scan cells that happened to line up at the same location in different scan chains that are compacted to the same output channel
•	Pattern types:
	Serial Patterns:
	For serial patterns, the number of clock cycles required for each pattern, during shift cycle is same as the number of flops in that chain (Without EDT).
	Serial patterns without EDT: number of cycles in shift phase = number of flops   in scan chains + masking bits + initialization cycles
	For capture, 1 clock pulse is required
	The output is measured and compared against the expected value (ATPG tool while writing patterns will give the input sequence + expected output) in the shift out phase at the scan out port
	Procedure for serial pattern:
1.	Force si
2.	Measure so
3.	Pulse clock
	Parallel Patterns:
	For parallel patterns, for each pattern only 1 clock pulse is required for shift as patterns are loaded parallelly
	Output is measured and compared with the expected value at the output of each flop
1.	Sense scan cell
2.	Force parallel data 
3.	Pulse clock
	We measure output first in parallel simulation because, in parallel simulation, Verilog keyword ‘force’ is used to load the data parallelly. If ‘force’ is used, then the net will automatically get the value which is forced. So, the previous output of the flop will get overwritten with the new value that is forced. So, in parallel simulation, we first sense the scan cell and then force parallel data 
	We need both serial and parallel pattern:
	Parallel patterns:
	Run time is less. So, we validate all the patterns and only a sample of serial patterns before manufacturing
	Easy to debug
	serial patterns:
	In tester serial patterns will be given. So, before going to the tester, a sample of serial patterns will be validated

•	ATPG flow
•	SETUP → ANALYSIS → coverage improvement → finalize patterns
•	SETUP Phase: Read input files, configuration, provide declarations if required. Input:
	Files required to generate patterns with EDT: edt_top_gate.v edt.dofile (internally edt_testproc will also be called)
	Files required to generate bypass patterns: edt_top_gate.v bypass.dofile (internally bypass.testproc will also be called)
•	ANALYSIS: 
	Check DRCs
	Generate patterns for a given fault model
	Get the coverage value 
	Write the output (patterns)
•	Coverage analysis and improvement:
	AU.PC: 
[1]	Write all the pin constraints faults into a file 
write_faults <file name> -class AU.PC -replace
[2]	Select a fault from the file which is written and analyse that fault
analyze_fault <pin hierarchy> -stuck 0/1 -D
[3]	After opening the fault in the visualize, give data into fault
Fix: Add a new flop 
	AU.SEQ: 
[1]	Write all the sequential depth faults into a file 
write_faults <file name> -class AU.SEQ -replace
[2]	Select a fault from the file which is written and analyse that fault
analyze_fault <pin hierarchy> -stuck 0/1 -D
[3]	After opening the fault in the visualize, give data into fault
Fix: Increase the sequential depth
	AU.WIRE: 
[1]	Write all the wire faults into a file 
write_faults <file name> -class AU.WIRE -replace
[2]	Select a fault from the file which is written and analyse that fault
analyze_fault <pin hierarchy> -stuck 0/1 -D
[3]	After opening the fault in the visualize, give data into fault
Fix: Ensure that the wire is driven by a single logic
•	Transition Delay Fault Model (TDF):
•	TDF fault model is used to detect that the logic level from 0-1 and 1-0 properly transit within the specific time (time period of fast frequency clock) or not
•	Why Fast frequency clock? Because after manufacturing (i.e., after testing when the chips go to the customer) only fast frequency clock will be used. So, we need to check whether the rise/fall of signals happens within the time period of fast frequency clock or not
•	In capture phase, we need 2 clock pulses because we need to launch and capture the transition
•	Ways to detect TDF:
	LOC: Launch on capture, Launch off capture
	LOS: Launch on Shift, Launch off shift
	LOS has slightly more coverage compared to LOC as the tool has more controllability on LOS than LOC
	Explanation: 
	The tool brings the transition value through shift path in LOS
	Whereas in LOC, the tool has to program the combo logic to get an appropriate value at the output of the combo logic to create a transition. If suppose the tool is not able to bring the appropriate value at the output of combo logic, then the faults that require that value will not get detected
	LOC requires more patterns than LOS as tool has to control the combo logic
	But there is a major limitation in LOS. Controlling SE (timing closure on SE) is extremely difficult
	Therefore, we mostly go for LOC
	Pipelined Scan Enable: In OCC, when SEN=1, we propagate slow frequency clock for both SA and TDF.
	How while using LOS method, we are able to propagate fast freq clock when SEN=1?  using Pipelined scan enable
	Why is TDF coverage less than SA
	Reason 1: Scenario where 1 input of AND gate is tied to ground. No transition faults were detected 
•	But in SA, we were able to detect 2 faults
	Reason 2: Because of timing exceptions 
•	False path (AU.FP): faults untestable due to false path
•	Multicycle path (AU.MCP): Faults untestable due to multicycle path
	The information of false path and multicycle path will be written in SDC file (Synopsys Design Constraints)
	False path: For asynchronous path, designer will be declaring as false path so that the timing engine will not try to time those path
	How false paths are taken care during SA?  if CLKA and CLKB are asynchronous, designer will declare it as false path so that timing tool will not time those paths
	SCLKA and SCLKB will also be asynchronous. As designer has declared it as a false path, timing will not be closed for F1QF2D path
	For SA, tool will take care of the problem of not closing timing along that path

	Multicycle path: It is an exception of default single cycle timing requirement path. That is on MCP, signal requires more than one clock pulse to propagate the data from start point to end point
	Scenario: sometimes the designer want to relax the setup check
	Instead of checking at cycle1, designer wants to check at cycle4
2. Scan compression (EDT + OCC)
•	Add compression hardware [EDT – Embedded Deterministic Test]
•	Decompressor (LFSR, phase shifter)
•	LFSR: Given an initial value, the LFSR will start generating patterns. But the patterns generated by the LFSR logic will have a disadvantage, i.e., there will be a diagonal relationship between the patterns
•	Phase Shifter: The patterns from LFSR logic will be given to the Phase shifter to add randomness to the patterns.
•	Even after adding phase shifter, the tool might not be able to generate all possible combinations. So to further increase the randomness in the patterns, the patterns will also be given from external channels
•	Compressor and mask logic
•	Compressor: It has levels of XORs. Compressor compresses the value of all scan chains to a single value. The compressor if encountered with any black boxes, propagates X.
•	Mask logic: To stop the propagation of X we add masking logic. Masking logic will mask the propagation of X with the help of decoders.
•	Decoders: Suppose if we don’t have decoders, the size and area will increase. There are 2 types of decoders, XOR decoder and One-hot decoder.
•	Relationship: external channels vs internal scan chains
•	Compression Ratio = Number of internal chains / Number of external channels
•	OCC
•	It is a circuit that decides which clock has to be propagated during the functional mode and DFT mode (Shift and capture) and how many clock pulses are required during the capture phase
•	Insert EDT/OCC and run DRC
•	Set context to DFT 
•	SETUP phase
•	read_verilog (scan inserted netlist)
•	read_cell_library (library file)
•	Elaboration
•	Dofile [atpg_dofile] (internally testproc will be called)
•	Specify the number of channels
•	set_edt_options
•	Go to Analysis mode
•	Check for DRC violations [analyze_drc_violations]
•	Fix EDT related violations 
•	K13: Gives information regarding newly inserted pins
•	E5: X-propagation violation
•	INSERTION
•	write_edt_files (Do the Insertion and it will write the output)
•	The script for the complete process will automatically written by the tool [dc_script.scr]
•	Link library: Used to link a design. It is used to ‘resolve’ the instanted reference in a RTL or gate level netlist
•	Target library: Used during compile command to create a technology node specific gate level netlist. DC optimization selects smallest gates that meet timing and functionality
•	Outputs
•	Compressed scan netlist [edt_top_gate.v]
•	EDT/OCC configuration

3. Simulation 
•	To validate the patterns that were generated during ATPG
•	Need of Simulation
•	While generating patterns during ATPG, we are not considering the timings (delays of cells and nets)
•	We need to check the patterns with delay before giving it to the tester. Hence, we do timing simulation
•	PD netlist  STA  Timing reports + SDF [Standard Delay Format]
•	 Delays of each and every cell and net will be available in the SDF file. In QuestaSim tool, if we read the SDF file, QuestaSim will map the delay of each and every net and cell given in SDF file to the design. This mapping is called annotation
•	Reasons for getting Simulation Mismatches:
•	Internal cut points
•	Suppose if there is a hold violation in Q-SI path which is not fixed by timing team, patterns will fail on tester
•	Suppose there is a false path (timing not done). Suppose if we have not read SDC file during ATPG TDF run, the tool will generate patterns along that path
•	Taking wrong database: Generated patterns for second release netlist. But while doing simulation, we have read the netlist of first release
•	Mismatch in library file
•	Compilation: Tool will take library, netlist and testbench and checks for syntax error (vlog command)
•	Elaboration: Elaborate the design w.r.to top module (vsim command)
•	Simulation: Apply stimuli to DUT (vsim command)
•	Pattern_serial_sample.v.0.vec (vector file/pattern file)
•	Pattern_serial_sample.v.po.name (PO names, edt_channel_out names will be present in this file) (At all locations, comparison between simulated and expected value will be done)
•	The above 2 files will be called in the testbench file pattern_serial_sample.v
•	The pattern will be applied onto our design by the testbench
•	After manufacturing, tester (ATE) applies patterns on our design
•	Similarly, during simulation, testbench is used. Testbench is used to mimic the tester
•	If the simulated output is equal to the expected output, the simulations are correct. It can be given to the tester
•	If there is a mismatch between the simulated and expected output, the simulations are wrong
•	vlog pattern_serial_sample.v -f Verilog_files.list +nospecify -override 1ns/1ps -work work -l compile.log
•	vsim -c debugDB -voptargs=+acc DmaWr_pattern_serial_sample_v_ctl -do “add wave -r /*; run -all” -c +nospecify -l simulation.log -wlf wave.wlf
•	Command to source the script file for simulation: source simulation.csh
•	To open waveform: vsim wave.wlf(name of the file where we have dumped the waveform)
•	Serial simulation: Patterns will be loaded serially into the scan chain
•	Parallel simulation: Patterns will be loaded parallelly into the scan chain
•	Debug
•	Simulation mismatches 
•	For practicing simulation mismatch debug, we have to create a simulation mismatch scenario
•	case1_edt_top_gate.v  generate the patterns (write out all the patterns I.e., don’t use -begin and -end switch in the write_pattern command)
•	Go to edt outputs directory, create a copy of case1_edt_top_gate.v file (name  simulation_mismatch.v). In this file, u8 AND gate’s A input is grounded [all this was just to create a mismatch scenario]
•	 For debugging simulation mismatches, we will use parallel simulation as we will exactly get which flop is failing
•	Steps for simulation mismatch debug
	In the simulation log, we have to check which flop is failing, at which pattern it is failing and at what time it is failing
	Then we open the QuestaSim waveform
	We have to load the particular flop’s waveform and go to the timestamp where the failure (failure flop) is happening
	In order to find out the root cause of the failure, we need to open the same flop in the ATPG session and load the failure pattern (Data  Pattern Index, Failing pattern number)
	In ATPG session, we can trace back the D input. We can open the same instance in the QuestaSim. Compare the value. Repeat the process till we find the root cause
	We can find the root cause for the failure and fix it
4. ATPG
•	Extract ICL for instrument network
•	ICL Extraction Process: 
	Automated generation of the interconnection information of the various IJTAG building blocks (SIBs, TDRs etc)
	Tessent Instrument ICL files are created during process_dft_specification
	Extract ICL process verifies the proper connectivity of the ICL modules that were inserted during the process_dft_specification command
	ICL extraction must pass with no violations in order to generate the test patterns
•	Create pattern specification
•	Default pattern spec object
•	Validate pattern spec
Create_patterns_speification is used to generate a pattern specification
Validation and processing of the specification is done using the process_patterns_verification
•	ATPG pattern generation
•	Fault Classes: The tool categorizes faults into fault classes, based on how the faults were detected or why they could not be detected
	Untestable (UT): faults for which no pattern can exist to either detect or possible-detect them. Untestable faults cannot cause functional failures, so the tool exclude them when calculating test coverage
	Unused (UU): The unused fault class includes all faults on circuitry unconnected to any circuit observation point and faults on floating primary outputs
	Tied (TI): The tied fault class includes faults on gates where the point of the fault is tied to a value identical to the fault stuck value.
	Blocked (BL): The blocked fault class includes faults on circuitry for which tied logic blocks all paths to an observable point
	Redundant (RE): The redundant fault class includes faults in the test generator consider undetectable. After the test pattern generator exhausts all patterns, it performs a special analysis to verify that the fault is undetectable under any condition
	Untestable (UT): Testable faults are all those faults that cannot be proven untestable 
	Detected (DT): The detected fault class includes all faults that the ATPG process identifies as detected. The detected fault class contains two groups:
	det_simulation (DS): faults detected when the tool performs fault simulation
	det_implication (DI): faults detected when the tool performs learning analysis. Sub classes of DI faults:
scan_path: DI faults that are directly in the scan path
scan_enable: DI faults that can propagate to and disrupt scan shifting
clock: DI faults that are in clock cone of state elements
set_reset: DI faults that are in the set/reset cone of the state elements
	Posdet (PD): The posdet, or possible-detected fault class includes all faults that fault simulation identifies as possible-detected but not hard detected. A possible-detected fault results from a good-machine simulation observing 0 or 1 and the faulty machine observing X. a hard-detected fault results from binary (not X) differences between the good and faulty machine simulations
	ATPG_untestable (AU): Testable faults become ATPG_untestable faults because of constraints, or limitations, placed on the ATPG tool (such as a pin constraint or an insufficient sequential depth). These faults may be possible-detectable, or detectable, if you remove some constraint, or change some limitation, on the test generator (such as removing pin constraint or changing the sequential depth). Sub classes of ATPG untestable (AU)
	AU.BB – BLACK_BOXES: These are faults that are untestable due to a black box, which includes faults that need to be propagated through a black box to reach an observation point, as well as faults whose control or observation requires values from the output(s) of a black box 
	AU.PC – PIN_CONSTRAINTS: These are faults that are uncontrollable or that cannot be propagated to an observation point, in the presence of a constraint value. That is, because the tool cannot toggle the pin, the tool cannot test the fanout. The only possible solution is to evaluate whether you really need the input constraint, and if not, to remove the constraint
	AU.TC – TIED_CELLS: These are faults associated with cells that are always 0 (TIE 0) or 1 (TIE 1) or X (TIE X) during capture. One example is test data register that are loaded during test_setup and then constrained so as to preserve that value during the rest of scan test
	AU.CC – CELL_CONSTRAINTS: If some faults are not getting detected because some person has added some cell constraints, we need to discuss it with the person and check whether it is a necessary cell constraint or not
	AU.UDN – UNDRIVEN: These are faults that cannot be tested due to undriven input pins. For the purpose of this analysis, undriven input pins include inputs driven by X values
	AU.WIRE – WIRE: This normally occurs when there is contention (2  or more inputs are given to a single wire)
	AU.SEQ – SEQUENTIAL_DEPTH: Sequential depth refers to the number of clock cycles required in order to capture the fault. Increasing the sequential depth means increasing the number of capture pulses. 
set_pattern_type -sequential (enables the tool to create clock sequential patterns with up to two clock pulses between scan loads)
	AU.EDT – EDT_BLOCKS: These are faults inside an instance identified as an EDT instance
	AU.IJTAG – These are faults located within the IJTAG instruments
	AU.OCC – ON_CHIP_CLOCK_CONTROL: These are faults located within the On-Chip Clock Controller (OCC). These faults (faults in the test logic) will be inherently detected by the tool.
	AU.MPO – These are faults that exist in the fanin cone of the masked POs and have no path to other observation points. PO will be masked because after manufacturing, chips will be tested parallelly on the tester and the number of pins on the tester will be limited
	Undetected (UD):
	Uncontrolled (UC): Undetected faults, which during pattern simulation, never achieve the value the value at the point of the fault detection – that is, they are uncontrollable
	Unobserved (UO): Faults whose effects do not propagate to an observable point
	UD.AAB – ATPG_ABORT: these are faults that are undetected because the tool reached its abort limit. You can raise the abort limit to increase coverage using the set_abort_limit command
	UD.UNS – UNSUCCESS: These are faults that are undetected for unknown reasons. There is nothing you can do about faults in this sub-class
	UD.EAB – EDT_ABORT: If your design’s chain:channel ratio is insufficient to detect the fault
	Fault Coverage: It is a measure of percentage of faults that can be detected by the generated test patterns out of the total number of faults that could exist in a circuit
	Fault coverage = total number of faults detected / total faults in a circuit (Where total faults includes testable faults and untestable faults)
	Test coverage = total number of faults detectable / total faults testable
	Test coverage will be more compared to fault coverage as in test coverage untestable faults are not included
	We will mainly try to improve the test coverage because untestable faults will not aaffect the functionality of the design
•	Fault models: SA, TDF, IDDQ, PDF
•	Fault categories: dominance, equivalence, collapse
	Fault Dominance: Fault F dominates fault G if detecting set of F contains that of G
	Fault Equivalence: Two faults F1 and F2 are equivalent if all tests that detect F1 also detect F2 and vice versa
	Fault Collapse: Removing equivalent faults from the total fault list is called fault collapsing
Fault Aliasing: A fault is aliased when it is observed by an even number of scan cells that happened to line up at the same location in different scan chains that are compacted to the same output channel
•	Pattern types:
	Serial Patterns:
	For serial patterns, the number of clock cycles required for each pattern, during shift cycle is same as the number of flops in that chain (Without EDT).
	Serial patterns without EDT: number of cycles in shift phase = number of flops   in scan chains + masking bits + initialization cycles
	For capture, 1 clock pulse is required
	The output is measured and compared against the expected value (ATPG tool while writing patterns will give the input sequence + expected output) in the shift out phase at the scan out port
	Procedure for serial pattern:
4.	Force si
5.	Measure so
6.	Pulse clock
	Parallel Patterns:
	For parallel patterns, for each pattern only 1 clock pulse is required for shift as patterns are loaded parallelly
	Output is measured and compared with the expected value at the output of each flop
4.	Sense scan cell
5.	Force parallel data 
6.	Pulse clock
	We measure output first in parallel simulation because, in parallel simulation, Verilog keyword ‘force’ is used to load the data parallelly. If ‘force’ is used, then the net will automatically get the value which is forced. So, the previous output of the flop will get overwritten with the new value that is forced. So, in parallel simulation, we first sense the scan cell and then force parallel data 
	We need both serial and parallel pattern:
	Parallel patterns:
	Run time is less. So, we validate all the patterns and only a sample of serial patterns before manufacturing
	Easy to debug
	serial patterns:
	In tester serial patterns will be given. So, before going to the tester, a sample of serial patterns will be validated

•	ATPG flow
•	SETUP → ANALYSIS → coverage improvement → finalize patterns
•	SETUP Phase: Read input files, configuration, provide declarations if required. Input:
	Files required to generate patterns with EDT: edt_top_gate.v edt.dofile (internally edt_testproc will also be called)
	Files required to generate bypass patterns: edt_top_gate.v bypass.dofile (internally bypass.testproc will also be called)
•	ANALYSIS: 
	Check DRCs
	Generate patterns for a given fault model
	Get the coverage value 
	Write the output (patterns)
•	Coverage analysis and improvement:
	AU.PC: 
[4]	Write all the pin constraints faults into a file 
write_faults <file name> -class AU.PC -replace
[5]	Select a fault from the file which is written and analyse that fault
analyze_fault <pin hierarchy> -stuck 0/1 -D
[6]	After opening the fault in the visualize, give data into fault
Fix: Add a new flop 
	AU.SEQ: 
[4]	Write all the sequential depth faults into a file 
write_faults <file name> -class AU.SEQ -replace
[5]	Select a fault from the file which is written and analyse that fault
analyze_fault <pin hierarchy> -stuck 0/1 -D
[6]	After opening the fault in the visualize, give data into fault
Fix: Increase the sequential depth
	AU.WIRE: 
[4]	Write all the wire faults into a file 
write_faults <file name> -class AU.WIRE -replace
[5]	Select a fault from the file which is written and analyse that fault
analyze_fault <pin hierarchy> -stuck 0/1 -D
[6]	After opening the fault in the visualize, give data into fault
Fix: Ensure that the wire is driven by a single logic
•	Transition Delay Fault Model (TDF):
•	TDF fault model is used to detect that the logic level from 0-1 and 1-0 properly transit within the specific time (time period of fast frequency clock) or not
•	Why Fast frequency clock? Because after manufacturing (i.e., after testing when the chips go to the customer) only fast frequency clock will be used. So, we need to check whether the rise/fall of signals happens within the time period of fast frequency clock or not
•	In capture phase, we need 2 clock pulses because we need to launch and capture the transition
•	Ways to detect TDF:
	LOC: Launch on capture, Launch off capture
	LOS: Launch on Shift, Launch off shift
	LOS has slightly more coverage compared to LOC as the tool has more controllability on LOS than LOC
	Explanation: 
	The tool brings the transition value through shift path in LOS
	Whereas in LOC, the tool has to program the combo logic to get an appropriate value at the output of the combo logic to create a transition. If suppose the tool is not able to bring the appropriate value at the output of combo logic, then the faults that require that value will not get detected
	LOC requires more patterns than LOS as tool has to control the combo logic
	But there is a major limitation in LOS. Controlling SE (timing closure on SE) is extremely difficult
	Therefore, we mostly go for LOC
	Pipelined Scan Enable: In OCC, when SEN=1, we propagate slow frequency clock for both SA and TDF.
	How while using LOS method, we are able to propagate fast freq clock when SEN=1?  using Pipelined scan enable
	Why is TDF coverage less than SA
	Reason 1: Scenario where 1 input of AND gate is tied to ground. No transition faults were detected 
•	But in SA, we were able to detect 2 faults
	Reason 2: Because of timing exceptions 
•	False path (AU.FP): faults untestable due to false path
•	Multicycle path (AU.MCP): Faults untestable due to multicycle path
	The information of false path and multicycle path will be written in SDC file (Synopsys Design Constraints)
	False path: For asynchronous path, designer will be declaring as false path so that the timing engine will not try to time those path
	How false paths are taken care during SA?  if CLKA and CLKB are asynchronous, designer will declare it as false path so that timing tool will not time those paths
	SCLKA and SCLKB will also be asynchronous. As designer has declared it as a false path, timing will not be closed for F1QF2D path
	For SA, tool will take care of the problem of not closing timing along that path

	Multicycle path: It is an exception of default single cycle timing requirement path. That is on MCP, signal requires more than one clock pulse to propagate the data from start point to end point
	Scenario: sometimes the designer want to relax the setup check
	Instead of checking at cycle1, designer wants to check at cycle4
5. Scan insertion
•	Invoke scan insertion flow
•	Source the Mentor Graphics tool [source /home/cshrc]
•	Invoke the Mentor Graphics tool [/home/TESSENT/bin/tessent -shell]
•	Input files required for scan insertion: Design input file (Gate level netlist) and Library file (Contains the functionality of each and every cell present in the design)
•	Files need to be created by us: Dofile (can store all the instructions or commands) and Log file (Stores the procedure carried out in the tool environment).
•	Phases: SETUP → ANALYSIS → INSERTION
•	SETUP: Set the context (scan/edt), read the design files and library files and elaboration and fixing DRC violations after analysing
•	ANALYSIS: Checking for DRC violations and analysing them and declaring the number of chains that we want to create
•	INSERTION: Replace the normal flops with scan flops. Write out reports (scan cell and scan chain reports) and outputs (scan inserted netlist, ATPG Dofile, ATPG test proc and Scandef file).
•	Insert scan structures
•	Replace flops with scan cells
•	Adding a mux to the design which includes Functional input (0) and Scan input (1) connected to the Flop (D) 
•	Build scan chains (shift/capture)
•	All scan flop will be stitched together to form a scan chain
•	Define test mode vs scan enable
•	Test mode: To distinguish between Functional mode [TM = 0] and Test mode (DFT) [TM = 1]
•	Scan Enable: To distinguish between Shift phase [SE = 1] and Capture phase [SE = 0]
•	Fix scan DRC and quality
•	Scan DRC violations
DRC (Design Rule Checks) The rules which a flop must pass to convert from normal flop to scan flop
1.	Clock controllability DRC 
•	The clock definition of a flop is missing or it is connected to some other flop 
•	Fix: Declare the clock [add_clock <Off state> <name of the clock>] or add a mux
2.	Set/Reset controllability DRC
•	If Set/Reset port of a flop is connected to some other flop or combo logic’s output, if is tied to ground or if Set/Reset port of a flop is missing
•	Fix: Declare Set/Reset port [add_pin_constraints <Off state C0/C1> <Set/Reset name>] or [add_clock <off state> <Set/Reset name>] or add a mux
3.	Bus contention
•	Occurs when two or more bus drivers force opposite values onto a bus
•	Fix: Command [set_test_logic -tristate on] 
4.	Feedback loop DRC
•	The output of the feedback will be oscillating, so the tool cannot decide the value of the net is 0 or 1
•	Fix: Add a mux
5.	X-Source DRC
•	A logic whose output is X because of presence of analog block or memories
•	Fix: Bypass the logic
6.	Potential Race DRC
•	Clock Data Race (clock signal goes to Clock port and D port) Set Reset Race (Same signal is controlling the Set and Reset port of the flop)
•	Fix: Add a mux
7.	Violations 
•	S1: When clock or set/reset declaration is missing or it is connected to some flop/combo logic’s output or if it is tied to a fix value
o	Fix: Command: [add_clock <off state> <name of the clock>]
•	S2: When clock of a flop is tied to a fixed value
o	Fix: Add a mux
•	D5: If a flop has S1/S2 violation, if the scan equivalent of a flop is not available in the library file or if it is a latch
o	Declare the clock
•	C6: Clock pin is connected to D input and clock pin of another flop.
o	Fix: Add a mux
•	C3: Positive edge triggered flop connected to Negative edge triggered flop
o	Fix: Add a lockup latch
•	C7: States that the flop doesn’t have the ability to capture. It is because clock of a flop is tied to a fixed value or if it is not declared
o	Fix: Add a mux
•	We can fix these violations manual editing of netlist, by using some tessent commands [create_connection, delete_connection, create_instance], and also there are also some of the autofix commands
•	set_test_logic -clock on
•	set_test_logic -set on
•	set_test_logic -reset on
•	set_test_logic -tristate on [E10-bus contention]
•	set_test_logic -C6 on
•	Scan chain balancing
•	Keeping almost equal number of flops in the scan chains
•	Unbalanced chains will lead to increase in test time. We fix this by adding dummy values to the chains
•	Edge / clock domain mixing
•	It is a solution to balance the chains. Edge Mixing: Keeping the positive and negative edge triggered flops in the same chain. Domain Mixing: Keeping the clocks with different clock domains in the same chain
•	Command: insert_test_logic -number (number of chains) -edge merge -clock merge
•	Scan chain reordering
•	It is done by PD team. They use the scandef file for this.
•	Outputs
•	Scan inserted netlist
•	Design modified with inserted test structures. Normal flops are replaced with their scan equivalents and scan chains are stitched
•	New ports are added such as scan en, scan in and scan out
•	Scan cell and scan chain reports
•	Scan chain report: Holds number of scan chains. And also, the information about scan chains such as 
•	Scan cell report: 
•	Scan DRC report




































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

-exclude port port_spec → Excludes specified ports from wrapper analysis

set_dedicated_wrapper_cell_options
set_dedicated_wrapper_cell_options


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




