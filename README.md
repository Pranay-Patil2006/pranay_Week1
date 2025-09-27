# pranay_Week1
week 1, documentation and notes of RISC-V SoC Tapeout workshop

<details>
  <summary>Day 1 - Introduction to Verilog RTL Design and Synthesis</summary>

# RTL Design and Synthesis Workshop Notes

This repository contains comprehensive notes from a workshop covering RTL design simulation and synthesis using open-source tools.

## Table of Contents
- [RTL Simulation](#rtl-simulation)
- [Design and Testbench](#design-and-testbench)
- [Iverilog Design Flow](#iverilog-design-flow)
- [Logic Synthesis](#logic-synthesis)
- [Liberty Files (.lib)](#liberty-files-lib)
- [Timing Considerations](#timing-considerations)
- [Synthesis with Yosys](#synthesis-with-yosys)

## RTL Simulation

RTL (Register Transfer Level) design verification is performed through simulation to ensure the design meets specifications. The simulator monitors input signal changes and re-evaluates outputs whenever changes are detected.

**Key Tool:** Iverilog - An open-source Verilog simulator used for design verification.

## Design and Testbench

### Design
- Contains Verilog code that implements the required specifications
- Includes primary inputs and outputs
- Represents the actual hardware functionality

### Testbench
- Setup for applying stimulus to verify the design
- Acts as a stimulus generator
- Contains logic to drive inputs to the design under test
- Monitors and verifies design outputs
- Bidirectional relationship: testbench outputs feed design inputs, design outputs feed back to testbench

## Iverilog Design Flow

The simulation flow follows these steps:

1. **Input Files:** Design file (`.v`) and testbench file (`.v`)
2. **Compilation:** Use iverilog to compile both files
3. **Simulation:** Execute the compiled output to generate waveform data
4. **Visualization:** View results using GTKWave

### Commands:
```bash
# Compile design and testbench
iverilog input_design_file.v input_test_bench_file.v

# Execute simulation
./a.out

# View waveforms (generates input_test_bench_file.vcd)
gtkwave input_test_bench_file.vcd
```

**VCD File:** Value Change Dump file containing signal transitions over time for waveform analysis.

## Logic Synthesis

### Overview
Logic synthesis transforms RTL (behavioral) code into gate-level netlist representation. This process converts high-level Verilog descriptions into actual hardware gates that can be implemented.

**Key Tool:** Yosys - Open-source synthesis tool

### Synthesis Process
1. **Input:** RTL design + Liberty file (.lib)
2. **Process:** Synthesis tool maps RTL to available gates
3. **Output:** Gate-level netlist

### Verification
The synthesized netlist must be functionally equivalent to the original RTL:
- Same testbench can verify both RTL and netlist
- Same primary inputs and outputs
- Identical simulation results (VCD files should match)

## Liberty Files (.lib)

Liberty files contain characterization data for standard cell libraries:

- **Content:** Logical modules (AND, OR, NOT, etc.)
- **Variations:** Multiple drive strengths (slow, medium, fast)
- **Configurations:** Different input counts (2-input, 3-input, 4-input gates)
- **Purpose:** Provides timing, power, and area information for synthesis optimization

## Timing Considerations

### Setup Time Constraint
For proper sequential circuit operation:

```
T_clk > T_cq_A + T_combi + T_setup_B
```

Where:
- `T_clk`: Clock period
- `T_cq_A`: Clock-to-Q delay of source flip-flop
- `T_combi`: Combinational logic delay
- `T_setup_B`: Setup time of destination flip-flop

### Maximum Frequency
```
f_max = 1/T_clk
```

### Cell Selection Strategy

**Fast Cells:**
- Reduce combinational delays
- Help meet setup time requirements
- Higher power consumption and area

**Slow Cells:**
- Provide necessary delays for hold time requirements
- Prevent race conditions
- Lower power and area

**Optimization Goal:** Balance speed, power, and area requirements by selecting appropriate cell variants.

## Synthesis with Yosys

### Basic Yosys Commands

| Command | Purpose |
|---------|---------|
| `read_verilog` | Load Verilog design files |
| `read_liberty` | Load standard cell library files |
| `write_verilog` | Generate synthesized netlist |

### Synthesis Flow Example

```tcl
# Read design file
yosys> read_verilog good_mux.v

# Read liberty file
yosys> read_liberty -lib /home/chippy/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130B/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# Synthesize design (specify top module)
yosys> synth -top good_mux

# Technology mapping using ABC
yosys> abc -liberty /home/chippy/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130B/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# Display synthesized netlist
yosys> show

# Write netlist file
yosys> write_verilog netlist.v
```

### Key Points
- The same testbench verifies both RTL and synthesized netlist
- Netlist represents the true gate-level implementation
- ABC command performs technology mapping to standard cells
- The liberty file used is the Sky130 PDK standard cell library at typical corner (tt_025C_1v80)

## Workshop Tools Summary

- **Iverilog:** Simulation and verification
- **GTKWave:** Waveform visualization
- **Yosys:** Logic synthesis
- **Sky130 PDK:** Process design kit with standard cell libraries
</details> <details> <summary>Day 2 - Timing libs, Hierarchical vs Flat Synthesis and Efficient Flop Coding Styles</summary>
# RTL Design and Synthesis Workshop Notes - Day 2

# RTL Design and Synthesis Workshop Notes - Day 2

## Table of Contents
- [PVT in Liberty Files](#pvt-in-liberty-files)
- [Cell Variants and Area](#cell-variants-and-area)
- [Hierarchical Synthesis](#hierarchical-synthesis)
- [Flip-Flop Reset Strategies](#flip-flop-reset-strategies)
- [Yosys Flow for Sequential Logic](#yosys-flow-for-sequential-logic)
- [Optimization Techniques](#optimization-techniques)
- [Common Yosys Commands](#common-yosys-commands)

## PVT in Liberty Files

**PVT** stands for **Process, Voltage, and Temperature**. These three factors determine how silicon chips behave in real-world conditions:

- **Process**: Variations in manufacturing (like doping, lithography) cause each chip to behave slightly differently
- **Voltage**: Chips may be run at different supply voltages, affecting speed and power
- **Temperature**: Performance and leakage change with temperature

A typical liberty file name, like `sky130_fd_sc_hd__tt_025C_1v80.lib`, encodes these conditions:
- `tt` = typical process
- `025C` = 25°C
- `1v80` = 1.80V supply

This file contains detailed parameters for each cell (like AND, OR, etc.) under these conditions: leakage power, current, rise/fall times, slew rates, and more. Each cell may have several variants (e.g., `and1`, `and2`) with different drive strengths and areas. Larger area usually means higher speed and more leakage.

## Cell Variants and Area

- **Multiple cells** for the same logic function (e.g., `and1`, `and2`) differ mainly in transistor sizing
- **Larger cells**: Faster, but use more area and power
- **Smaller cells**: Slower, but save area and power

## Hierarchical Synthesis

**Stacking PMOS** transistors increases resistance and slows down the circuit. Synthesis tools prefer using NAND gates (which have parallel PMOS) over NOR gates (which have stacked PMOS) for efficiency.

By default, `write_verilog` in Yosys preserves the module hierarchy. To flatten the design into a single module, use:

```tcl
yosys> flatten
```

**Submodule-level synthesis** is useful when you have multiple instances of the same module or want to break down a large design for easier synthesis and optimization:

```tcl
yosys> synth -top <sub_module_name>
```

## Flip-Flop Reset Strategies

- **Asynchronous reset**: Flip-flop resets immediately when the reset signal changes, regardless of the clock
- **Synchronous reset**: Flip-flop resets only on the clock edge, making timing analysis easier

## Yosys Flow for Sequential Logic

After synthesis, map flip-flops to library cells using your specific liberty file:

```tcl
yosys> dfflibmap -liberty /home/chippy/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130B/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

## Optimization Techniques

- The synthesizer tries to minimize the number of gates and optimize for area, speed, and power
- For example, multiplying by 2 is implemented as a left shift, not a full multiplier
- If the synthesis does not use library cells, the `abc` command will not generate a mapped netlist. In that case, use `show` to view the netlist directly

## Common Yosys Commands

| Command | Purpose |
|---------|---------|
| `read_verilog <file.v>` | Read Verilog source |
| `read_liberty -lib <libfile.lib>` | Read liberty file |
| `synth -top <module>` | Synthesize top module |
| `flatten` | Remove hierarchy |
| `dfflibmap -liberty <libfile.lib>` | Map flip-flops to library |
| `abc -liberty <libfile.lib>` | Technology mapping |
| `write_verilog <out.v>` | Write synthesized netlist |
| `show` | View netlist |

</details> <details> <summary>Day 3 - Combinational and Sequential Optimizations</summary>
Content for Day 3 goes here.

</details> <details> <summary>Day 4 - GLS, Blocking vs Non-blocking and Synthesis-Simulation Mismatch</summary>
Content for Day 4 goes here.

</details> <details> <summary>Day 5 - Introduction to DFT</summary>
Content for Day 5 goes here.

</details> <details> <summary>Day 6 - Introduction to Logic Synthesis</summary>
Content for Day 6 goes here.

</details> <details> <summary>Day 7 - Basics of Static Timing Analysis (STA)</summary>
Content for Day 7 goes here.

</details> ```
