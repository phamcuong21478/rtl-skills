---
name: vtestrun
description: Run all IP-level testcases from tc_list.md, capture results, and write per-failure issue reports with no fix suggestions
---

# Verilog Testcase Runner

## Overview

This guide defines the workflow for running all IP-level testcases listed in `tc_list.md`, capturing simulation output, and writing structured issue reports for every failure.

The input is an IP name. The outputs are:

1. `./issue/<ip_name>/issue_NNN_<tcname>.md` — one file per failing testcase
2. `./issue/<ip_name>/run_summary.md` — overall pass/fail count and failure index

**This skill reports failure context only. It does not analyze root cause, propose fixes, or suggest code changes.**

## Prerequisites

| Tool  | Purpose                          | Check             |
|-------|----------------------------------|-------------------|
| xvlog | Verilog compile / lint           | `xvlog --version` |
| xelab | Elaborator                       | `xelab --version` |
| xsim  | Simulator                        | `xsim --version`  |

## Step 1 - Read Testcase List

Read `./tb/<ip_name>/tc_list.md`. Parse the testcase table to build a run list.

Include only rows with status `OK`. Skip rows with status `SKIPPED`.

For each entry, record:
- `file` — the `.v` filename
- `module` — the top-level module name inside that file
- `category` — test category
- `description` — one-line description

The shared filelist is `./tb/<ip_name>/files.f`. Verify this file exists before proceeding.

## Step 2 - Create Output Directory

Create `./issue/<ip_name>/` if it does not already exist.

If the directory already contains issue files from a previous run, do not delete them. New issue files from this run use the next available NNN index (find the highest existing index and continue from there). If no prior issues exist, start at `001`.

## Step 3 - Run Each Testcase

For each entry in the run list, run the three simulation steps from inside `./tb/<ip_name>/`.

```bash
xvlog -nolog -f files.f <file>
xelab -nolog --top <module> --snapshot snap_run
xsim snap_run --runall --nolog
```

Capture the combined stdout and stderr output of the `xsim` step. Record:
- Exit code of each step
- Full captured output of xsim

A testcase is a **FAIL** if any of the following is true:
- `xvlog` or `xelab` exits with a non-zero exit code
- xsim output contains `TESTCASE FAIL`
- xsim output contains any `Error:` line (from `$error` calls)
- xsim does not terminate within a reasonable time (treat as TIMEOUT FAIL)

A testcase is a **PASS** only if xsim exits normally and output contains `TESTCASE PASS`.

## Step 4 - Write Issue Reports

For each FAIL, assign the next sequential issue number (zero-padded to 3 digits) and write `./issue/<ip_name>/issue_NNN_<module>.md`.

Write `./issue/<ip_name>/issue_NNN_<module>.md` with the following sections. Fill every section from the captured simulation output. Do not add analysis, speculation, or fix suggestions.

**Section: Testcase**
A table with four rows: File (`tb/<ip_name>/<file>`), Module, Category, and Description.

**Section: Failure Summary**
Verbatim first `TESTCASE FAIL` line or first `Error:` line from the simulation output. One line only.

**Section: Simulation Log**
A fenced code block containing up to the first 10 `Error:` lines and their immediately preceding `[CONTEXT]` lines, in order of occurrence. If xvlog or xelab failed, include those error lines instead.

**Section: Reproduction**
A fenced bash block with the three commands needed to rerun the testcase from inside `tb/<ip_name>/`.

**Closing note**
End the file with a blockquote: `> This report documents observed failure context only. No fix is proposed.`

**Rules for the Simulation Log section:**
- Include up to the first 10 `Error:` lines produced by `$error` calls
- Include every `[CONTEXT]` line that immediately precedes an included `Error:` line
- Do not include unrelated `$display` lines or Vivado tool banners
- If the testcase timed out, include the TIMEOUT error line and the last 10 lines of output before termination
- If xvlog or xelab failed, include their full error output and leave the Simulation Log empty

## Step 5 - Write Run Summary

After all testcases have been run, write `./issue/<ip_name>/run_summary.md`.

```markdown
# Test Run Summary: <ip_name>

## Results

| Metric          | Count |
|-----------------|-------|
| Total run       | N     |
| Passed          | N     |
| Failed          | N     |
| Skipped (vtestgen) | N  |

## Failures

| Issue | Testcase | Category | Failure Summary |
|-------|----------|----------|-----------------|
| [001](issue_001_<module>.md) | tc_<name> | <category> | <first Error: line, truncated to 80 chars> |
| ... | | | |

## Skipped Testcases

| Testcase | Reason |
|----------|--------|
| tc_<name> | Failed self-check in vtestgen |

---
> Run completed: <date and time>
```

If all testcases passed, write the summary with zero failures and omit the Failures table.

## Known Issues and Fixes

This section records issues encountered when running this skill and how they were resolved.

<!-- Add entries below as issues are discovered. Format:
### [Tool] - <short issue title>
**Symptom:** what was observed
**Cause:** root cause
**Fix:** what was done to resolve it
-->
