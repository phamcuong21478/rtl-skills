# RTL Design Project

## Project Overview

This is an RTL design project for developing hardware modules in Verilog.
The project includes RTL source code, reusable library modules, documentation generated during development, and project files for simulation and synthesis.

## Project Structure

```text
.
├── CLAUDE.md              // project instructions (this file)
├── ddoc/                  // per-module requirement files, inputs for vdesign
├── doc/                   // generated module and IP documentation
├── lib/                   // reusable RTL library modules
│   ├── module_1.v         // single-file library module
│   ├── module_1.md        // documentation for module_1
│   └── module_2/          // multi-file library module
│       ├── module_2.v     // top-level RTL
│       ├── any.v          // submodule used by module_2
│       └── module_2.md    // documentation for module_2
├── rtl/                   // RTL code developed for this project
│   ├── module_m.v
│   └── module_n.v
└── prj/                   // simulation and synthesis project files
```

- Every module in `./rtl` must have a corresponding `./doc/<module>.md`.
- Every module in `./lib` must include a `<module>.md` in the same folder.
- Per-module requirement files live in `./ddoc/<module>_req.md`.
- Design proposals live in the same folder as the RTL: `./rtl/<module>_proposal.md`.

## Design Workflow

```
ip_req.md
  │
  ├─[varch]──► ddoc/<submodule>_req.md  (one per submodule)
  │            rtl/<ip>_top.v           (instantiation skeleton)
  │
  ├─[vdesign]─► rtl/<module>.v          (backbone with //! directions)
  │             <module>_proposal.md    (architecture + algorithm)
  │             doc/<module>.md         (module documentation)
  │
  ├─[vfill]──► rtl/<module>.v           (complete implementation)
  │            tb/<module>/             (testbench + simulation)
  │
  └─[vdoc]───► doc/<ip>.md             (full IP integration document)
```

## Available Skills

| Skill     | Purpose |
|-----------|---------|
| `varch`   | Decompose an IP requirement into submodules; generate per-module `ddoc/` requirement files and a top-level skeleton |
| `vdesign` | Generate a design proposal, module documentation, and RTL backbone with `//!` directions from a requirement file |
| `vfill`   | Implement the Verilog code from an approved proposal, then lint and simulate |
| `vexplain`| Generate documentation for any RTL module from its Verilog source |
| `vdoc`    | Aggregate all per-module documentation into a single IP-level document |
| `voptimize`| Optimize Verilog code for better synthesis results |

## Available Agents

| Agent              | Model  | Skills          | Purpose |
|--------------------|--------|-----------------|---------|
| `rtl-architect`    | Opus   | varch, vexplain | Decompose an IP requirement into submodules |
| `rtl-designer`     | Opus   | vdesign         | Generate proposal, docs, and RTL backbone |
| `rtl-coder`        | Sonnet | vfill           | Implement code, lint, and simulate |
| `rtl-documentation`| Sonnet | vdoc, vexplain  | Build complete IP documentation |

## Conventions

- `//!` comments in RTL files are design direction markers for `vfill`. Do not remove them until the implementation is complete.
- Coding style rules are defined in `.claude/skills/shared/CodingStyle.md`.
- Reset: always synchronous; active-low by default; use `RS_LV` parameter when configurable.
- All Verilog must be Verilog-2001 compatible — no SystemVerilog constructs.
