---
name: vdebug
description: Trace failing signals through the RTL hierarchy, classify root cause, and write a debug report per issue — no fix is proposed
---

# Verilog Debugger

## Overview

This guide defines the workflow for diagnosing RTL failures captured in issue reports. Starting from the failure context, the skill traces signal paths statically through the module hierarchy, instruments intermediate nodes when needed, and classifies the root cause with file-level precision.

**This skill diagnoses root cause only. It does not propose or apply any code change.**

### Operating Modes

| Mode | Input | Use case |
|------|-------|----------|
| **Single** | IP name + issue number (e.g. `foo issue_003`) | Deep-dive on one specific issue |
| **Batch** | IP name only (e.g. `foo`) | Process all open issues; group by root cause |

In batch mode, process issues sequentially from lowest to highest number. Group issues that share the same root cause at the end.

### Outputs

- `./issue/<ip_name>/debug_NNN_<tcname>.md` — one per issue analyzed (NNN matches the issue number)
- `./issue/<ip_name>/debug_summary.md` — batch mode only; root cause grouping across all analyzed issues

## Prerequisites

| Tool  | Purpose              | Check             |
|-------|----------------------|-------------------|
| xvlog | Verilog compile      | `xvlog --version` |
| xelab | Elaborator           | `xelab --version` |
| xsim  | Simulator            | `xsim --version`  |

## Step 1 - Triage the Issue

Read `./issue/<ip_name>/issue_NNN_<tcname>.md`. Extract:

- The failing testcase file and module name
- The verbatim `Failure Summary` line
- All `[CONTEXT]` and `Error:` lines from the Simulation Log — these are the signal values at the moment of failure
- The reproduction commands (compile, elaborate, run)

Build a working set of facts: which output signal failed, what value it held, what value was expected, and at what simulation time.

## Step 2 - Understand the Test

Read `./tb/<ip_name>/tc_<name>.v`. Identify:

- The stimulus sequence applied to the DUT (what inputs, in what order, under what conditions)
- The reference model or expected-value computation inside the testbench
- The exact `$error` statement that fired — which output it checked and how the expected value was derived
- Whether the `[CONTEXT]` signals captured at failure time are consistent with the stimulus (i.e., whether the testcase itself could be the source of the error)

If the expected value in the testcase is clearly wrong (miscalculated, wrong port name, incorrect timing assumption), note it immediately. Classification in Step 5 will be `TC_BUG`.

## Step 3 - Trace the Signal Path (Static Analysis)

This is the primary debug method. Perform static RTL analysis before considering any simulation run.

Read `./doc/<ip_name>.md` and `./rtl/<ip_name>_top.v` to understand the IP structure.

Starting from the failing output port, trace backwards through the hierarchy using this process:

1. Identify which submodule at the current level drives the failing signal (look at wire assignments and port connections in the instantiation).
2. Read that submodule's documentation first. If documentation is missing, use `vexplain` to generate it before reading the RTL.
3. Read the submodule RTL. Find the `always` block or `assign` statement that computes the signal.
4. List every upstream signal that feeds into that block.
5. Cross-reference each upstream signal against the `[CONTEXT]` dump to determine which one carried a wrong value at failure time.
6. Descend into the submodule that produced the wrong upstream signal and repeat from step 1.

Continue until one of the following stopping conditions is met:

- A logic expression clearly computes the wrong result (wrong operator, wrong condition, missing case branch, off-by-one in counter)
- An FSM is in an unexpected state — check `localparam` state definitions and state transition conditions
- A pipeline stage register holds a wrong value traceable to a known incorrect computation upstream
- A primary input was driven with a value that the RTL was not designed to handle (testcase issue)

Use the project coding style conventions during tracing:
- Signals with suffix `_ff` are registers — their value at failure time was latched on the previous clock edge
- Signals with prefix `s0_`, `s1_`, etc. are pipeline stages — trace across stages using the clock offset between them
- Bus prefixes `s_`/`m_` identify slave/master interface direction
- Internal bus names `<master>2<slave>_<bus>` identify the exact module pair involved

Record the full chain as a table:

| Level | Signal | Module | File | Value at Failure |
|-------|--------|--------|------|-----------------|
| Top output | `m_data` | `foo_top` | `rtl/foo_top.v` | `0x00` (expected `0xFF`) |
| Stage 1 | `s1_result_ff` | `foo_proc` | `rtl/foo_proc.v` | `0x00` |
| Stage 0 | `s0_sum` | `foo_proc` | `rtl/foo_proc.v` | unknown |
| Root | `s0_op_sel` | `foo_ctrl` | `rtl/foo_ctrl.v` | `1'b0` (should be `1'b1`) |

If static analysis fully identifies the root cause, skip Step 4.

## Step 4 - Dynamic Probing (Only If Step 3 Is Inconclusive)

Use probing only when static tracing reaches a point where multiple upstream signals could be the source and none of their values appear in the `[CONTEXT]` dump.

### 4A — Add Probes

Add `$display` probes inside the suspected submodule only. Place them in the `always @(posedge clk)` block nearest to the suspected logic:

```verilog
// vdebug probe — temporary
always @(posedge clk)
    if (<enable_condition>)
        $display("[PROBE] t=%0t <sig_a>=%0h <sig_b>=%0h state=%0d",
                 $time, <sig_a>, <sig_b>, state);
```

Add a narrow enable condition (e.g., a cycle range or a specific input pattern) to limit output volume. Do not add probes across multiple submodules in the same run — instrument one level at a time.

### 4B — Re-run

Run from inside `./tb/<ip_name>/` using the reproduction commands from the issue report.

```bash
xvlog -nolog -f files.f tc_<name>.v
xelab -nolog --top tc_<name> --snapshot snap_dbg
xsim snap_dbg --runall --nolog
```

Capture the full output. Locate the `[PROBE]` lines nearest to the failure time and extract the signal values.

### 4C — Remove Probes

**Remove all probe `$display` statements and any temporary `always` blocks added for probing before proceeding to Step 5.** Verify the RTL is clean by re-reading the modified file.

Never leave debug artifacts in the RTL.

## Step 5 - Classify Root Cause

Assign exactly one classification:

| Class | Meaning |
|-------|---------|
| `RTL_BUG` | A logic block in a specific file computes the wrong value; behavior disagrees with the design documentation or proposal |
| `TC_BUG` | The testcase stimulus or assertion is incorrect; RTL behavior is consistent with its specification |
| `DOC_AMBIGUITY` | RTL behavior and testcase expectation both have a reasonable interpretation; the specification is unclear or contradictory |

For `RTL_BUG`: identify the file, module, and the specific `always` block or `assign` statement. State what it currently computes and what it should compute, citing the relevant section of the design proposal or module documentation as the specification.

For `TC_BUG`: identify the testcase file and the specific assertion or stimulus that is wrong. State why it is incorrect.

For `DOC_AMBIGUITY`: quote the conflicting specification sections and describe the ambiguity.

## Step 6 - Deduplication (Batch Mode Only)

After all issues have been analyzed, scan all debug reports written in this session. Group issues that point to the same root cause location (same file + same module + same block).

A single RTL bug can cause multiple test failures. Identifying shared root causes prevents redundant fix effort.

## Step 7 - Write Debug Report(s)

### Per-issue report: `./issue/<ip_name>/debug_NNN_<tcname>.md`

NNN must match the issue number from the corresponding `issue_NNN_<tcname>.md`.

Write the following sections:

**Section: Issue Reference**
A single link to the corresponding issue file: `issue_NNN_<tcname>.md`.

**Section: Classification**
One of `RTL_BUG`, `TC_BUG`, or `DOC_AMBIGUITY`.

**Section: Root Cause Location**
For `RTL_BUG`: file path, module name, and the specific block (always block description or assign expression). For `TC_BUG`: testcase file and the assertion or stimulus line. For `DOC_AMBIGUITY`: both the RTL file and the documentation section in conflict.

**Section: Root Cause Description**
Two to four sentences. For `RTL_BUG`: what the logic currently computes vs. what the specification says it should compute. For `TC_BUG`: why the testcase expectation is incorrect. For `DOC_AMBIGUITY`: what both interpretations are and where they conflict. Cite the relevant proposal or documentation section by name.

**Section: Signal Trace**
The full chain table from Step 3, filled with signal names, module names, file paths, and values at failure time. Include every level traversed, even if intermediate values were correct.

**Section: Evidence**
The `[CONTEXT]` and `Error:` lines from the original issue report, plus any `[PROBE]` lines collected in Step 4 if probing was performed.

**Closing note**
End the file with a blockquote: `> This report identifies root cause only. No code change is proposed.`

---

### Batch summary: `./issue/<ip_name>/debug_summary.md`

Write this file only in batch mode, after all per-issue reports are complete.

**Section: Results**
A table: total issues analyzed, breakdown by classification (`RTL_BUG` / `TC_BUG` / `DOC_AMBIGUITY`).

**Section: Root Cause Groups**
One sub-section per unique root cause location. Each sub-section lists:
- Root cause location (file, module, block)
- Classification
- All issue numbers that share this root cause, each linking to its `debug_NNN_<tcname>.md`

**Section: Unique Issues**
Issues that do not share a root cause with any other issue — listed individually with a one-line summary.

**Closing note**
End with a blockquote: `> This summary identifies root causes only. No code changes are proposed.`

## Known Issues and Fixes

This section records issues encountered when running this skill and how they were resolved.

<!-- Add entries below as issues are discovered. Format:
### [Tool] - <short issue title>
**Symptom:** what was observed
**Cause:** root cause
**Fix:** what was done to resolve it
-->
