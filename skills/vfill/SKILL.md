
---
name: vfill
description: Generate Verilog code based on the directions and information provided in comment lines
---

# Verilog Writer

## Overview

This guide describes the workflow for implementing a Verilog module from annotated source files.
Comments beginning with `\\!` are treated as the authoritative design instructions for the module.

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

When searching for information to use a sub-module in library, prioritize reading only the documentatRe-run lint after apply fix for all warning. 
ion file of the sub-module. Only read the RTL code file if you lack sufficient information.

---
## Step 3 - Clean code

Use verilator to check lint for file code. command format:

```bash
verilator --lint-only --top-module <top_module> -y <link_to_rtl_folder> -y <link_to_library_folder> -y <link_to_module_folder_in_library_folder> <link_to_top_module_file>
```

where:
- `--top-module <top_module>` define top design module
- `-y <link_to_rtl_folder>` define relative link to folder hold rtl code of project. for example, it could be `-y ./rtl`
- `-y <link_to_library_folder>` define relative link to folder hold sub-module library. for example, it could be `-y ./lib`
- `-y <link_to_module_folder_in_library_folder>` define relative link to folder hold sub-module library. for example, it could be `./lib/module_1`. for each sub-module that have sub-folder, we need to add this own flag.
- `<link_to_top_module_file>` define relative link to file hold top design that defined in flag `--top-module`. for example, it could be `./rtl/top_module.v`

check warning and error report. for each warning/error, propose fix code to clean warning



---
name: vfill
description: Generate Verilog code from design instructions provided in comment lines
---

# Verilog Writer

## Overview

This guide defines the workflow for implementing a Verilog module from annotated source files.

Treat all comments beginning with `\\!` as authoritative design instructions for the target module.

## Prerequisites

| Tool      | Purpose                                      | Check                 |
|-----------|----------------------------------------------|-----------------------|
| Verilator | Transpile Verilog into C++                   | `verilator --version` |
| g++       | Compile C++ and link shared objects          | `g++ --version`       |
| GTKWave   | View FST waveforms (optional)                | `which gtkwave`       |
| Vivado    | Run synthesis, implementation, and reporting | `vivado -version`     |

## Step 1 - Review Requirements and Prepare Design Documentation

Read the target Verilog source file.

Interpret all comments starting with `\\!` as design requirements or implementation guidance for the target module. Ignore all other comment types unless explicitly instructed otherwise.

If any `\\!` comment is unclear, ambiguous, or contains spelling or wording issues that could affect interpretation, ask the user for permission before modifying those lines.

Based on the available requirements, generate a design proposal. The proposal must include:

- Functional description of the module
- Algorithm description, if applicable
- Module architecture description, if the design is large, complex, or algorithmically non-trivial
- Timing diagrams showing input and output behavior for different operating scenarios. Must cover all scenarios.

Timing diagrams must be written in WaveDrom format.

If the design is small or simple, the architecture description may be omitted.

## Step 1A - Get User Confirmation

Present each proposal item to the user for confirmation.

Confirmation rules:

1. Each item must be confirmed individually.
2. If the user rejects an item, provide one alternative.
3. If the alternative is also rejected, stop the workflow.

Do not proceed to documentation generation or code implementation until all required proposal items are accepted.

## Step 1B - Generate Proposal and Documentation Files

Generate the following files:

### Proposal File

Create a new Markdown file in the same folder as the target Verilog file. Use wavedrom format for write timing diagrams.

Output file rules:

- Use the same base name as the target Verilog file
- Append `_proposal` before the extension
- Use the `.md` extension

Example:

- `foo.v` → `foo_proposal.md`

### Design Documentation File

Create a new Markdown file in the `./doc` folder.

Output file rules:

- Use the same base name as the target Verilog file
- Use the `.md` extension

Example:

- `foo.v` → `./doc/foo.md`

## Required Content for the Design Documentation

The design documentation must include:

- Module overview
- Parameter descriptions, if applicable
- Input/Output port descriptions
- Functional description of the module
- Timing diagrams for different operating scenarios

The functional description, and timing diagrams must be consistent with the approved proposal.

## Step 2 - Fill in the Verilog Code

Complete the target Verilog source file based on:

- The approved proposal
- The coding style rules defined in `assets/CodingStyle.md`

When the target module uses library submodules, follow this lookup order:

1. Read the submodule documentation first
2. Read the submodule RTL code only if the documentation is not sufficient

Do not read library RTL code unless additional information is required to understand the interface or behavior.

## Step 3 - Clean the Code

Run Verilator lint on the completed design.

Use the following command format:

```bash
verilator --lint-only --top-module <top_module> -y <path_to_rtl_folder> -y <path_to_library_folder> -y <path_to_library_submodule_folder> <path_to_top_module_file>
```

Where:
- `--top-module <top_module>` specifies the top-level module name
- `-y <path_to_rtl_folder>` specifies the relative path to the project RTL folder, for example -y ./rtl
- `-y <path_to_library_folder>` specifies the relative path to the library root folder, for example -y ./lib
- `-y <path_to_library_submodule_folder>` specifies the relative path to a submodule folder inside the library, for example -y ./lib/module_1
- `<path_to_top_module_file>` specifies the relative path to the file containing the top-level module, for example ./rtl/top_module.v

Add one `-y` option for each required library subfolder.

### Step 3A - Resolve Lint Issues

Review all Verilator warnings and errors.

For each warning or error:

1. Determine whether it is caused by incorrect RTL code, missing files, missing include paths, or intentional design behavior.
2. Propose a fix.
3. If the user chooses to ignore a warning, suppress it explicitly by using a Verilator lint directive.
4. Apply the fix only if it does not conflict with the approved design requirements.

Use the following format to suppress a specific warning:

```verilog
/* verilator lint_off <WARNING_CODE> */
    a <= b;     // line of code that causes the warning
/* verilator lint_on <WARNING_CODE> */
```

After all proposed fixes have been applied, run lint again.

Repeat this process until:

- No remaining lint errors exist
- Remaining warnings, if any, are explicitly justified and cannot be safely removed without changing the intended design

Do not ignore warnings silently.
