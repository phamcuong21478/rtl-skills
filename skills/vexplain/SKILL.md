---
name: vexplain
description: Generate documentation for any RTL module from Verilog source files
---

# Verilog Reader

## Overview

This guide defines the workflow for generating documentation for a Verilog RTL module from its source files.

## Step 1 - Analyze the Design

Read the target Verilog source file and determine the function of the RTL module.

Use `.claude/skills/shared/CodingStyle.md` to correctly interpret signal names, bus interfaces, and FSM patterns found in the code. For example:
- Signal suffix `_ff` indicates a flip-flop or register.
- Signal prefix `s0_`, `s1_` indicates pipeline stage.
- Port prefix `s_` / `m_` / `b_` indicates slave / master / monitor bus interface.

If the module instantiates submodules, follow this order:

1. Check whether each submodule already has documentation.
2. If documentation exists, use it to understand the submodule behavior.
3. If documentation does not exist, read the submodule RTL code only as needed to understand the behavior of the target module.

After the analysis is complete, generate a new Markdown file according to the following output rules:

Output file rules:

- The output file must use the same base name as the Verilog file.
- The output file must use the `.md` extension.
- If the target file is under `./rtl`, place the output in `./doc/`.
- If the target file is under `./lib`, place the output in the same folder as the Verilog file.

## Required Content

The generated documentation must include:

- Module overview
- Parameter descriptions, if applicable
- Input and Output port descriptions
- Functional description of the module
- Basic algorithm description, if applicable

Use mermaid format to draw FSM state and Algorithm flow, if applicable.

## Excluded Content

The generated documentation must not include:

- Detailed implementation internals
- Line-by-line code explanation
- Redundant documentation for submodules that are already documented elsewhere
- Information unrelated to the behavior or interface of the target module
