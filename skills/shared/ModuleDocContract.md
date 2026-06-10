# Module Documentation Contract

This contract defines the required content of a per-module design document
`./doc/<module>.md` (for `./rtl` modules) or `<module>.md` alongside the source
(for `./lib` modules).

Any skill that generates a module document (`vdesign`, `vexplain`) must satisfy
this contract. The goal is a single rule: a module document must be complete
enough that another skill can **instantiate, drive, and reason about the module
without reading its RTL source**. Skills that consume documentation (`vfill`,
`vdebug`, `vsynth`, `vtestgen`, `vdoc`) rely on this being true.

## Required Sections

A module document must contain the following sections, in this order. Sections
marked _(if applicable)_ may be omitted when they genuinely do not apply to the
module; all others are mandatory.

### 1. Module Overview

One short paragraph: what the module does and its role in the larger design.

### 2. Parameters _(if applicable)_

A table. Omit the section only if the module has no parameters.

| Parameter | Default | Legal Range | Description |
|-----------|---------|-------------|-------------|

- `Default` must be the actual default value in the module declaration.
- `Legal Range` states the values the design supports (e.g. `1–32`, `power of 2`).
- If parameters constrain **each other** (e.g. `DEPTH` must be a power of 2;
  `DATA_W` a multiple of 8 when `BYTE_EN=1`), state the cross-parameter
  constraints in a line below the table — a per-parameter range cannot express
  them.

### 3. Ports

A table covering **every** port. This is the instantiation contract — names and
widths must match the RTL exactly.

| Name | Direction | Width | Reset Value | Description |
|------|-----------|-------|-------------|-------------|

- `Direction` is `input` / `output` / `inout`.
- `Width` is the expression as declared (e.g. `DATA_W`, `[7:0]`, `1`).
- `Reset Value` is the value the port holds after reset for outputs, or `—` for
  inputs and for outputs that are not registered to a defined reset value.

### 4. Interface Protocol

For each bus or handshake interface, describe how to drive or sample it:

- Handshake semantics (e.g. valid/ready, request/grant, strobe).
- When outputs are valid relative to inputs.
- Any ordering, framing, or back-pressure rules.

For a module with only trivial standalone signals (no handshake), state that
explicitly in one line rather than omitting the section.

For any **non-trivial handshake** (anything beyond a single always-accepted
valid), include an **example transaction** as a beat-by-beat cycle table — one
row per cycle, one column per interface signal — covering at least one normal
beat and one stalled/back-pressured beat. Prose states the rule; the table
removes the ambiguity prose hides.

### 5. Reset Behavior

- Synchronous or asynchronous (this project: always synchronous).
- Active level (active-low by default, or per `RS_LV`).
- The state the module enters on reset and the value of each output at reset
  (consistent with the Reset Value column in Section 3).

### 6. Timing Characteristics

- Latency from input to corresponding output, in clock cycles.
- Pipeline depth, if pipelined.
- Throughput (e.g. one transaction per cycle, or initiation interval).
- Clock domain(s); note any multi-clock or CDC boundary.

For a purely combinational module, state "combinational; no clocked latency".

### 7. Functional Description

What the module does, not how it is coded. Must be consistent with the design
proposal when one exists.

### 8. Algorithm / FSM _(if applicable)_

For algorithmically non-trivial modules or any module with an FSM, include:

- A basic algorithm description.
- FSM state diagram and/or algorithm flow drawn in **mermaid** format.

### 9. Register Map _(if applicable)_

For any module exposing CPU-accessible registers (a CSR block, any
bus-controlled module). Omit the section only if the module has no registers.

A table covering **every** register:

| Address | Name | Access | Reset | Description |
|---------|------|--------|-------|-------------|

- `Access` is `RO` / `RW` / `WO` / `W1C` / `RC` (read-to-clear). **Mark `W1C`/`RC`
  explicitly** — clear-on-read/-write bits carry the one-shot poll hazard
  (`CodingStyle §14`) and an integrator must know which bits self-clear *before*
  polling them.
- State whether addresses are **word** or **byte** addresses on the module's
  native bus.
- For a register with multiple fields, add a per-register bit-field table
  (bits, field name, access, reset, meaning).

This section is the source the bench reg-maps (`vtestgen`'s `dut.vh`, `vveri`'s
`register.h`) and the release `register_reference.md` derive from. If it is
missing, those skills fall back to reading the RTL address decode — exactly what
this contract exists to prevent.

### 10. Usage Constraints _(if applicable)_

Integration rules an integrator cannot see in the port table:

- **Sequencing rules** — e.g. "configure before setting `ENA`; do not change
  `MODE` while busy"; required idle cycles between operations.
- **Input stability assumptions** — e.g. "`s_data` must hold while
  `s_valid && !s_ready`" (if it is the standard valid/ready rule, say so in one
  line).
- **Behavior on documented error / boundary conditions** — overflow, underflow,
  illegal input, protocol violation: what the module guarantees, drops, or
  flags.

When in doubt, "No usage constraints beyond the standard handshake" is a
one-line statement, not an omitted section.

### 11. Instantiation Template

A copy-paste Verilog instantiation with **named port connections**
(`CodingStyle §9`) and every parameter bound to its default:

```verilog
foo #(
    .DATA_W (8),
    .RS_LV  (0)
) u_foo (
    .clk     (clk),
    .rst_n   (rst_n),
    .s_data  (/* [DATA_W-1:0] in */),
    .s_valid (/* in */),
    .m_data  (/* [DATA_W-1:0] out */),
    .m_valid (/* out */)
);
```

Names, widths, and parameters must match §2/§3 exactly — this is the first thing
an integrator copies, and a drifted template is worse than none.

## Excluded Content

A module document must **not** include:

- Detailed implementation internals or line-by-line code explanation.
- Redundant documentation for submodules already documented elsewhere
  (link to their `<module>.md` instead).
- Information unrelated to the behavior or interface of the module.

## Interpreting Signal Names

Use `.claude/skills/shared/CodingStyle.md` to interpret names while documenting:
suffix `_ff` = register; prefix `s0_`/`s1_` = pipeline stage; port prefix
`s_`/`m_` = slave/master interface (CodingStyle §3.2); internal bus
`<master>2<slave>_<bus>` identifies the connected module pair.
