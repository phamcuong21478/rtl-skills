---
name: vtestgen
description: Generate a comprehensive set of self-checking IP-level testcases, validate each with a self-check, and write a testcase list
---

# Verilog Testcase Generator

## Overview

This guide defines the workflow for generating a comprehensive set of IP-level testcases. Each testcase is a self-checking Verilog file. After writing each file, a self-check confirms structural validity before moving to the next testcase.

The input is an IP name. The outputs are:

1. `./tb/<ip_name>/files.f` — shared RTL filelist for all testcases
2. `./tb/<ip_name>/tc_<name>.v` — one file per testcase
3. `./tb/<ip_name>/tc_list.md` — table of all generated testcases

## Prerequisites

| Tool  | Purpose                          | Check             |
|-------|----------------------------------|-------------------|
| xvlog | Verilog compile / lint           | `xvlog --version` |
| xelab | Elaborator                       | `xelab --version` |
| xsim  | Simulator                        | `xsim --version`  |

## Step 1 - Understand the IP

Read the following sources before planning any tests:

1. `./doc/<ip_name>.md` — full IP behavior, architecture, interface protocols
2. `./rtl/<ip_name>_top.v` — exact top-level port list, signal names, and widths

Identify:
- All input stimulus interfaces and their handshake protocols
- All output interfaces and their expected response behavior
- All operating modes and configuration parameters
- Any documented error or edge conditions

## Step 2 - Build RTL Filelist

Walk the instantiation hierarchy of `<ip_name>_top` (same method as `vsynth` Step 2) and collect all required Verilog files.

Create `./tb/<ip_name>/files.f`. List one file per line. All paths must be relative to `tb/<ip_name>/` using `../../` prefixes.

Example:
```
../../rtl/foo_top.v
../../rtl/foo_ctrl.v
../../rtl/foo_proc.v
../../lib/fifo/fifo.v
```

## Step 3 - Plan the Test Suite

Before writing any Verilog, list all planned testcases. Organize them across the six mandatory categories:

| Category       | Coverage target |
|----------------|-----------------|
| Reset          | DUT reaches defined idle state after reset; all outputs at documented reset values |
| Basic          | One test per documented feature or operating mode |
| Edge           | Boundary data values (0, all-ones, max-minus-1), empty/full conditions, single-cycle bursts |
| Back-pressure  | Valid/ready handshakes held off; pipeline stalls and drains correctly under sustained back-pressure |
| Error inject   | Out-of-spec stimulus, concurrent events, overflow/underflow conditions |
| Stress         | Back-to-back transactions with no idle cycles; maximum throughput for at least 1000 clock cycles |

Present the full plan to the user as a table before writing any code. Adjust based on user feedback.

## Step 4 - Write and Self-Check Each Testcase

For each planned testcase, follow this loop:

### 4A — Write the Testcase File

Create `./tb/<ip_name>/tc_<name>.v`. File name must be lowercase, using underscores, no spaces.

#### Required testcase structure

```verilog
// tc_<name>.v
// Category   : <category>
// Description: <one-line description>
`timescale 1ns/1ps

module tc_<name>;

// ── Parameters ────────────────────────────────────────────────────────────────
localparam CLK_PERIOD = 10;
localparam MAX_CYCLES = 10000;

// ── DUT signals ───────────────────────────────────────────────────────────────
// declare all ports of the DUT

// ── DUT instantiation ─────────────────────────────────────────────────────────
<ip_name>_top u_dut (
    // connect all ports
);

// ── Clock ─────────────────────────────────────────────────────────────────────
reg clk;
initial clk = 0;
always #(CLK_PERIOD/2) clk = ~clk;

// ── Error counter ─────────────────────────────────────────────────────────────
integer error_count;
initial error_count = 0;

// ── Watchdog ──────────────────────────────────────────────────────────────────
initial begin
    #(MAX_CYCLES * CLK_PERIOD);
    $error("tc_<name>: TIMEOUT — simulation exceeded %0d cycles", MAX_CYCLES);
    error_count = error_count + 1;
    $finish;
end

// ── Stimulus and checks ───────────────────────────────────────────────────────
initial begin
    // reset sequence
    // stimulus
    // wait for outputs
    // check results

    // Final result
    if (error_count == 0)
        $display("TESTCASE PASS");
    else
        $display("TESTCASE FAIL: %0d error(s)", error_count);
    $finish;
end

endmodule
```

#### Assertion convention

Every output check must use this pattern so vtestrun can locate failure context:

```verilog
if (actual !== expected) begin
    $display("  [CONTEXT] time=%0t <relevant signal dumps>", $time);
    $error("tc_<name>: <what failed> — expected %0h got %0h", expected, actual);
    error_count = error_count + 1;
end
```

The `[CONTEXT]` display line immediately before `$error` provides the signal state that `vtestrun` will include in the issue report. Include all signals that are meaningful for diagnosing the failure.

### 4B — Self-Check

Run from inside `tb/<ip_name>/`. All three steps must pass before moving to the next testcase.

**Step i — Compile**
```bash
xvlog -nolog -f files.f tc_<name>.v
```
Must exit with no errors.

**Step ii — Elaborate**
```bash
xelab -nolog --top tc_<name> --snapshot snap_<name>
```
Must exit with no errors.

**Step iii — Quick run**
```bash
xsim snap_<name> --runall --nolog
```
Must terminate (not hang). Output must contain `TESTCASE PASS` or `TESTCASE FAIL`.

**Step iv — Structure check**
Grep `tc_<name>.v` for `$error`. At least one must be present.

#### Self-check failure handling

| Failure | Action |
|---------|--------|
| xvlog error | Fix syntax/port mismatch and retry |
| xelab error | Fix missing module or signal type mismatch and retry |
| xsim hangs | Add or reduce watchdog timeout; re-run |
| No PASS/FAIL in output | Fix testcase result reporting block and retry |
| No `$error` in source | Add at least one assertion and retry |

Maximum 2 rewrite attempts per testcase. If still failing after 2 attempts, skip the testcase, record it as `SKIPPED` in the plan table, and continue to the next one. Do not block the full suite on a single testcase.

## Step 5 - Write Testcase List

After all testcases have been written and self-checked, write `./tb/<ip_name>/tc_list.md`.

```markdown
# Testcase List: <ip_name>

## RTL Filelist

All testcases use the shared filelist: `tb/<ip_name>/files.f`

## Testcases

| # | File | Module | Category | Description | Status |
|---|------|--------|----------|-------------|--------|
| 1 | tc_reset_basic.v | tc_reset_basic | Reset | DUT reaches idle after reset | OK |
| 2 | tc_basic_stream.v | tc_basic_stream | Basic | Single packet flows end-to-end | OK |
| 3 | tc_edge_empty.v | tc_edge_empty | Edge | Empty input condition | SKIPPED |
```

Status column values: `OK` for self-check passed, `SKIPPED` for testcases that failed self-check after 2 attempts.

## Known Issues and Fixes

This section records issues encountered when running this skill and how they were resolved.

<!-- Add entries below as issues are discovered. Format:
### [Tool] - <short issue title>
**Symptom:** what was observed
**Cause:** root cause
**Fix:** what was done to resolve it
-->
