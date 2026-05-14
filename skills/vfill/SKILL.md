---
name: vfill
description: Implement Verilog code from an approved design proposal, then lint and simulate
---

# Verilog Writer

## Overview

This guide defines the workflow for implementing a Verilog module from an approved design proposal.

Before starting, read:

1. The target Verilog source file — for `//!` annotations and port declarations
2. The proposal file `<module_name>_proposal.md` in the same folder — this is the source of truth for the implementation

## Prerequisites

| Tool      | Purpose                             | Check                 |
|-----------|-------------------------------------|-----------------------|
| Verilator | Verilog lint                        | `verilator --version` |
| xvlog     | Xilinx Verilog compiler / lint      | `xvlog --version`     |
| xelab     | Xilinx elaborator                   | `xelab --version`     |
| xsim      | Xilinx simulator                    | `xsim --version`      |

## Step 1 - Fill in the Verilog Code

Complete the target Verilog source file based on:

- The approved proposal file
- The coding style rules defined in `.claude/skills/shared/CodingStyle.md`

When the target module uses submodules, follow this lookup order:

1. Read the submodule documentation first. It could be in `./doc`, `./lib` and sub-folder, or `./rtl`
2. Read the submodule RTL code only if the documentation is not sufficient

Do not read RTL code unless additional information is required to understand the interface or behavior.

## Step 2 - Clean the Code

Run lint in two passes. Both must pass before proceeding to simulation.

### Pass 1 — Verilator

```bash
verilator --lint-only --top-module <top_module> -y <path_to_rtl_folder> -y <path_to_library_folder> -y <path_to_library_submodule_folder> <path_to_top_module_file>
```

Where:
- `--top-module <top_module>` specifies the top-level module name
- `-y <path_to_rtl_folder>` specifies the relative path to the project RTL folder, for example `-y ./rtl`
- `-y <path_to_library_folder>` specifies the relative path to the library root folder, for example `-y ./lib`
- `-y <path_to_library_submodule_folder>` specifies the relative path to a submodule folder inside the library, for example `-y ./lib/module_1`
- `<path_to_top_module_file>` specifies the relative path to the file containing the top-level module, for example `./rtl/top_module.v`

Add one `-y` option for each required library subfolder.

### Pass 2 — Xilinx xvlog + xelab

```bash
xvlog -nolog <rtl_files>
xelab -nolog --top <top_module>
```

Where `<rtl_files>` is the list of all Verilog source files required by the design.

### Step 2A - Resolve Lint Issues

Review all warnings and errors from both tools.

For each warning or error:

1. Determine whether it is caused by incorrect RTL code, missing files, missing include paths, or intentional design behavior.
2. Propose a fix.
3. If the user chooses to ignore a warning, suppress it explicitly using a lint directive.
4. Apply the fix only if it does not conflict with the approved design.

For Verilator, use the following format to suppress a specific warning:

```verilog
/* verilator lint_off <WARNING_CODE> */
    a <= b;     // line of code that causes the warning
/* verilator lint_on <WARNING_CODE> */
```

After all proposed fixes have been applied, re-run both lint passes.

Repeat this process until:

- No remaining lint errors exist in either tool
- Remaining warnings, if any, are explicitly justified and cannot be safely removed without changing the intended design

Do not ignore warnings silently.

## Step 3 - Functional Simulation

Generate a self-checking testbench file `<module_name>_tb.v` in the folder `./tb/<module_name>`. All compile and simulation commands must run from inside this folder.

The testbench must:

- Cover every operating scenario from the approved design
- Assert outputs against expected values using `$error` on mismatch
- Print a final PASS or FAIL summary

Run simulation using xsim from inside `./tb/<module_name>`. All paths must be relative to that folder — use `../../rtl` for the RTL folder and `../../lib` for the library folder.

```bash
xvlog -nolog <rtl_files> <testbench_file>
xelab -nolog --top <testbench_module> --snapshot sim_snap
xsim sim_snap --runall --nolog
```

### Step 3A - Resolve Simulation Failures

For each failing scenario:

1. Determine whether the fault is in the RTL or the testbench.
2. Propose a fix.
3. Apply the fix only if it does not conflict with the approved design.

Repeat until all scenarios pass.

## Known Issues and Fixes

This section records issues encountered when running this skill and how they were resolved. Use it to avoid repeating the same mistakes.

<!-- Add entries below as issues are discovered. Format:
### [Tool] - <short issue title>
**Symptom:** what was observed
**Cause:** root cause
**Fix:** what was done to resolve it
-->

