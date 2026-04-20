# RTL Design Project

## Project Overview

This is an RTL design project for developing small hardware modules using Verilog.  
The project includes the RTL source code, reusable library modules, supporting documentation, and generated project files for simulation and synthesis.

## Project Structure

```text
.
├── CLAUDE.md              // this file
├── doc                    // folder containing design documentation
├── lib                    // folder containing reusable RTL library modules used by the project
│   ├── module_1.v     // RTL design of module 1, only one file
│   ├── module_1.md    // detailed documentation for module 1
│   └── module_2           // folder for module 2, could include more than one file
│       ├── module_2.v     // top-level RTL design of module 2
│       ├── any.v          // submodule used by module 2
│       └── module_2.md    // detailed documentation for module 2
├── rtl                    // folder containing all RTL code developed specifically for this project
│   ├── module_m.v
│   └── module_n.v
└── prj                    // folder containing generated project files for simulation and synthesis
```

Each module in the `./rtl` folder should have a corresponding documentation file in the `./doc` folder.

Each module in the `./lib` folder should include documentation that explains its functionality and how to use it.

