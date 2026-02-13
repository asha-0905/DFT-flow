

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


