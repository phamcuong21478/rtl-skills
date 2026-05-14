---
name: vflow
description: Scan project state to determine phase completion, write flow_status.md, and orchestrate the full RTL design flow from architecture to documentation
---

# RTL Flow Orchestrator

## Overview

This skill has two responsibilities:

1. **State scan** — inspect the project directories to determine what has been completed and what is pending for a given IP, then write `./flow_status.md`.
2. **Flow execution** — drive the complete RTL design flow by invoking the right skill at the right time, pausing at defined checkpoints, and looping through the debug-fix-retest cycle.

The input is an IP name. If not provided, infer it from the top-level requirement file in the project root (a `.md` file that is not `CLAUDE.md`).

## Step 1 - Scan Project State

Run the following checks to determine the completion status of each phase. All file checks are relative to the project root.

### Phase detection rules

| Phase | Complete when | Pending when |
|-------|--------------|--------------|
| 1 – Architecture | `ddoc/` contains at least one `_req.md` AND `rtl/<ip>_top.v` exists | Either is missing |
| 2 – Design | Every `ddoc/<module>_req.md` has a corresponding `rtl/<module>.v` | Any req file lacks an RTL file |
| 3 – Implementation | No `rtl/*.v` file (excluding `_top.v`) contains a `//!` marker | Any RTL file still has `//!` markers |
| 4 – Synthesis | `synth/<ip>/synth_report.md` exists | File is absent |
| 5 – IP Testing | `issue/<ip>/run_summary.md` exists | File is absent |
| 6 – Debug | `issue/<ip>/debug_summary.md` exists OR `run_summary.md` shows zero failures | Debug not yet run and failures exist |
| 7 – Documentation | `doc/<ip>.md` exists | File is absent |

To detect `//!` markers:
```bash
grep -rl '//!' rtl/
```

To check failure count in run_summary:
```bash
grep 'Failed' issue/<ip>/run_summary.md
```

### Submodule completion tracking

For Phase 2 and 3, track per-submodule status. A submodule list is derived from the files in `ddoc/` (one per `_req.md`). Cross-reference with `rtl/` to determine design and fill status individually.

## Step 2 - Write flow_status.md

Write `./flow_status.md` in the project root. Overwrite any existing file — this is a live status snapshot, not a history.

**Section: header**
IP name and scan date.

**Section: phase table**
One row per phase. Status symbols: `✓ DONE`, `⚠ IN PROGRESS (N/M)`, `✗ PENDING`, `✗ BLOCKED (<reason>)`. A phase is BLOCKED when a prerequisite phase is not yet DONE.

**Section: next action**
A single line stating the immediate next skill invocation and its target (e.g., `vfill → rtl/foo_ctrl.v`). If blocked, state what must complete first.

**Section: open issues** _(only if `issue/<ip>/run_summary.md` exists and shows failures)_
Number of open failures. If `debug_summary.md` exists, list each `RTL_BUG` root cause location (one line per unique location).

## Step 3 - Execute the Flow

After writing `flow_status.md`, determine the next incomplete phase and execute it. Follow the rules below for each phase.

A phase that is already DONE must be skipped entirely — do not re-run completed work.

---

### Phase 1 — Architecture `[CHECKPOINT]`

**Invoke when:** Phase 1 is PENDING.

Run `varch` on the top-level IP requirement file.

**Checkpoint:** Present the proposed module decomposition and interface definitions to the user. Wait for explicit approval before proceeding. If the user requests changes, revise and re-present. Do not advance to Phase 2 without approval.

---

### Phase 2 — Design `[CHECKPOINT per submodule]`

**Invoke when:** Phase 2 is PENDING or IN PROGRESS.

For each submodule in `ddoc/` that does not yet have a corresponding `rtl/<module>.v`:

1. Run `vdesign` on `ddoc/<module>_req.md`.
2. **Checkpoint:** Present the generated proposal for that submodule. Wait for explicit approval before running the next submodule. If the user requests changes, revise and re-present.

Process submodules one at a time. Do not batch-approve.

---

### Phase 3 — Implementation `[AUTO]`

**Invoke when:** Phase 3 is PENDING or IN PROGRESS.

For each `rtl/<module>.v` that still contains `//!` markers, run `vfill`.

Process all pending submodules sequentially without checkpoints. If `vfill` fails lint or simulation for a submodule, report the failure and pause — do not continue to other submodules while lint/sim errors are unresolved.

---

### Phase 4 — Synthesis `[AUTO]`

**Invoke when:** Phase 3 is DONE and Phase 4 is PENDING.

Run `vsynth` on the full IP.

No checkpoint. After `vsynth` completes, include the Fmax and utilization summary in the status output and continue automatically.

---

### Phase 5 — IP Testing `[AUTO]`

**Invoke when:** Phase 4 is DONE and Phase 5 is PENDING.

Run `vtestgen` to generate the IP-level test suite, then run `vtestrun` to execute all testcases.

No checkpoint between the two skills. After `vtestrun` completes:
- If `run_summary.md` shows zero failures → skip Phase 6, proceed to Phase 7.
- If failures exist → proceed to Phase 6.

---

### Phase 6 — Debug and Fix Loop `[AUTO + GATE]`

**Invoke when:** Phase 5 is DONE and `run_summary.md` shows failures.

#### 6A — Debug run
Run `vdebug` in batch mode on the IP.

#### 6B — Gate: await user RTL fix
After `vdebug` writes its reports, present the following to the user:

- Total open failures
- Every `RTL_BUG` entry from `debug_summary.md`: classification, file, module, block, and a one-line description
- Every `TC_BUG` entry: note that these require testcase corrections, not RTL fixes
- Every `DOC_AMBIGUITY` entry: note that these require clarification

State explicitly: **"RTL fixes are required before the flow can continue. Signal when fixes are applied."**

`TC_BUG` and `DOC_AMBIGUITY` issues do not block progression. If all remaining issues are `TC_BUG` or `DOC_AMBIGUITY` and no `RTL_BUG` remains, skip the gate and proceed to Phase 7.

#### 6C — Re-test after fix
When the user signals that RTL fixes have been applied:

1. Re-run `vflow` scan to confirm that RTL files were modified (check file timestamps or ask the user to confirm).
2. Run `vtestrun` again.
3. If new failures exist → run `vdebug` again (only on the new failures) and return to 6B.
4. If no failures remain → proceed to Phase 7.

Repeat the loop until either all failures are resolved or the user explicitly chooses to stop.

---

### Phase 7 — Documentation `[AUTO]`

**Invoke when:** All prior phases are DONE and no open `RTL_BUG` issues remain.

Run `vdoc` on the IP.

After `vdoc` completes, run a final `vflow` scan, write the updated `flow_status.md` showing all phases DONE, and present it to the user as the completion summary.

---

## Orchestration Rules

- **Never skip a CHECKPOINT.** Always wait for user confirmation at defined checkpoints before proceeding.
- **Never re-run a DONE phase** unless the user explicitly requests it.
- **Never proceed past a lint or simulation failure** in Phase 3.
- **Never leave probe `$display` statements** in RTL after a vdebug probing run (enforced by vdebug, but verify if in doubt).
- **Always update `flow_status.md`** after each phase completes.

## Known Issues and Fixes

This section records issues encountered when running this skill and how they were resolved.

<!-- Add entries below as issues are discovered. Format:
### <short issue title>
**Symptom:** what was observed
**Cause:** root cause
**Fix:** what was done to resolve it
-->
