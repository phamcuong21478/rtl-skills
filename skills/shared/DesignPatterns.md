# RTL Design Pattern Cookbook

A curated set of **vetted micro-patterns** for the recurring structural problems
that come up while designing (`vdesign`) and implementing (`vfill`) submodules.

The patterns themselves are **common, general RTL idioms** — a skid buffer, an
arbiter, a sticky bit are not unique to this project. What is project-specific is
the **Ruling** on each: which shape this codebase standardizes on, and the local
decisions (`vb_sdp_ram_pp`, the `s_upi_*` bus, `rr_`/`sch_` naming, single-clock
only) that general knowledge would not reproduce. So the title is deliberately not
project-branded; the rulings carry the project flavor, the patterns do not.

This file is **not** a Verilog tutorial — the implementer can already write an FSM
or a skid buffer. Its job is to **canonicalize** one shape per recurring problem
(so two modules solve the same problem the same way, across runs and across
models) and to record the **project-specific rulings** that general Verilog
knowledge would not reliably reproduce. Each entry is therefore terse: the
canonical shape and the one or two non-obvious decisions, not a full code listing.

It is the companion to `CodingStyle.md`: `CodingStyle.md` says *how to write a
line*; this file says *what structure to build* and *which decision to make at the
fork*.

## Scope

These patterns target this project's hardware: **FPGA, single-clock, synchronous
reset** (see the commercial-IP scope). Everything here lives in **one clock
domain**.

- **No CDC patterns here.** A genuine multi-clock crossing is out of scope for this
  IP family; if one is ever truly needed it is an architecture decision, and the
  synchronizer rules live in `CodingStyle §15`. **Do not** add a 2-flop sync or
  gray pointer "to be safe" inside a single clock — it only adds latency and hides
  bugs.
- All sequential idioms use synchronous reset against `RS_LV` (`CodingStyle §5`,
  `§12`) and the `_ff` suffix (`§3`).
- **A slow "clock" stays in this domain too.** A low-rate strobe (a 1 kHz tick, a
  divided enable) is a *signal*, not a clock — sample it, edge-detect it (P7), and
  gate logic with the resulting one-cycle enable. Never route it to a second
  `always @(posedge …)` — that would create the very clock domain this scope rules
  out.

## How the skills use this file

- **`vdesign`** — when a requirement matches a pattern, **name the pattern in the
  proposal's architecture section** (e.g. "the per-id count RAM needs P4 preclear +
  P8 forwarding") and
  shape the backbone `//@` markers after its canonical structure. Markers stay
  plain-English, no code (`CodingStyle §18`). Naming the pattern is the handshake
  that tells `vfill` which shape to build.
- **`vfill`** — when the proposal names a pattern (or the backbone is clearly one),
  build that pattern's canonical shape and honor its **Ruling**, rather than
  picking a different valid shape. Adapt names, widths, and reset scope per module.

## Pattern → problem index

| Pattern | Solves |
|---------|--------|
| [P1 Stream decoupling](#p1-stream-decoupling--buffering--use-the-lib-fifo) | decouple/buffer a `valid`/`ready` stream — instantiate `vb_stream_fifo` from lib, don't hand-roll |
| [P2 Arbiter](#p2-priority--round-robin-arbiter) | pick one of N requesters, fixed-priority or fair |
| [P3 Counter + strobe](#p3-counter-with-wrap--strobe) | period/terminal counts, divide-by-N, address walk |
| [P4 RAM preclear](#p4-ram-preclear-init-engine) | bring RAM to a known state after reset |
| [P5 Sticky / W1C bit](#p5-sticky--w1c-status-bit) | latch a must-not-miss one-shot event |
| [P6 CSR decode](#p6-csr-register-file-decode) | map a CPU/UPI bus onto config/status registers |
| [P7 Pulse ↔ level](#p7-pulse--level-conversion) | edge-detect / strobe / stretch in one clock |
| [P8 RAM RDW forwarding](#p8-ram-read-during-write-forwarding) | bypass stale RAM data when a pipeline reads an address it is also writing |
| [P9 Resource-type select](#p9-resource-type-selection-by-size-threshold) | pick flops/LUT-RAM/block/ultra by depth at elaboration, derive read-latency from it |
| [P10 Latency delay line](#p10-latency-matching-side-band-delay-line) | align side-band/control with datapath or memory read latency |
| [P11 Up/down counter](#p11-updown-occupancy-counter) | track occupancy that fills and drains from two events |
| [P12 Port arrays as buses](#p12-port-arrays-as-flattened-buses) | pass an array of signals through a Verilog-2005 port (no array ports) |
| [P13 Persistency](#p13-persistency--integrate-before-declare) | declare a status only after a condition persists N samples (defect integration, debounce, lock) |
| [P14 LFSR / PRBS](#p14-lfsr--parameterized-prbs) | polynomial bit sequence — scrambler, PRBS, CRC |
| [P15 Wide-word reduction](#p15-combinational-wide-word-reduction) | fold a wide bus to a scalar — parity, popcount, error count |
| [P16 Windowed accumulator](#p16-windowed-accumulator--performance-counter) | count/sum a per-key quantity over a time window — perf counters, rate, error counts |

(A small Moore FSM is not listed as a pattern — its shape and rulings are fully
fixed by `CodingStyle §16`; build it straight from there.)

## Reflex triggers — patterns you apply from a *structural* choice

Some patterns fire not from what the module does but from a building block the
design uses. The requirement won't ask for them by name, so they are the
most-often-missed. Check these the moment the architecture commits to the block on
the left:

| The moment the design commits to… | …apply |
|---|---|
| a RAM/array read before it is guaranteed written | P4 (preclear) |
| a RAM read-modified-written (counter/accumulator/table by key) | P8 (RDW forwarding) |
| on-chip storage whose size should drive the primitive choice | P9 (resource select) → P10 (latency match) |
| a pipelined / latency-bearing datapath carrying side-band signals | P10 (delay line) |
| the module sitting on a CPU/UPI register bus | P6 (CSR decode) |
| a slow strobe / divided-clock *input* | P7 (edge-detect enable) |
| an array of signals crossing a module port (Verilog-2005) | P12 (flattened bus) |

The feature-driven patterns (P1 stream decoupling, P2 arbiter, P3 counter, P5 sticky, P11
occupancy) are recognized instead from each entry's **Recognize when** line.

---

## P1. Stream decoupling / buffering — use the lib FIFO

Decouple a `valid`/`ready` stream, break a backpressure path, or absorb a burst.

- **Recognize when.** A streaming `valid`/`ready` interface needs buffering,
  elastic decoupling, full-throughput backpressure with a registered `ready`, or a
  small queue between two rates.
- **Ruling — don't hand-roll it.** Instantiate **`vb_stream_fifo`** from `lib`
  (`wr_valid`/`wr_ready` ↔ `rd_valid`/`rd_ready`, `stat_*`) at the depth you need;
  this is a parameterized, instantiable block, so it belongs in `lib`, not as an
  idiom here (same reasoning as the `hc` comparator). A skid buffer / depth-2
  decoupler is just this FIFO at small depth — don't re-derive one.
- **The only hand-rolled case.** A single **register slice** to break a long
  combinational path when a one-cycle bubble on stall is acceptable:
  `s_ready = !m_valid_ff || m_ready`, one `*_ff` stage. Anything that must hold a
  beat under backpressure → the lib FIFO.

## P2. Priority / round-robin arbiter

N requesters, one grant per cycle. Fixed priority = a `casez` lowest-index encoder;
round-robin = the same encoder with a rotated request vector.

- **Recognize when.** Two or more requesters compete for one shared output/resource
  and exactly one must be served per cycle. "Fair" / "no starvation" / "round-robin"
  → RR; "strict priority" → fixed.
- **Shape.** Fixed: `casez (req)` lowest-set-bit → binary index, always a
  `default` arm (`CodingStyle §7`). RR: rotate `req` right by `rr_ptr_ff`, encode,
  add `rr_ptr_ff` back mod N; on accepted grant set `rr_ptr_ff = gnt_id + 1`.
- **Ruling.** `?` belongs to the *pattern* only — never let `req` carry `x`/`z`
  into `casez` (`§14` casez hazard). Grant must be qualified by downstream `ready`,
  or you grant into a stall. Match the existing `rr_`/`sch_` block naming.
- **Wide vectors — group, don't build one giant encoder.** For many requesters,
  split the request vector into fixed-size slices (`GSIZE`), keep a group pointer
  `rr_loop_ff` that round-robins across the slices, and priority-encode *within* the
  selected slice with a bounded `casez`. The grant index is `{rr_loop_ff,
  low_bits}`; advance `rr_loop_ff` when the current slice is empty. This keeps the
  encoder size fixed as N grows.

## P3. Counter with wrap & strobe

Period counters, divide-by-N, address walks; wraps at a limit and emits a
one-cycle strobe.

- **Recognize when.** The spec asks for a periodic event, divide-by-N, time-slot /
  sample index, or a sequential address walk — anything that increments and wraps. A
  framing signal that restarts the count → the reload-on-event variant.
- **Shape.** `cnt_ff <= at_last ? 'd0 : cnt_ff + 'd1` under an enable;
  `tick = en && (cnt_ff == LIMIT)`.
- **Ruling.** `LIMIT`/width come from a `parameter`/`localparam`, never a magic
  number (`§14`). A period counter that must never read 0 resets to `'d1`
  (`§12`). A run-time limit is registered through P6 and compared against the
  registered copy.
- **Variant — sync reload-on-event.** A phase/sequence counter often reloads to a
  known value on a framing event (`sob`, restart) instead of only wrapping:
  `if (event) cnt_ff <= RELOAD; else if (en) cnt_ff <= ...`. Reset *and* the reload
  value are both explicit — don't assume the event aligns with the wrap.

## P4. RAM preclear init engine

Block/distributed RAM powers up undefined; logic that reads-before-write needs a
known initial state. This is the `ip_init`/`ip_cnt` engine of `CodingStyle §12`.

- **Recognize when.** *(reflex — see the trigger table)* A RAM/array is read before
  it is guaranteed written (valid bits, tags, per-id state), or the spec says entries
  are empty/zero after reset, or there is a flush/re-init command.
- **Shape.** Out of reset, walk `ip_cnt_ff` 0→DEPTH-1 with `ip_init_ff` asserted;
  while `ip_init_ff`, the RAM write port selects `ip_cnt`/clear-value over the
  functional write; clear `ip_init_ff` on the last address.
- **Ruling.** Hold off all functional traffic and any `ready` until `init_done`,
  or the first real write races the preclear. Use the standard `vb_sdp_ram_pp`
  storage element (`§9`).
- **Runtime re-init.** Drive the same walk from a request port (`ctrl_init_req`),
  not only from reset, when the IP must be re-cleared on command — gate the request
  the same way reset does (`if (rst || init_req) start the walk`).
- **One walk, many RAMs.** A single address counter can preclear several RAMs of
  *different* depths: give each RAM its own range-limited store-enable (one stops at
  `NO_ID-1` while the walk runs to `NO_ENTRY-1`). Mux each write port between the
  init source and the functional source — and **register that mux** if the RAM is
  pipelined, so the init address/data arrive in the same cycle relationship as
  functional writes.
- **Centralized broadcast across the hierarchy.** The walk can live in one block
  (often the register file) and fan out as `ip_init` / `ip_cnt` *ports* to every
  submodule; each submodule runs its own local preclear off the shared signals.
  Then the engine is built and re-triggered once, not re-implemented per module —
  the submodules just consume the broadcast.
- **Clear value is any reset constant.** The preclear value need not be 0 — it is
  whatever the spec defines as the reset state of an entry (e.g. an LFSR seed
  `INITIAL_VALUE`, an all-ones tag, a count of `'d1`). Mux that constant in during
  the walk the same way.

## P5. Sticky / W1C status bit

Latch a one-shot event (error edge, "done" strobe) for software to observe, cleared
on read or write-1-to-clear.

- **Recognize when.** A transient event (error, overflow, "done", an interrupt
  source) must be held for a later software/observer read, or the register map lists
  a `W1C` / read-to-clear / interrupt-status bit.
- **Shape.** `if (event_set) status_ff<=1'b1; else if (w1c_clear) status_ff<=1'b0;`
- **Ruling.** **The set arm must take priority over the clear** — this ordering is
  the whole pattern; swapping the arms reintroduces the lost-event bug (`§14`
  lost-interrupt rule). A coincident clear must only *defer*, never drop. For a
  clear-on-read flag the read itself is the racing clear, so for a must-not-miss
  event expose the underlying **level** for polling instead of a racy one-shot.
  *Worked example:* an SM4 `CFG_DONE` set on a one-cycle busy-fall and cleared on
  read could be lost by a coincident poll-read — the fix was set-priority in the RTL
  latch *and* polling the `CFG_BUSY` **level** in software (both layers above).
- **Variant — set-once arm latch.** A "have I seen the first event yet" flag that
  latches high and never clears (until reset) is the degenerate case with no clear
  arm: `arm_ff <= arm_ff || event`. Use it to enable an output path after the first
  valid beat; the same set-priority reasoning makes a missed first event impossible.
- **Variant — per-key status word in RAM.** For one sticky bit (or a few) *per
  channel/id*, hold the status word in RAM instead of a flop: read it, `set = old |
  set_mask`, `clear = old & ~clr_mask`, write back — with P8 forwarding for
  back-to-back same-id access and P4 preclear. The set-over-clear priority still
  applies per bit. This is the scoreboard/pending-vector form.

## P6. CSR register-file decode

A CPU/UPI slave bus (`s_upi_*`, `CodingStyle §3.2`) writes config and reads back
config + status.

- **Recognize when.** *(reflex — see the trigger table)* The module exposes
  configuration or status over a CPU/register bus — a register map, address offsets,
  `s_upi_*` / AXI-Lite / APB.
- **Shape.** Per-register address `localparam`s; capture `s_upi_din` into the
  matching `*_ff` on a write strobe at its address; readback via an aligned ternary
  chain selecting the addressed register.
- **Ruling.** Reserved/unmapped reads return a **defined 0**, not a latched bus.
  RO registers reject writes. Status registers with W1C use P5. The map must stay
  byte-identical to the doc's register table — `vvplan`/`vpackage` trace against it.
- **Base-address decode (mandatory for a composable block).** Claim a transaction
  only when the upper address bits match a `BASE_ADDR` parameter:
  `up_ce = (s_upi_addr[hi:ADDR_BIT] == BASE_ADDR[hi:ADDR_BIT])`. Gate every write
  capture and the readback with `up_ce`, and **drive `s_upi_dout` to 0 when the
  block is not selected** — otherwise two blocks on one bus both drive read data.
- **Write-strobe edge.** The bus write-enable is a level; derive a one-cycle write
  pulse from it (`st <= up_ce & s_upi_we & ~s_upi_we_ff`, P7) and qualify all
  captures with that pulse, so a held `we` does not re-write every cycle.
- **Reset to the spec default, not blind 0.** Config registers reset to their
  documented default, often a parameter-derived value (`rf_period_1s_ff <=
  N_CYCLE_1S`), so the block is usable before software configures it.
- **Large / pipelined map.** When the map is wide or RAM-backed, register the
  readback in stages (accept → decode → read → mux) instead of one combinational
  ternary, for timing closure; each stage still has a `default`/0 arm (`§7`).
- **Register arrays via range decode.** Map an address *range* to a RAM-backed
  per-entry array (`s_upi_addr[hi:IDX_BIT] == ARRAY_BASE[hi:IDX_BIT]`, low bits =
  index) rather than N individual flops, when there are many like-shaped entries
  (per-channel config). The array RAM follows P4 (preclear) and P8 (read-during-
  write) like any other.

## P7. Pulse ↔ level conversion

Edge-detect, level→strobe, pulse-stretch, all within one clock. (Cross-domain
versions are CDC — out of scope, see Scope.)

- **Recognize when.** You must act once on a transition ("on the rising edge of",
  "when X asserts"), produce a single-cycle strobe from a level, or consume a
  slow/divided clock or low-rate tick as an *input*.
- **Shape.** Register the level one cycle; `rise = lvl & ~lvl_ff`,
  `fall = ~lvl & lvl_ff`. Stretch = P3 counter armed by the strobe.
- **Ruling.** Reset the history flop so the first cycle out of reset cannot
  manufacture a phantom edge. If "single-cycle" matters downstream, assert the
  pulse width in the bench.
- **Use it for slow strobes / divided "clocks".** This is how a low-rate enable
  (a 1 kHz tick, a divided clock) is consumed in a single-clock design: edge-detect
  the incoming level and gate the slow logic with the one-cycle `rise` pulse —
  never feed the strobe to a second clocked block (see Scope).

## P8. RAM read-during-write forwarding

When a pipeline reads a RAM address it may also be writing — a per-key value
read-modified-written by the same or a parallel pipeline — a `vb_sdp_ram_pp` with
registered output returns **stale** data on that cycle, so the read must be
forwarded. (Only relevant to read-modify-write-on-RAM designs; a plain delay or
FIFO that never re-reads a live address does not need it.)

- **Recognize when.** *(reflex — see the trigger table)* Per-key/per-channel state
  lives in RAM and is updated from its own current value (a counter, accumulator, or
  status table indexed by id) — especially if updated every cycle or from two
  pipelines.
- **Shape.** One cycle, register the conflict and capture the write data:
  `hz_ff <= st & ld & (raddr == waddr)`. Next cycle the consumer selects
  `hz_ff ? captured_wdata : m_*_rdata`. Name the flag `hz_<res>_p<X>_p<Y>_ff`
  ("resource hazard between stage X and stage Y", `CodingStyle §3.1`).
- **Ruling.** Match the number of forwarding stages to the RAM's **read latency**:
  a `PIPELINE=N` RAM needs a bypass at every in-flight stage (`p3_p2` *and* `p4_p2`,
  the deeper one gated by the pipeline-enable param). Qualify the address compare
  with **both** the write strobe and the read enable — comparing addresses alone
  forwards on idle cycles. Two pipelines sharing one RAM-backed value forward across
  each other the same way (this pipeline's read stage vs the other's write stage).
- **When you don't need it.** Forwarding is only required if the *same address* can
  be re-read within the RAM's latency window. If an upstream scheduler guarantees a
  minimum spacing between accesses to one key (e.g. a frame scheduler that never
  revisits an id within N cycles), the plain RAM read is safe — don't add the bypass.
- **Two writers: forward or block.** When two pipelines (or a functional path and a
  background scan) may write the *same* location, either forward the in-flight value
  or **block the lower-priority writer** for a couple of cycles (`co_blk <=
  same_id_recent`). A shared write port driven by several sources takes a
  priority-ordered mux on `st`/`waddr`/`wdata` (`init > A > B`).
- **CPU read shares the port.** A CPU status read of per-key state usually
  multiplexes onto the *functional* read port at lower priority
  (`raddr = func_active ? func_id : cpu_id`), stealing idle cycles rather than
  adding a second read port.

## P9. Resource-type selection by size threshold

Pick the FPGA storage primitive (flops / LUT-RAM / block / ultra) from the depth at
elaboration instead of hardcoding one.

- **Recognize when.** *(reflex — see the trigger table)* The design needs on-chip
  storage whose ideal implementation depends on depth, or a depth-parameterized
  buffer/FIFO/delay that must stay efficient from a few entries up to thousands.
- **Shape.** A `localparam` ladder: an explicit `TYPE` override wins, else `"AUTO"`
  maps depth to a primitive by threshold (e.g. `<=4 → REGISTER`,
  `<=64 → DISTRIBUTED`, `<=1024 → BLOCK`, else `ULTRA`); a paired `generate` builds
  the chosen implementation.
- **Ruling.** **Derive the read-latency parameter from the chosen primitive**
  (flops/LUT-RAM = 0, block = 1, ultra = 3, plus any extra requested pipeline) and
  propagate it, so the latency-matching delay line (P10) and any read-during-write
  forwarding (P8) stay correct when the primitive changes. Keep the override port so
  a caller can force a type, and a defined error/zero branch for an illegal size.
  Drive every output in all `generate` branches (`§8`).

## P10. Latency-matching side-band delay line

Carry control/side-band signals (`valid`, `sob`, id, status) alongside a datapath
that has pipeline or memory latency, so they arrive aligned.

- **Recognize when.** *(reflex — see the trigger table)* Control/side-band signals
  (`valid`, `sob`, id, status) must stay aligned with a datapath that has pipeline or
  memory read latency, or you need a fixed N-cycle delay of a data bus.
- **Shape.** A shift register — packed reg array `reg sig_ff [1:DEPTH]`, `s<X>t<Y>`
  naming (`CodingStyle §3.1`) — whose **DEPTH = the datapath/memory's latency
  parameter**; `generate`-bypass to a wire at DEPTH 0.
- **Ruling.** Tie the depth to the *same* parameter that sets the datapath latency
  (P9's derived `*_PIPELINE`), never a separately hand-counted constant — they must
  move together or the side-band slips against the data. For a long *data* delay
  (not just side-band), build the line in RAM rather than flops and **offset the
  write pointer by the RAM's read latency** (`waddr = raddr + (TAP - RAM_PIPELINE)`)
  so the net delay stays exactly `TAP`; choose flops-vs-RAM by depth per P9. Keep
  outputs driven in both `generate` branches (`§8`).
- **Variant — configurable delay knob.** A `REG_IN` / `DLY_IN` (0…N) parameter that
  inserts that many register slices on a path purely for timing/floorplanning,
  behavior-invariant, via a `generate` ladder. When you turn the knob, **every
  parallel lane and the downstream latency-match must move by the same N** — the
  whole point of the alignment ruling above. Common as `ENA_DLY_IN/OUT`, `REG_IN`,
  `DLY_IN/DLY_OUT`.

## P11. Up/down occupancy counter

Track occupancy of a shared resource that fills from one event and drains from
another.

- **Recognize when.** You must report how full/occupied a shared resource is, filled
  by one event and drained by another (queue/FIFO/buffer occupancy, in-flight count,
  credit count, "number of entries").
- **Shape.**
  `if (inc && !dec) cnt_ff <= cnt_ff + 'd1; else if (!inc && dec) cnt_ff <= cnt_ff - 'd1;`
  with `full`/`empty` derived from `cnt_ff`; register the `inc`/`dec` conditions
  first.
- **Ruling.** Handle the **simultaneous inc+dec** case explicitly — it must net to
  zero, which the `if / else if` above does but a pair of independent `+`/`-` does
  not. Reset `empty` true and `full` false. If the resource is RAM-backed, re-init
  the count alongside the preclear (P4). When two independent pipelines would
  contend to write one RAM-backed per-key count, the alternative is to split it into
  a producer counter and a consumer counter and take occupancy as their difference
  (then P8 forwarding covers each side's read-modify-write) — but the single up/down
  counter here is the default; reach for the split only under that specific
  dual-writer constraint.

## P12. Port arrays as flattened buses

Verilog-2005 has no unpacked-array ports, so an array of like signals (per-channel
ids, per-lane valids) crosses a module boundary as one wide bus.

- **Recognize when.** *(reflex — see the trigger table)* A module port must carry an
  *array* of like signals (per-lane valids, per-channel ids) and the RTL is
  Verilog-2005, which has no unpacked-array ports.
- **Shape.** Declare the port as a flat `[N*W-1:0]` bus; on each side rebuild the
  array with a `generate` loop and a part-select:
  `assign arr[gi] = bus[W-1 + gi*W -: W];` (and the reverse on the driving side).
- **Ruling.** Use the indexed part-select (`-:`) with the *same* `W*gi` stride on
  both producer and consumer so the lanes can't transpose. A run-time lane pick is
  then just `arr[sel]`. Keep this flatten/slice at the port only — work with the
  unpacked array internally. (This subsumes packing several *fields* into one word:
  same concat-out / slice-in discipline, fields instead of lanes.)

## P13. Persistency / integrate-before-declare

Declare a status only after a raw condition has held for several samples/frames —
defect integration (G.798 dXXX), debounce, sync/lock detection, "accepted after N
consecutive".

- **Recognize when.** The spec says "declare after N consecutive", "set after N /
  clear after M", "accepted value" (AcTI/AcDM), "persists for X frames", debounce, or
  alarm-integration / lock-detect.
- **Shape.** A windowing or consecutive-count counter (P3) gates the integration;
  **accumulate or agree** the raw condition across the window (sticky-OR within a
  frame, or a consecutive-match counter); at the threshold flip a separate declared
  `*_ff`. Per-key state lives in RAM (P4 + P8). Keep the **raw condition distinct
  from the declared state**.
- **Ruling.** Pick **N-consecutive** (the count resets to 0 on any disagreement,
  declare at N) or **N-of-M with hysteresis** (separate set/clear thresholds, or a
  persistence count that advances only while the candidate differs from the declared
  state). Sample/clear the window accumulator at the window **boundary**, not
  mid-window. **Never drive the declared output straight from the raw condition** —
  that is the whole point of integrating. Reset the declared state to the
  no-defect value.

## P14. LFSR / parameterized PRBS

A polynomial bit sequence — scrambler/descrambler, PRBS generate/check, CRC, or any
pseudo-random stream.

- **Recognize when.** PRBS-N, scrambler, "pseudo-random", CRC, or a polynomial /
  feedback shift sequence appears in the spec.
- **Shape.** Parameterize the polynomial (`POLYNOMIAL`, `POLY_LEN`) and the per-cycle
  width. Unroll the feedback in a combinational loop — `lsb = ^(lfsr & POLY);
  lfsr = {lfsr[POLY_LEN-2:0], lsb}` — once per output bit, so `DWIDTH` bits emerge
  per cycle; register the state.
- **Ruling.** Seed to a **non-zero** `INITIAL_VALUE` (an all-zero LFSR is stuck), and
  preclear per-channel state RAM to that seed via P4. For per-channel operation,
  persist the state in RAM with P8 forwarding (back-to-back same channel). Keep the
  polynomial a *parameter*, never hard-coded, so one block serves PRBS-7/23/31; when
  more than one site needs it, factor the unrolled core into its own small
  block (it is `lib`-grade, like a comparator).

## P15. Combinational wide-word reduction

Fold a wide bus down to a few bits in one cycle — parity (BIP/checksum), bit-error /
Hamming-weight count, set-bit count, OR/AND-reduce.

- **Recognize when.** BIP-N / parity / checksum, "count the errored bits", popcount /
  Hamming distance, or any reduce across a wide word.
- **Shape.** A combinational `always @*`: for **parity**, XOR-fold strided bits
  (`for jj: for ii=jj; ii<W; ii+=STRIDE: p[jj] ^= d[ii]`); for a **count**, map each
  small chunk (2-bit / nibble) to its popcount with a `case` LUT and sum the chunks
  in an adder tree. Register the scalar result.
- **Ruling.** Keep the reduction purely combinational and register only the final
  scalar — don't spread it across cycles unless timing forces it (then pipeline the
  adder tree, not the fold). Size the sum to hold the maximum (`$clog2(W+1)`). For
  interleaved parity (BIP-N) the stride *is* the interleave; note which input bits
  feed which parity bit.
- **Variant — pipelined hierarchical tree.** For a wide popcount, stage the reduction:
  nibble-LUT → per-byte sum → per-8-byte sum → word sum, one pipeline register per
  level (depth selected by `generate` on width). Down-stream consumers then align to
  that depth via P10.

## P16. Windowed accumulator / performance counter

Count or sum a per-key quantity over a time window and present the result — error
counts, rate measurement, per-second performance counters, errored seconds, TSE.

- **Recognize when.** "Count errors per second", performance counter, rate / TSE /
  EBC, errored-second, "accumulate over a window then report". (For a level that goes
  up *and* down — occupancy — use P11 instead; this is monotone-accumulate-then-latch.)
- **Shape.** Accumulate per key in RAM (read-modify-add per event, with P8 forwarding
  and P4 preclear). Produce the per-window result one of two ways:
  **snapshot-and-restart** — on a periodic tick (P7 edge of a 1 s strobe) latch the
  running total to a shadow and reset the accumulator in the same cycle; or
  **read-to-clear** — the CPU read returns the count and zeroes it.
- **Ruling.** Pick snapshot-on-tick when a periodic boundary defines the window
  (per-second PM); pick read-to-clear when software defines it. Either way the latch
  and the reset happen **together** so no event between read and clear is lost (the
  P5 set-over-clear hazard, applied to a counter). Size the accumulator for the
  worst-case window. For a *sliding* (overlapping) window, keep a running sum and
  subtract the sample leaving the window via a P10 delay line (boxcar) instead of
  resetting. Burst-prone event sources may need a small FIFO ahead of the RMW engine.

---

## Adding a pattern

Add one only when it is (a) genuinely recurring across modules, (b) single-clock
and in scope, and (c) reducible to a stable shape + a clear ruling. Keep new
entries terse — canonical shape and the non-obvious decision, not a code listing;
cite `CodingStyle` rather than restating it. A one-off does not belong here; a
parameterized, instantiable block belongs in the `lib/` library (with its own
`ModuleDocContract` doc), not as an idiom.
