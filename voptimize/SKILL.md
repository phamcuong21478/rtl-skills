
---
name: voptimize
description: Optimize Verilog code for better synthesis result
---

# Verilog Optimizer

## Overview

This guide describes the workflow for optimizing a Verilog module from annotated source files. The target of optimization could be Maximum Frequency or Minimum Resource usage

---
## Prerequisites

| Tool      | Purpose                                      | Check                 |
|-----------|----------------------------------------------|-----------------------|
| Verilator | Transpile Verilog into C++                   | `verilator --version` |
| g++       | Compile C++ and link shared objects          | `g++ --version`       |
| GTKWave   | View FST waveforms (optional)                | `which gtkwave`       |
| Vivado    | Run synthesis, implementation, and reporting | `vivado -version`     |

---
## Step 1 - Review Requirements and Prepare Design Documentation

Read the Verilog source file carefully. All comments starting with `\\!` contain the information and guidance for the module to be implemented. Ignore all other comment types unless explicitly instructed otherwise.

You may ask the user for permission to revise the module information lines if clarification is needed or if spelling and wording issues may affect interpretation.

Propose a design based on the available requirements. The proposal should include:

- A description of the module’s function for user confirmation
- A basic description of the algorithm, if applicable
- A description of the module architecture if the design is large, complex, or algorithmically non-trivial. This item may be skipped for small or simple modules
- Timing diagrams showing input and output behavior under different operating scenarios. Use wavedrom format to draw the waveform

Each item must be confirmed by the user individually. If the user disagrees with a proposal item, provide an alternative. If the second proposal is still not accepted, stop the process.

Once the user confirms that all proposed content is acceptable:

1. Write the design proposal to a new Markdown file in the same folder as the Verilog source file. The new file must use the same base name as the Verilog file and end with `_proposal.md`.

2. Write the design documentation to a new Markdown file in the `./doc` folder. The new file must use the same base name as the Verilog file and have the `.md` extension.

The design documentation must include:

- Module overview
- Parameter description, if applicable
- Input and output port descriptions
- The module function description taken from the proposal
- A basic description of the algorithm, if applicable, taken from the proposal
- Timing diagrams for different module operating scenarios taken from the proposal

---
## Step 2 - Fill in the Verilog Code

Complete the Verilog source file based on the approved proposal and the coding style guidelines defined in `assets/CodingStyle.md`.

