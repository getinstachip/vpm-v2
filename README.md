# VPX Tool Installation and Usage Guide

## Overview
**VPX** is a command-line tool that leverages state-of-the-art (SOTA) AI agents to assist with design, debugging, and documentation of RTL modules. 

### Key Commands
- `vpx implement <instructions>`: Implements RTL modules based on high-level design instructions.
- `vpx document <module_path.sv>` (coming soon): Generates module documentation.
- `vpx debug <module_path.sv>` (coming soon): Detects and suggests fixes for RTL modules.

---

## Installation Steps

### Step 1: Install VPX Using pipx

To install VPX, use `pipx`, which ensures isolated dependencies. Run:

```bash
pipx install vpx
```

### Step 2: Log In to VPX

Once installed, log in to activate your license:

```bash
vpx login
```

When prompted, enter:
- **Your email address**
- **Your license key**

### Troubleshooting

- **Command Not Found**: Verify `pipx` installation and ensure it’s in your PATH.
- **Login Issues**: Check that the correct email and license key are used and confirm internet connectivity.
- **Dependency Issues**: Reinstall VPX using `pipx reinstall vpx` if needed.

---

## Using the `vpx implement` Command

The `vpx implement` command guides an AI model through a structured thought process for RTL design. Below are example use cases demonstrating how VPX helps generate RTL code.

---

### Example 1: 8-bit Multiply-Accumulate (MAC) Module

#### Design Requirements

The `MAC8` module performs multiply-accumulate operations on two 8-bit inputs (`a` and `b`), storing the accumulated result in a 16-bit output (`result`). Here’s how the AI model approaches this implementation.

#### Thought Process

1. **Define the Module Interface**
   - Inputs: `clk`, `rst_n`, `en`, `clear`, `a[7:0]`, `b[7:0]`
   - Output: `result[15:0]`
   ```verilog
   module MAC8 (
       input  wire        clk,
       input  wire        rst_n,
       input  wire        en,
       input  wire        clear,
       input  wire [7:0]  a,
       input  wire [7:0]  b,
       output reg  [15:0] result
   );
   ```

2. **Identify Storage Elements**
   - Registers: `a_reg[7:0]`, `b_reg[7:0]` store inputs; `acc_reg[23:0]` accumulates the result to prevent overflow.
   - **Data Flow**: `a` and `b` → Input registers → Multiply → Accumulate in `acc_reg`.

3. **Timing Diagram Analysis**
   ```plaintext
   Clock     |‾|_|‾|_|‾|_|‾|_|‾|_|
   en        |1|1|1|1|0|0|1|1|0|0|
   clear     |1|0|0|0|0|0|1|0|0|0|
   a[7:0]    |A1|A1|A2|A2|xx|xx|A3|A3|xx|xx|
   b[7:0]    |B1|B1|B2|B2|xx|xx|B3|B3|xx|xx|
   result    |00|P1|S1|S2|S2|S2|00|P3|S3|S3|
   
   Where:
   - A1, B1: First input values
   - P1: First product (A1*B1)
   - S1, S2: Accumulated sums
   ```
4. Register Update Order
   - Execution Sequence:
      1. Clear: Clear `acc_reg` if clear=1.
      2. Sample Inputs: Capture new `a` and `b` values in `a_reg` and `b_reg` if `en=1`.
      3. Multiply: Calculate product = `a_reg * b_reg` (combinational).
      4. Accumulate: Add product to `acc_reg` on the next clock edge.
   - Critical Timing Rules:
      1. `en=1` enables register updates; `clear=1` takes precedence.
      2. Accumulator and inputs update in sync with `clk`.
         
5. **Final RTL Code**
   ```systemverilog
   module MAC8 (
       input  wire        clk,
       input  wire        rst_n,
       input  wire        en,
       input  wire        clear,
       input  wire [7:0]  a,
       input  wire [7:0]  b,
       output reg  [15:0] result
   );

       // Internal registers
       reg [7:0]  a_reg, b_reg;
       reg [23:0] acc_reg;  // 24-bit accumulator to prevent overflow
       wire [15:0] product;

       assign product = a_reg * b_reg;
       
       always_ff @(posedge clk or negedge rst_n) begin
           if (!rst_n) begin
               a_reg <= 8'h0;
               b_reg <= 8'h0;
               acc_reg <= 24'h0;
               result <= 16'h0;
           end else if (en) begin
               a_reg <= a;
               b_reg <= b;
               if (clear) acc_reg <= {8'h0, product};
               else acc_reg <= acc_reg + {8'h0, product};
               result <= acc_reg[15:0];
           end
       end
   endmodule
   ```

---

### Example 2: FSM-Based Lemmings Module

The `TopModule` implements an FSM to model a Lemming's walking, falling, and digging behaviors. Below is the thought process for this complex design.

#### Thought Process

1. **Define the Module Interface**
   - Inputs: `clk`, `areset`, `bump_left`, `bump_right`, `ground`, `dig`
   - Outputs: `walk_left`, `walk_right`, `aaah`, `digging`

2. **Identify and Define Storage Elements (State and Counters)**
   - **State Register**: `WALK_LEFT`, `WALK_RIGHT`, `FALL_LEFT`, `FALL_RIGHT`, `DIG_LEFT`, `DIG_RIGHT`, `SPLAT`
   - **Counter**: `fall_count` to track cycles in `FALL` state. When `fall_count > 20`, Lemming enters `SPLAT` on ground return.

3. **Determine Combinational Logic Paths**
   - **Priority**:
     1. **FALL**: Ground = 0 (highest priority).
     2. **SPLAT**: `fall_count > 20` when ground returns.
     3. **DIG**: Dig = 1 and Ground = 1 (overrides direction change).
     4. **Direction Change**: Triggered by `bump_left` or `bump_right`.

4. **Timing Diagram Analysis**
   ```plaintext
   Clock     |‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|
   ground    |1|1|0|0|0|0|1|1|1|1|1|1|1|1|
   bump_left |0|0|0|0|0|0|0|1|0|0|0|0|0|0|
   dig       |0|0|0|0|0|0|0|0|1|1|0|0|0|0|
   state     |WL|WL|FL|FL|FL|FL|WL|WR|DG|DG|FL|FL|WL|WL|
   
   fall_cnt  |0|0|1|2|3|4|0|0|0|0|0|1|0|0|
   walk_left |1|1|0|0|0|0|1|0|0|0|0|0|1|1|
   walk_right|0|0|0|0|0|0|0|1|0|0|0|0|0|0|
   aaah      |0|0|1|1|1|1|0|0|0|0|1|1|0|0|
   digging   |0|0|0|0|0|0|0|0|1|1|0|0|0|0|
   ```

5. **FSM Representation**
   ```
              WALK_LEFT ---------- bump_left=1 ----------> WALK_RIGHT
                  |                                         |
      ground=0    |                                         |   ground=0
                  v                                         v
              FALL_LEFT <------------ bump_right=1 ------- WALK_RIGHT
                  |                                         |
       dig=1,     |                                         |   dig=1,
     ground=1     v                                         | ground=1
     ------------> DIG_LEFT                               DIG_RIGHT <----------
                  |                                         |                |
        ground=0  v                                         v ground=0       |
     ------------ FALL_LEFT <-------- bump_right=1 ------- FALL_RIGHT       SPLAT
                  |                   ground=1, fall_count > 20
                  v
               SPLAT (All outputs = 0)
   ```

   - **Outputs**:
     - `walk_left`: Asserted in `WALK_LEFT`.
     - `walk_right`: Asserted in `WALK_RIGHT`.
     - `aaah`: Asserted in `FALL_LEFT` and `FALL_RIGHT`.
     - `digging`: Asserted in `DIG_LEFT` and `DIG_RIGHT`.
     - **SPLAT**: All outputs are deasserted.

---

## Using the `

vpx document` Command

The `vpx document` command will soon generate module documentation based on RTL code or natural language input.

Coming soon.

## Using the `vpx debug` Command

The `vpx debug` command will offer debugging tools, including logic cone analysis, timing diagrams, and autonomous RTL debugging assistance.

Coming soon.

--- 

This documentation provides a comprehensive overview of VPX, guiding users through installation, setup, and the use of the `vpx implement` command with clear design examples and detailed thought processes.
