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


# Logic Optimization and Synthesis – Workshop Notes

## Table of Contents
- [Combinational Optimization](#combinational-optimization)
- [Sequential Logic Optimization](#sequential-logic-optimization)
- [Unused Output Optimization](#unused-output-optimization)
- [Gate-Level Simulation (GLS) and Synthesis-Simulation Mismatch](#gate-level-simulation-gls-and-synthesis-simulation-mismatch)

***

## Combinational Optimization

Combinational logic can be optimized in several ways to reduce area, power, and delay:

1. **Constant Propagation**
   - If an input to a logic gate is a constant, the output can often be simplified. For example, if one input to an AND gate is 0, the output is always 0.
2. **Boolean Simplification**
   - Techniques like Karnaugh maps (K-maps) or the Quine–McCluskey algorithm are used to minimize Boolean expressions, reducing the number of gates and logic levels.

***

## Sequential Logic Optimization

**Basic Techniques:**
- **Sequential Constant Propagation:**
  - If a flip-flop's input is always a constant (e.g., D=0 for a DFF), its output is fixed and the flip-flop can be removed, simplifying the circuit.

**Advanced Techniques:**
- **State Optimization:**
  - Remove unused or redundant states in state machines to reduce complexity.
- **Cloning:**
  - Duplicate registers closer to where their outputs are needed to reduce delay.
- **Retiming:**
  - Move flip-flops across combinational logic to balance delays and potentially increase the maximum clock frequency.

**Example:**
- If D=0 for a DFF, Q will always be 0, so the DFF and any logic depending on Q can be eliminated. However, if an asynchronous set or reset is present, optimization may not be possible unless the output is truly constant.

***


1. **UpCounter (3-bit)**
   - If only `q` is used, the other outputs are unused and can be removed from the design. For example, if `q = count`, only the LSB is needed, so a single flip-flop is sufficient (as seen in the synthesis report).
   - If `q = count[2:0] == 3'b100`, all bits are needed, so three flip-flops are required and no optimization is possible.
   - Any logic or storage not affecting the output is optimized away by the synthesis tool.

***
## Unused Output Optimization 

1. **UpCounter (3bit)**
   we are only usinf the q[0] the other outputs are unused thus they neednot be present in the design.q=count[0] -> depend on msb only.
   In q=count[2:0]==3'b100 depend on all the bits.
   In case 1 the bit is toggled in all cycle.-> one flop is enough which we see in the synthesis report. the dff output is take and fed back into the d which toggles it evervy cycle.
   Any LOGIC that doesnt used all the outputs is OPTIMIZED.
   In case 2 three flop is needed which we see in the synthesis report. So the Output is not Optimized.




</details> <details> <summary>Day 4 - GLS, Blocking vs Non-blocking and Synthesis-Simulation Mismatch</summary>

# Logic Optimization and Synthesis – Workshop Notes

## Table of Contents
- [Combinational Optimization](#combinational-optimization)
- [Sequential Logic Optimization](#sequential-logic-optimization)
- [Unused Output Optimization](#unused-output-optimization)
- [Gate-Level Simulation (GLS) and Synthesis-Simulation Mismatch](#gate-level-simulation-gls-and-synthesis-simulation-mismatch)

***

## Combinational Optimization

Combinational logic can be optimized in several ways to reduce area, power, and delay:

1. **Constant Propagation**
   - If an input to a logic gate is a constant, the output can often be simplified. For example, if one input to an AND gate is 0, the output is always 0.
2. **Boolean Simplification**
   - Techniques like Karnaugh maps (K-maps) or the Quine–McCluskey algorithm are used to minimize Boolean expressions, reducing the number of gates and logic levels.

***

## Sequential Logic Optimization

**Basic Techniques:**
- **Sequential Constant Propagation:**
  - If a flip-flop's input is always a constant (e.g., D=0 for a DFF), its output is fixed and the flip-flop can be removed, simplifying the circuit.

**Advanced Techniques:**
- **State Optimization:**
  - Remove unused or redundant states in state machines to reduce complexity.
- **Cloning:**
  - Duplicate registers closer to where their outputs are needed to reduce delay.
- **Retiming:**
  - Move flip-flops across combinational logic to balance delays and potentially increase the maximum clock frequency.

**Example:**
- If D=0 for a DFF, Q will always be 0, so the DFF and any logic depending on Q can be eliminated. However, if an asynchronous set or reset is present, optimization may not be possible unless the output is truly constant.

***


1. **UpCounter (3-bit)**
   - If only `q` is used, the other outputs are unused and can be removed from the design. For example, if `q = count`, only the LSB is needed, so a single flip-flop is sufficient (as seen in the synthesis report).
   - If `q = count[2:0] == 3'b100`, all bits are needed, so three flip-flops are required and no optimization is possible.
   - Any logic or storage not affecting the output is optimized away by the synthesis tool.

***

## Gate-Level Simulation (GLS) and Synthesis-Simulation Mismatch

### What is GLS?
Gate-Level Simulation (GLS) is the process of simulating the synthesized netlist (post-synthesis) to verify that the design still functions as intended. The netlist is logically equivalent to the RTL, so the same testbench can be used.

### Why Run GLS?
1. **Verify logical correctness after synthesis**
2. **Ensure timing constraints are met**

### GLS with Icarus Verilog
- The netlist is now a gate-level model. The simulator must be aware of the standard cells used. The rest of the simulation flow remains the same, but the GLS model can be timing-aware.

### What Happens in GLS?
- Example RTL: `assign y = (a & b) | c;`
- Example Netlist:
  ```verilog
  and a1(m, a, b);
  or o1(y, c, m);
  ```
- The gate-level model includes definitions for gates like AND and OR, which may include timing information. This allows both functional and timing verification.
- The simulator only evaluates the design when there is a change in the inputs.

### Why Validate Functionality Again?
Even if the RTL and netlist are logically equivalent, mismatches can occur due to:
- **Missing sensitivity lists**: For example, using `always @(sel)` instead of `always @(*)` can cause the simulator to miss changes on other inputs, leading to incorrect behavior (like unintended latches).
- **Blocking vs. Non-Blocking Assignments**:
  - `=` (blocking): Executes statements in order, like C code.
  - `<=` (non-blocking): Schedules assignments to happen in parallel, regardless of order.

#### Example: Shift Register
- **Blocking:**
  ```verilog
  q = q0;
  q0 = d;
  ```
  Here, `q0` is assigned to `q`, then `d` is assigned to `q0`. This works as expected.
- **Non-Blocking:**
  ```verilog
  q0 <= d;
  q <= q0;
  ```
  Both assignments happen in parallel, so the order doesn't matter and the correct behavior is achieved.

#### Caveats
- If you use a variable before it is updated in the same always block, you might unintentionally create a latch or a flop, depending on the assignment type and order.
- Even if two codes produce the same output in RTL simulation, they might behave differently after synthesis. That's why GLS is essential to catch these mismatches.



</details> <details> <summary>Day 5 - Introduction to DFT</summary>

## If-Else and Elif Ladder in Verilog

The `if-else` and `else if` ("elif ladder") constructs in Verilog implement conditional logic with **priority**. In hardware, these synthesize into a chain of multiplexers:

- The **first** `if` condition has the highest priority. If it is true, its block executes and the rest are skipped.
- If none of the conditions are true, the `else` (default) block executes.

**Example:**
```verilog
always @(*) begin
  if (cond1)
    y = a;
  else if (cond2)
    y = b;
  else
    y = E;
end
```

**Hardware Analogy:**
- This is like a nested multiplexer: the first true condition determines the output.
- If all conditions are false, the default value (`E`) is selected.

***

## Inferred Latches: Dangers and Prevention

**Inferred latches** occur when not all possible conditions assign a value to an output in a combinational always block. This causes the synthesis tool to create a latch (memory element) to "remember" the previous value.

**Example of Inferred Latch:**
```verilog
always @(*) begin
  if (cond1)
    y = a;
  else if (cond2)
    y = b;
  // No else: y keeps its previous value (latch inferred)
end
```

**Why is this a problem?**
- Latches can cause unpredictable behavior and are usually not intended in combinational logic.

**Best Practice:**
- Always include an `else` or `default` case to assign all outputs in every branch, unless you specifically want a latch (e.g., in counters).
- If you want to "hold" the value, use `y = y;` in the `else` block.

***

## Case Statements: Usage and Caveats

The `case` statement is used for multi-way branching, similar to a multiplexer with many inputs.

**Syntax Example:**
```verilog
always @(*) begin
  case (sel)
    2'b00: y = a;
    2'b01: y = b;
    2'b10: y = c;
    default: y = d;
  endcase
end
```

**Key Points:**
- The variable assigned in a `case` statement should be declared as `reg`.
- If not all possible values of the selector are covered and there is no `default`, inferred latches may occur.
- If you have multiple outputs, you must assign *all* outputs in *every* case branch. Otherwise, latches may still be inferred, even with a `default`.
- `case` statements do not have priority: all cases are checked in parallel, and only one should match.
- Overlapping cases are not allowed.

**Case vs. If-Else:**
- Use `if-else` for priority logic.
- Use `case` for one-hot or mutually exclusive conditions.

***

## Loops in Verilog

Verilog supports two main types of loops: for use in always blocks (behavioral) and for hardware instantiation (generate).

### Always Block For Loops
- Used inside `always` blocks for repeated assignments or calculations.
- Does **not** create hardware loops; instead, it unrolls into repeated logic.
- Useful for things like ripple adders or wide multiplexers.

**Example:**
```verilog
always @(*) begin
  for (i = 0; i < 8; i = i + 1)
    sum[i] = a[i] ^ b[i];
end
```

### Generate For Loops
- Used outside `always` blocks to instantiate multiple hardware modules or logic blocks.
- Cannot be used inside `always` blocks.
- Common for creating arrays of gates, registers, etc.

**Example:**
```verilog
genvar i;
generate
  for (i = 0; i < 4; i = i + 1) begin : and_gen
    and u_and (out[i], in1[i], in2[i]);
  end
endgenerate
```

***

## Summary Table: Best Practices

| Construct         | Best Practice                                      |
|------------------|----------------------------------------------------|
| if-else ladder   | Always include an else/default branch              |
| case statement   | Cover all cases and assign all outputs in each     |
| always for loop  | Use for repeated assignments, not hardware loops   |
| generate for     | Use for hardware instantiation, not in always      |

***


</details> 
