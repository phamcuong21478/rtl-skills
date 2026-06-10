# FGBW RTL — Verilog Coding Style Guide

Rules for writing Verilog in this project. Follow them when creating or editing
any `.v` module so new code matches the house style. Rules are mandatory unless
marked *prefer*.

> The file-header banner (Company/Author/Email/dates/Function) is generated and
> maintained by the company's automatic tool — do not write or edit it by hand.

---

## 0. Language standard

- **RTL (synthesizable `.v` files) must be plain Verilog-2005 (IEEE 1364-2005)
  only.** No SystemVerilog (IEEE 1800) constructs in design code — no
  `logic`/`bit`, `always_ff`/`always_comb`/`always_latch`, `typedef`/`struct`/
  `enum`, `interface`, `package`, `.*` port connections, `unique`/`priority`, or
  `assert`. Use only Verilog-2005 features (`reg`/`wire`,
  `always @(posedge clk)`, `generate`, `localparam`, `$clog2`, etc.).
- **Testbenches may use SystemVerilog** (`.sv` files): `logic`, classes,
  `interface`, `program`, constrained-random, assertions, `$display` helpers,
  etc. are all allowed in verification code, never in the DUT.
- Keep the split by file extension: synthesizable design in `.v` (Verilog-2005),
  verification in `.sv` (SystemVerilog).

## 1. Module declaration

- Use ANSI-style port lists (ports declared inline in the header), not Verilog-95.
- Use the `module name #( ... ) ( ... );` form for parameters.
- Indent the parameter and port declarations one level (4 spaces) inside the
  module header; put the closing `)`, and the `) (` between the parameter and
  port lists, in column 0 (aligned with `module`).

```verilog
module fgbw_cal #(
    parameter ID_NO = 16,   //! number of channel ID, ID_NO < G1*G2
    parameter ID_W  = 4,    //! ID bus width
    parameter RS_LV = 0     //! reset active level (0 = active-low)
) (
    input clk,      //! clock
    input rst_n,    //! reset
    ...
);
```

- Parameter names are `UPPER_SNAKE_CASE`, aligned in a column, with `//!`
  doc-comments aligned on the right.
- `clk` and `rst_n` are always the first two ports.
- `RS_LV` (reset active level) is always the **last** parameter in a clocked
  module. **Default `RS_LV = 0` (active-low)** is the project standard. The reset
  port is always named `rst_n` regardless of `RS_LV` — the name is a fixed
  convention, not a polarity claim; the actual polarity is whatever `RS_LV` sets
  (reset is asserted when `rst_n == RS_LV`, see §5).

## 2. Parameters & localparams

- Use `parameter` for module-configurable values; use `localparam` for
  internally-derived values.
- Width-parameter naming: use `*_W` for any width in bits (bus, field, counter);
  use `*_NO` for a quantity/count of items (e.g. `ID_W`, `D_W`, `RM_W`, `ID_NO`).
  Prefer `*_W` for widths — the older `*_BIT` suffix is deprecated.
- Localparams may use elaboration-time expressions (`$clog2`, modulo, ternaries).
  Write multi-line arithmetic with the leading-operator style and the `;` on its
  own line:

```verilog
localparam LPF_W  = MBIT          // main phase
                    + 2             // extended for detect overflow
                    + $clog2(TAP)   // number of tap
                    ;
```

## 3. Naming conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Module | `fgbw[_<block>]` lowercase | `fgbw_cal_scheduler` |
| Parameter / localparam | `UPPER_SNAKE` | `TS_W`, `QR_RAM_W` |
| Registers (flip-flops) | suffix `_ff` | `cal_req_ff`, `pus_s1_id_ff` |
| FSM next-state (comb) | suffix `_ns` | `cal_state_ns` |
| Wires / nets | lowercase snake, functional prefix | `sch_rd_req`, `rr_rd_id` |
| Pipeline stage signals | `[<pipe>_]s<N>_<signal>[_ff]` | `pus_s3_count_waddr`, `s2_head_ff` |
| Hazard / forwarding | `hz_<resource>_p<X>_p<Y>_ff` | `hz_cww_p3_p2_ff` |
| Memory port bundle | `m<name>_{st,waddr,wdata,ld,raddr,rdata}` | `mlink_raddr`, `mhead_rdata` |
| Generate index | `gi` (declared `genvar gi;`) | |
| Register-file regs | `rf_*` (+ `_ff`) | `rf_period_1s_ff` |
| CPU/UPI signals | `upi_*` / `up_*` | `s_upi_addr`, `up_ce` |
| Master/slave bus (at a port) | `m_*` / `s_*` | `m_valid`, `s_carry` |
| Inter-module net (in parent) | `<master>2<slave>_<bus>` or `<master>_<bus>` | `ctl2prc_data`, `sch_rd_req` |

- **Every flip-flop carries the `_ff` suffix; combinational nets never do.**
- Use functional sub-block prefixes to namespace signals within a module
  (e.g. `rr_` round-robin, `ar_` access-request, `bm_` burst-monitor,
  `ck_` checker, `sp_` scheduler-process).

### 3.1 Pipeline naming

A module may contain several independent pipeline **processes** (e.g. a PUSH
path and a POP path running in parallel). Give each process its own short name
and prefix **every** signal in that process with it, so two pipelines never
collide on a stage number:

| Process | Prefix | Example signals |
|---------|--------|-----------------|
| init engine | `ip_` | `ip_waddr_ff`, `ip_fifo_init_ff` |
| push path | `pus_` | `pus_s0_active`, `pus_s3_count_wdata` |
| pop path | `pop_` | `pop_s2_head_ff`, `pop_s3_rnull` |

- **Stage signal format: `[<pipe>_]s<N>_<signal>[_ff]`**, where `<pipe>` is the
  process name, `s<N>` is the stage number (`s0`, `s1`, …), `<signal>` is the
  functional name, and `_ff` is appended only to the registered copy.
- **`<pipe>` is optional when the module has only one data pipeline** (the `ip_`
  init engine does not count). In that case drop the prefix for shorter names —
  `s0_active`, `s3_count_wdata`. Add the `<pipe>` prefix as soon as a second
  data pipeline appears, so the two never collide on a stage number.
- Within a stage, the combinational value carries no `_ff` and is fed by the
  previous stage's register; the registered copy adds `_ff` and feeds the next
  stage. This makes the data hand-off explicit:

```verilog
    // stage 2 reads stage 1's register, stage 2's register feeds stage 3
    wire  pus_s2_active = pus_s1_active_ff;
    reg   pus_s2_active_ff;
    always @(posedge clk)
        if (rst_n == RS_LV) pus_s2_active_ff <= 'd0;
        else                pus_s2_active_ff <= pus_s2_active;
```

- Label each stage with a `//=== stage N: <what happens>` banner (see §4).
- For a variable-depth shift of one stage, use a range in the stage field
  (`s<X>t<Y>` = stages X through Y) and a packed register array indexed by the
  depth parameter:

```verilog
    reg  pop_s4t8_active_ff [1:MEM_PIPELINE];   // stages 4..8, depth = MEM_PIPELINE
```

- **Hazard / forwarding signals** that compare two pipeline stages use
  `hz_<resource>_p<X>_p<Y>_ff`, read as "resource hazard between stage X and
  stage Y" (e.g. `hz_cww_p3_p2_ff` = counter-write-write hazard, stage 3 vs 2).
  The matching forwarding mux selects the bypassed value when the `hz_*` flag is
  set.

### 3.2 Bus naming

A **bus** is a named group of related signals that move together and share one
prefix `<bus>_`. The prefix names the bus once; each member appends its role.
This keeps a multi-signal interface visually grouped and lets a whole bus be
passed down the hierarchy under one name.

- Prefix every member of a bus with the bus name, then the role suffix:
  `<bus>_<member>`. **Bus members never carry the `_ff` suffix** even when they
  are driven by a register — the bus name identifies an interface net, not a
  flip-flop. Register the value into a separate `_ff` signal and drive the bus
  member from it if needed.
- Add a direction prefix where the project uses one — `s_` for a slave (the
  block is the target) and `m_` for a master (the block drives) — giving
  `s_<bus>_*` / `m_<bus>_*`.
- Document the bus once with a section banner (§4) listing its members, rather
  than commenting each signal.

Standard buses in this project:

| Bus | Prefix | Members |
|-----|--------|---------|
| CPU / UPI register bus (slave) | `s_upi_*` | `s_upi_we`, `s_upi_oe`, `s_upi_addr`, `s_upi_din`, `s_upi_dout` |
| Valid/ready stream (master / slave) | `m_*` / `s_*` | `m_valid`, `m_ready`, `m_data` ↔ `s_valid`, `s_ready`, `s_data` |
| Memory port bundle | `m<name>_*` | `m<name>_st`, `_waddr`, `_wdata`, `_ld`, `_raddr`, `_rdata` |

```verilog
    // UPI CPU register bus (slave) — one banner documents the whole bus
    input               s_upi_we,       //! write enable
    input               s_upi_oe,       //! output (read) enable
    input  [UPI_AW-1:0] s_upi_addr,     //! address
    input  [31:0]       s_upi_din,      //! write data
    output [31:0]       s_upi_dout,     //! read data
    ...
```

### 3.3 Inter-module connection nets

The `m_`/`s_` direction prefix in §3.2 names a signal **at a module port** — it
states the port's role (master drives, slave receives) from that module's point
of view. A net in a **parent** that wires one module's master port to another's
slave port belongs to neither role, so it does not take an `m_`/`s_` prefix.
Name it for the connection it makes instead:

- **`<master>2<slave>_<bus>`** — the net carrying bus `<bus>` from module
  `<master>` to module `<slave>`. The `2` reads "to", so the producer and
  consumer are visible at the connection: `ctl2prc_data`, `ctl2prc_valid` wire
  `u_foo_ctrl`'s master port to `u_foo_proc`'s slave port.
- **`<master>_<bus>`** — the simplified form, used when the source alone is
  enough to identify the net unambiguously (a single consumer, or a fan-out that
  no specific destination owns): `sch_rd_req`, `rr_rd_id`.

Rules:

- Drop the `2<slave>` part only when it adds no information; keep it whenever two
  or more nets share the same `<master>_<bus>` root and would otherwise collide.
- `<master>` / `<slave>` are the instance/block short names, not the full module
  names — match the functional sub-block prefixes used elsewhere (§3, e.g. `sch`,
  `rr`, `ctl`, `prc`).
- These nets are wires in the parent and never carry `_ff` (same as bus members,
  §3.2). The producing module registers internally and drives the net from its
  master port.
- The port on each end still follows §3.2: the master drives `m_<bus>_*`, the
  slave receives `s_<bus>_*`. Only the connecting net uses the `<master>2<slave>`
  form.

```verilog
    // parent wires u_foo_ctrl (master) → u_foo_proc (slave)
    wire [DATA_W-1:0] ctl2prc_data;                       // ctl drives, prc receives
    foo_ctrl u_foo_ctrl ( .m_data (ctl2prc_data), ... );  // master port: m_*
    foo_proc u_foo_proc ( .s_data (ctl2prc_data), ... );  // slave port:  s_*
```

## 4. Section banners

Break code into labelled sections with a boxed comment, indented to body level
(4 spaces):

```verilog
    //==========================================================================
    // scheduler
    //==========================================================================
```

Use a single `//===` line or a `//----` divider for sub-sections.

## 5. Clocked process style

- One clock `posedge clk`. Use **synchronous reset** compared against `RS_LV`
  (never a hardcoded polarity):

```verilog
    always @(posedge clk) begin
        if (rst_n == RS_LV) begin
            cal_req_ff      <= 'd0;
            cal_req_id_ff   <= 'd0;
        end else begin
            cal_req_ff      <= sa_rd_valid & sa_rd_ready;
            cal_req_id_ff   <= sa_rd_id;
        end
    end
```

- Use non-blocking `<=` in sequential blocks. Write reset values as `'d0` / `'b0`.
- Use **one `always` block per logical group of registers** (often one block per
  signal or per pipeline stage), not one giant block.
- Column-align LHS targets and the `<=` operators.

## 6. Combinational logic

- Use continuous `assign` for output glue and simple combinational nets.
- Prefer declaring a wire with its driving expression inline (declare + assign),
  aligned in a column:

```verilog
    wire                rr_rd_req = (|(rr_cur_req_pending)) & rr_nstall;
    wire [ID_W-1:0]   rr_rd_id  = {rr_loop_ff, rr_low_id_bits};
```

- Write multi-way muxes as vertically-aligned ternary chains, with `?`/`:` and
  the trailing `;` each on their own aligned column:

```verilog
    assign mem_qr_waddr     = ip_init   ? ip_cnt
                                        : req_id
                                        ;
```

- Split multi-bit concatenations one field per line, leading-comma style:

```verilog
    assign mem_qr_wdata     = ip_init   ? 'd0
                                        : { req_c
                                          , req_r
                                          , req_q
                                          }
                                        ;
```

- Lead multi-line boolean expressions with the operator and align:

```verilog
            ar_wr_pending_req_ff    <= (ctrl_wr_req)
                                    && (  (!rq_running_ff)
                                       || (ar_rd_id_ff != ctrl_wr_id)
                                       )    /* check ID is not pending at output */
                                    ;
```

## 7. `case` / `casez` usage

- For priority/one-hot decoders, use `casez` with `?` don't-cares and `_` digit
  separators. **Always include a `default :` arm.**

```verilog
            casez (rr_cur_req_pending)
                4'b???1 : rr_low_id_bits = 'd0;
                4'b??10 : rr_low_id_bits = 'd1;
                4'b?100 : rr_low_id_bits = 'd2;
                4'b1000 : rr_low_id_bits = 'd3;
                default : rr_low_id_bits = 'd0;
            endcase
```

- Align the `:` and value columns. Echo the case selector in a comment after
  `endcase` (e.g. `endcase // upi_addr_ff[2:0]`).

## 8. `generate` constructs

- Use `generate`/`endgenerate` with `if/else` on parameters for structural
  choices (e.g. `ENA_IFG`, `REG_IN`).
- Name generate-for loops with a `begin : <label>` block name; declare
  `genvar gi;` once near the top of the module.

```verilog
        for (gi = 0; gi < NE; gi = gi + 1) begin : sa
            fgbw_cal_sae #( ... ) isae ( ... );
        end
```

- In every branch, keep all signals defined (drive disabled-branch signals to a
  known value) so no net is left floating.

## 9. Module instantiation

- Use **named connections only** (`.port(signal)`) — never positional.
- Column-align parameter overrides and port connections, port name on the left,
  signal on the right:

```verilog
    fgbw_cal_timer #(
        .TS_W  (TS_W),
        .RS_LV (RS_LV)
    ) itimer (
        .clk     (clk),
        .rst_n   (rst_n),
        .active  ('d1),
        .timeslot(tmr_timeslot)
    );
```

- Give instances a descriptive name with an `i` prefix (e.g. `itimer`,
  `icontroller`, `ischeduler`) — not an `_inst` suffix.
- Tie off unused output ports with an inline comment placeholder:
  `.stat_count (/*sa_stat_count*/)`.
- Use the shared `vb_sdp_ram_pp` (simple dual-port RAM) as the standard storage
  element, instantiated with `RAM_TYPE("AUTO"), PIPELINE(0), INPUTREG("OFF")`.
  Use `vb_ddelay` for tapped delay lines.

## 10. Comments & documentation

- Use the Doxygen-style `//!` marker on port and parameter doc-comments (enables
  automated documentation extraction). These are **permanent** documentation and
  are kept. Do not confuse them with `//@` design-direction markers, which are
  temporary scaffolding removed by `vfill` before completion (see §18).
- Use end-of-line `/* ... */` block comments for design rationale.
- Document pipeline architecture up front in a comment block describing each
  stage:

```verilog
    // scheduler pipeline has 4 stages
    // stage 0: accept request, read data from ram.
    // stage 1: retime for next process
    // stage 2: write back
```

## 11. Lint discipline

- Apply Verilator lint waivers narrowly around the offending line, always paired:

```verilog
    /* verilator lint_off WIDTHEXPAND */
    wire [NID-1:0]  req_pending_ze = req_pending_ff;   // zero extension
    /* verilator lint_on WIDTHEXPAND */
```

- Code must pass the Verilator lint flow. Common narrow waivers: `WIDTHEXPAND`,
  `WIDTH`, `UNUSEDSIGNAL`.

## 12. Reset & initialization

- Parameterize reset polarity via `RS_LV` so the same RTL can reset active-high
  or active-low; pass a single `RS_LV` down the hierarchy. The project default is
  `RS_LV = 0` (active-low). A testbench must drive reset at the DUT's own `RS_LV`
  default — derive the bench reset level from the DUT, never hardcode it — or it
  may hold the DUT out of reset.
- Reset counters that must never start at 0 to `'d1` (e.g. clock-period counters).
- For pre-clearing distributed/block RAM, use the **init-engine** pattern: walk
  an address counter (`ip_init` / `ip_cnt`) at start-up, with RAM write muxes
  selecting `ip_cnt`/`'d0` while `ip_init` is asserted.

## 13. Module hierarchy & data flow

- Decompose strictly: a wrapper module contains only wire declarations, child
  instances, and thin glue logic.
- Use a **valid/ready** (`*_valid` / `*_ready`) handshake on streaming
  interfaces; name buses per §3.2.
- Name pipelines per §3.1.

---

## 14. Correctness & safety rules

These prevent bugs — a reviewer should block a merge that violates them.

- **Put `` `default_nettype none `` at the top of every file**, restored with
  `` `default_nettype wire `` at the end. This makes an undeclared/typo'd signal
  a compile error instead of a silent 1-bit wire.
- **Blocking vs non-blocking is a hard rule:** non-blocking `<=` in every
  `always @(posedge clk)`; blocking `=` only in `always @*`. Never mix them in
  one block.
- **No inferred latches.** Every `always @*` must drive each output on every
  path — give combinational regs a default assignment at the top of the block,
  and include `default :` in every `case`/`casez`.
- **`casez` hazard:** `casez` against *input* data treats `z`/`?` bits in the
  data (not just the pattern) as don't-cares, masking bugs. Restrict `?` to the
  case *patterns*; use plain `case` when no don't-cares are needed.
- **Reset scope is explicit:** reset all control/state flops and anything that
  gates a handshake (`*_valid`, FSM state, counters). A wide datapath register
  always qualified by a valid may be left un-reset — but comment that choice.
- **No multi-driver nets.** Drive each signal from exactly one `always`/`assign`.
- **No magic numbers in logic.** Give every non-trivial constant a named
  `parameter`/`localparam`; size bit-width literals (`8'd0`, not `0`) wherever
  the inferred width is not obvious. Width-inferred `'d0` / `'b0` in reset and
  assignment contexts is fine — there the LHS sets the width (see §5).
- **Never let a clear race the capture of a non-repeating event.** When a *one-shot*
  event — a single-cycle pulse or one edge — is latched into a state bit that some
  other agent can clear, a clear that lands on the same cycle as the set destroys the
  event **permanently**, because a one-shot source never fires again. (Contrast an
  *idempotent* source: a level missed on one cycle is still asserted the next, so there
  is no race — this is why levels are safe and one-shots are not.) It is one hazard with
  many faces: a write-1-to-clear **pending-interrupt** bit vs. a new IRQ edge (the
  classic *lost interrupt*); a **sticky error/status** bit vs. its clear; a pulse/req
  captured across a **req-ack handshake** vs. the ack-clear; and the sneakiest, a
  **clear-on-read** "done"/event flag *polled by reading* — the read itself is the
  racing clear, so it is not even a visible write, and the poll hangs forever. Fix it at
  one of two layers: **(1)** in the latch, give the **set (event capture) priority over
  the clear**, so a coincident clear only *defers* and never drops the event (this is
  exactly the W1C interrupt-controller rule); and/or **(2)** capture the event into, and
  expose, the underlying **level**, and let consumers sample that. Reserve clear-on-read
  / W1C one-shots for events where a miss is acceptable; never make the *only* signal of
  a must-not-miss event a racy one-shot. (The sticky/W1C latch and a worked example are
  `DesignPatterns.md` P5.)

## 15. Clock-domain crossing (CDC)

- A signal crossing clock domains **must** pass through an explicit
  synchronizer; never sample a foreign-domain signal directly into logic.
- Single-bit control crossings: use a 2-flop synchronizer with `_meta_ff` /
  `_sync_ff` suffix so it is visible in review and constrainable in timing.
- Multi-bit crossings: use a gray-coded pointer (async FIFO) or a
  request/acknowledge handshake — never a bit-parallel 2-flop sync on a bus.
- Document each CDC path in a comment and give it the matching timing constraint
  (`set_false_path` / `set_max_delay`).

## 16. Finite state machines

- Hold state in a `*_state_ff` register; compute the next state in a single
  combinational `*_state_ns` `always @*` with a `default :` arm.
- Use named `localparam` (`UPPER_SNAKE`) state encodings, never bare numbers.
- Keep output logic separate from the next-state block (registered from state,
  or its own combinational block).

## 17. Verification & simulation hooks

- **Assertions live in the testbench `.sv`, never in RTL** (see §0 — RTL is
  Verilog-2005 only, which has no `assert`). Check design intent (one-hot,
  no-overflow, handshake legality) with SystemVerilog assertions bound to the
  DUT from the testbench, not inside the `.v` design files.
- In RTL, the only permitted simulation-only code is Verilog-2005 debug output
  (`$display` / `$write`; not `$error`/`$fatal`, which are SystemVerilog), and
  it **must** be guarded by `` `ifdef SIM `` … `` `endif `` so it never reaches
  synthesis.

## 18. Design Direction Markers (`//@`)

`//@` comments are design-direction markers: instructions that tell `vfill` what
logic to implement. They are temporary scaffolding, not permanent documentation.
Distinct from the `//!` Doxygen doc-comments of §10 (`//!` is permanent and kept;
`//@` is removed once the logic is implemented) — the two markers never overlap.

### Syntax

- A single-line comment whose leading token is `//@`. Write it as `//@ <direction>`.
- One direction per line, placed on its own line in the code section where the
  described logic belongs.

### Content

- Plain-English design intent only — describe *what* to implement, never *how*.
- A marker must never contain Verilog code: no assignments, expressions, or
  operators that form valid RTL. Refer to signals by name in prose instead.

### Lifecycle

- **Created by** `vdesign` in `rtl/<module>.v` backbones, and by `varch` as
  integration notes in `rtl/<ip>_top.v`.
- **Consumed by** `vfill`: once the described logic is implemented, `vfill` must
  convert every `//@` to a normal `//` comment (or remove it). No `//@` may
  remain in a completed module.
- `vflow` detects implementation completion (Phase 3) by the **absence** of
  `//@` in `rtl/` excluding `_top.v`. A single stray `//@` keeps the module
  marked IN PROGRESS, so an unconverted marker silently blocks the flow.
- Integration markers in `rtl/<ip>_top.v` are excluded from Phase 3 detection;
  they are resolved during IP integration, not during submodule fill.

## 19. Testbench Verdict Token

*(testbench `.sv` only)*

Every `vtestgen` testcase ends by printing exactly one verdict token, then
`$finish`:

- `[FINISH] PASS` when the testcase passed.
- `[FINISH] FAIL` on any mismatch or timeout. The shared `cpu_read_check` task
  emits `[FINISH] FAIL` automatically on a read mismatch.

This token — not the presence of any `[ERROR]` text — is the single
machine-readable result. The bench Makefiles and `vtestrun` decide pass/fail by
grepping for `FINISH` and reading the following word; absence of the token is
treated as FAIL (e.g. the bench crashed or never reached the result line).
Diagnostic context is printed on `[ERROR] time= ... ` lines, which `vtestrun`
harvests into issue reports but does not use for the pass/fail decision.

## 20. Do not

- **Do not leave dead or alternative code as commented-out blocks.** Delete it
  (git is the history), or — if it is a real build-time choice — guard it with a
  `parameter`/`` `ifdef ``.
- **Do not** use positional port connections, hardcoded reset polarity, magic
  numbers, or mixed blocking/non-blocking in one block (see above).

## 21. Enforcement & automation

- **Formatting** (alignment, banners, port order, indentation) → `verible-verilog-format` with a checked-in config, not by hand.
- **Lint rules** (`_ff` on flops, `RS_LV` last, named-only ports, `default_nettype`, no-latch, `default :`) → `verible-verilog-lint` + the Verilator flow.
- Run both in CI / a pre-commit hook so new files are checked at submit time.
