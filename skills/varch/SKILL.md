---
name: varch
description: Decompose an IP requirement into submodules, define interfaces, and generate per-module requirement files
---

# Verilog Architect

## Overview

This guide defines the workflow for decomposing a top-level IP requirement into submodules, defining their interfaces, and generating individual requirement files ready for `vdesign`.

The input is a top-level IP requirement `.md` file. The outputs are:

1. A module decomposition proposal with block diagram and interface definitions
2. One requirement `.md` file per submodule in `./ddoc/`
3. A top-level integration skeleton in `./rtl/`

## Step 1 - Analyze Requirements and Propose Decomposition

Read the top-level IP requirement `.md` file and extract the design intent.

Before proposing a new decomposition, check `./lib` for any existing library modules that already cover part of the required functionality. Use `vexplain` to read and understand any relevant library modules. Reuse them rather than redesigning.

Based on the requirements, propose a module decomposition. The proposal must include:

- A list of submodules, each with:
  - Name
  - Single-sentence responsibility
  - Key inputs and outputs
- A block diagram showing submodule connections, drawn in mermaid format
- Justification for any module boundary decision that is non-obvious

If an existing library module covers a required function, include it in the block diagram as-is and do not generate a requirement file for it.

## Step 2 - Define Interfaces

For each connection between submodules shown in the block diagram, define the interface in detail:

- Signal name (following project bus naming conventions: `<master>2<slave>_<bus_name>`)
- Direction relative to each module
- Signal width and type
- Protocol or handshake description if applicable

Present the interface definitions as a table for each inter-module connection.

## Step 3 - Generate Submodule Requirement Files

For each new submodule (excluding reused library modules), generate a requirement `.md` file in the `./ddoc/` folder.

Output file rules:

- Use the submodule name as the file name
- Use the `.md` extension

Example:

- submodule `foo_ctrl` → `./ddoc/foo_ctrl.md`

### Required Content for Each Requirement File

Each requirement file must include:

- Module overview and responsibility
- Parameter descriptions, if applicable
- Input/Output port descriptions with signal names and widths from Step 2
- Functional description — what the module must do, not how
- Interface protocol description for each port, if applicable

The requirement file is the direct input to `vdesign`. It must be self-contained and unambiguous.

## Step 4 - Generate Top-Level Integration Skeleton

Create a top-level Verilog skeleton file in the `./rtl/` folder.

Output file rules:

- Use the IP name as the file name with `_top` suffix
- Use the `.v` extension

Example:

- IP `foo` → `./rtl/foo_top.v`

### Skeleton Structure

The skeleton must include:

- Module declaration with the IP-level ports from the requirement
- `wire` declarations for all inter-module interfaces defined in Step 2
- One instantiation stub per submodule with port connections wired up
- `//!` comments on any connection or signal that requires special attention during integration

No logic implementation — only port declarations, wire declarations, and instantiation stubs.

Example skeleton structure:

```verilog
module foo_top #(
    parameter DATA_W = 8
) (
    input  wire              clk,
    input  wire              rst_n,
    input  wire [DATA_W-1:0] s_data,
    input  wire              s_valid,
    output wire [DATA_W-1:0] m_data,
    output wire              m_valid
);

//==============================================================================
// Inter-module interfaces
//==============================================================================

wire [DATA_W-1:0] ctl2prc_data;
wire              ctl2prc_valid;

//==============================================================================
// Submodule Instantiations
//==============================================================================

foo_ctrl #(
    .DATA_W (DATA_W)
) u_foo_ctrl (
    .clk      (clk),
    .rst_n    (rst_n),
    .s_data   (s_data),
    .s_valid  (s_valid),
    .m_data   (ctl2prc_data),
    .m_valid  (ctl2prc_valid)
);

foo_proc #(
    .DATA_W (DATA_W)
) u_foo_proc (
    .clk      (clk),
    .rst_n    (rst_n),
    .s_data   (ctl2prc_data),
    .s_valid  (ctl2prc_valid),
    .m_data   (m_data),
    .m_valid  (m_valid)
);

endmodule
```
