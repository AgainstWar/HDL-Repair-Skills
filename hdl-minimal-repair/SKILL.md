---
name: hdl-minimal-repair
description: >
  Use this skill for generating minimal, safety-checked patches to fix HDL
  (hardware description language) design bugs. Trigger only when the problem
  has already been localized: the user provides a Root-Cause Hypothesis Note
  (from hdl-fault-localization or equivalent), an explicit bug location and
  fix direction, a clear repair intent, or an existing patch diff to review.
  The skill handles minimal patch planning, side-effect scanning (ready/valid,
  reset/flush/replay, FSM state, stale state, pipeline ordering), artifact
  link checking (configs, generated files, register descriptions, YAML/HJSON/
  JSON/Python generators), and test verification. If none of the entry signals
  are present — no hypothesis note, no bug location, no fix direction, no
  repair intent, no existing patch — stop and require localization first.
  Do NOT use this skill for initial bug diagnosis or hypothesis formation;
  that belongs to a separate root-cause analysis skill.
---

# HDL Minimal Repair and Consistency Check

A workflow for generating the smallest safe patch to fix an HDL design bug,
then verifying that the fix doesn't break hardware contracts or leave
cross-artifact inconsistencies.

## What This Skill Covers

This skill is the repair half of a two-skill pipeline (localization → repair).
It accepts two entry paths:

1. **With a Root-Cause Hypothesis Note** (from `hdl-fault-localization` or
   equivalent) — parse the note, select the best hypothesis, and proceed.
2. **With an existing repair intent** — when the user or a prior agent has
   already given a specific bug location, fix direction, or patch diff, promote
   that intent to a provisional hypothesis and proceed.

The skill does four things and nothing else:

1. **Minimal patch planning** — the smallest set of changes that address the
   selected hypothesis, with no unrelated modifications.
2. **Side-Effect Scanner** — checking hardware contract pairs (ready/valid,
   reset/flush/replay, FSM transitions, stale state, pipeline ordering).
3. **Artifact Link Checker** — ensuring configs, generated files, register
   descriptions, and generator scripts stay consistent.
4. **Test verification** — running the test suite and handling failures via
   fallback hypotheses.

This skill does NOT perform fault localization. If the bug has not been
localized, stop and require localization first.

## When to Use This Skill

This skill activates only when the problem has already been localized.
Any one of the following is sufficient to enter:

- A Root-Cause Hypothesis Note is provided (from `hdl-fault-localization` or
  equivalent).
- The user or a prior agent has given an explicit **bug location** (file,
  module, signal) and **fix direction** (what to change and why).
- The user has stated a clear **repair intent** (e.g., "widen this pointer,"
  "add a skid buffer," "fix the FSM transition").
- An existing **patch diff** needs review for side effects or consistency.

If none of these entry signals are present — the bug is not localized, no
hypothesis exists, and there is no specific repair intent to promote — **stop
and require localization first.** Do not attempt to diagnose the bug; that is
the responsibility of a separate fault-localization skill. Vague requests like
"fix this bug", "something is wrong", or "my testbench fails" without any
localization information are NOT entry signals — they lack the specificity
required for this skill's repair-only scope.

Typical triggering phrases: "apply the fix", "make the patch", "implement the
repair", "check this patch for side effects", "verify hardware contracts after
this change", "audit cross-artifact consistency".

## Input Requirements

At least one of the following must be present before starting. If none exist,
stop and require localization first.

1. **Root-Cause Hypothesis Note** — a file or inline text describing one or
   more hypotheses, each with evidence strength and rationale. This is the
   preferred input and is produced by `hdl-fault-localization`.
2. **Explicit bug location + fix direction** — the user or prior agent has
   identified a specific file, module, or signal and described what change
   to make. Promote this repair intent to a provisional hypothesis.
3. **Existing repair intent** — a stated plan (e.g., "widen the pointer from
   4 to 5 bits") that can be treated as a single-hypothesis note.
4. **Existing patch diff** — a proposed code change to review for side effects
   and artifact consistency.

Additionally, gather:
- **Relevant source files** — paths to the HDL modules, testbenches, and
  supporting code involved.
- **Test command** — the exact command to run the test suite.
- **Failure logs** — optional but recommended for context.

## Core Workflow

Follow these steps in order. Each step produces output that feeds into the next.

### Step 0: Entry Gate Check (BEFORE any code analysis)

Before reading any source files, logs, or waveforms, verify that at least one
entry signal is present in the user's message. Check in this order:

1. Is there a **Root-Cause Hypothesis Note** (a file path, inline YAML, or
   structured text with labeled hypotheses and evidence)?
2. Has the user specified a **concrete bug location and fix direction**
   (a specific file, module, or signal AND a description of what change to
   make — not just "fix it" or "something is wrong")?
3. Has the user stated a **specific repair intent** (e.g., "widen the pointer
   to 5 bits", "add a skid buffer", "change the FSM transition from A to B")?
4. Is there an **existing patch diff** to review?

If NONE of these are present — the user says only "fix this bug", "my
testbench fails", "something is wrong", or similar vague requests without
localization — **stop immediately.** Do not read the source code. Do not
attempt to diagnose the bug. Output:

```
This skill requires prior localization. Please run fault localization
first (hdl-fault-localization), or provide one of:
- A Root-Cause Hypothesis Note
- An explicit bug location (file/module/signal) and fix direction
- A specific repair intent (what to change and why)
- An existing patch diff to review
```

Only proceed to Step 1 after confirming an entry signal is present.

### Step 1: Establish the Repair Hypothesis

If a formal Root-Cause Hypothesis Note is available, parse it to extract:

- Each hypothesis, labeled and described.
- The evidence supporting each hypothesis (strong, moderate, weak).
- The suggested fix direction for each hypothesis.
- Any fallback ordering or dependency between hypotheses.

If the note contains multiple hypotheses, list them all. If it contains only one,
that simplifies selection — but still evaluate its evidence strength before
committing.

If no hypothesis note is provided but a repair intent exists (explicit bug
location + fix direction, a stated repair plan, or an existing patch diff),
promote that intent to a **provisional hypothesis**. Give it a label (e.g.,
"P1-sync-fifo-pointer-widen") and document the evidence that supports it from
the user's description. This provisional hypothesis is treated as the sole
hypothesis for Steps 2-7. Do not attempt to generate alternative hypotheses
or perform fault localization — this skill's scope ends at repair execution.

### Step 2: Select the Best Hypothesis to Patch First

Not all hypotheses are equal. Apply these selection criteria:

1. **Evidence strength** — prefer hypotheses with concrete evidence (waveform
   traces, log patterns, reproducible conditions) over speculative ones.
2. **Patch risk** — prefer hypotheses whose fix is localized (one or two
   modules, no structural changes) over those requiring broad rewrites.
3. **Reversibility** — prefer hypotheses whose patch is easy to revert if
   it doesn't work.
4. **Dependency order** — if Hypothesis B depends on Hypothesis A being
   correct, patch A first.

Document why you chose this hypothesis over the alternatives. If you can't
decide between two equally strong hypotheses, pick the one with lower patch
risk and note the other as the primary fallback.

### Step 3: Generate a Minimal Patch Plan

A "minimal patch" does not mean "change only one file." It means: **the smallest
set of changes that correctly addresses the selected hypothesis, with no
unrelated modifications.**

For each file you plan to modify, answer:

- **Why must this file change?** — trace the causal chain from hypothesis to
  this specific change.
- **What exactly changes?** — describe the change at the signal/logic level,
  not implementation details.
- **Why is this the minimal change?** — explain why a smaller change would
  be insufficient or a larger change unnecessary.

Reject any change that:
- Is cosmetic (formatting, renaming, comment updates unrelated to the bug).
- Is a refactoring ("while I'm here, let me clean this up").
- Hardcodes a test expectation to make a test pass.
- Modifies a test file without evidence that the test itself is wrong.

### Step 4: Execute the Side-Effect Scanner

Hardware designs have implicit contracts between signals. Changing one side
of a contract often breaks the other side. Before applying the patch, scan
for these contract violations:

#### ready/valid pairs

For every `ready`/`valid` handshake pair in the modified module and its
immediate neighbors:
- Does the patch change when `valid` is asserted or deasserted? If so, does
  the consumer's `ready` logic still work correctly?
- Does the patch change when `ready` is sampled or driven? If so, does the
  producer's `valid` logic still make sense?
- Is backpressure still handled correctly — can the pipeline stall without
  dropping data?

#### reset/flush/replay

For every reset, flush, or replay signal:
- Does the patch change what state is cleared or preserved on reset?
- Does a flush path still correctly drain or invalidate in-flight data?
- Does a replay path still restore state to the correct checkpoint?

#### FSM transition completeness

For every finite state machine touched by the patch:
- Are all states reachable from reset?
- Can the FSM deadlock (no valid exit from a state)?
- Can the FSM livelock (infinite loop between states without progress)?
- Does the patch add or remove any state transitions? If so, verify the
  new transition table is complete.

#### stale state

- Does the patch introduce any register or signal that can hold a stale
  value across transactions or clock cycles where it should be cleared?
- Does the patch rely on a signal being cleared by a mechanism that no
  longer fires correctly?

#### pipeline ordering

- Does the patch change the order of operations in a pipeline? If so, do
  downstream stages still receive data in the expected order?
- Does the patch change pipeline depth (stages added or removed)? If so,
  are all bypass and forwarding paths adjusted?

Document findings for each category, even if the result is "no issues found."

### Step 5: Execute the Artifact Link Checker

HDL projects often have configuration files, register maps, and generated
code that must stay synchronized with the HDL source. Check these links:

#### Configuration files

Scan for config files (YAML, HJSON, JSON, TOML, INI, or similar) that
reference the signals, parameters, or modules touched by the patch.
If the patch changes a parameter value, module name, or port list, update
the config to match.

#### Generated files

Check if any files in the project are auto-generated (by code generators,
scripts, or build steps). If the patch changes a source that feeds a
generator, either:
- Regenerate the output files, or
- Flag the inconsistency for manual regeneration.

#### Register descriptions

If the design includes a register map (CSV, XML, IP-XACT, SystemRDL, or
similar), check that register field positions, widths, and access types
haven't drifted from the HDL after the patch.

#### Generator scripts

Check Python, Perl, Tcl, or shell scripts that generate HDL or configuration
files. If the patch changes a pattern these scripts rely on (naming conventions,
signal widths, interface protocols), the scripts may produce broken output.

Document all artifacts checked and any inconsistencies found.

When the patch changes only internal logic (no public interfaces, parameters,
register fields, address maps, generated sources, or configuration-visible
behavior), the artifact link check collapses to "not applicable — internal-only
patch with no cross-artifact impact." Avoid scanning the full project tree for
artifacts that cannot be affected by the change. Run a full artifact search only
when the patch touches one of the surfaces listed above.

### Step 6: Apply the Patch and Run Tests

Apply the patch to the actual source files. Then run the test command:

1. Run the test suite once.
2. If all tests pass — document the result and proceed to Step 7.
3. If some tests fail — analyze:
   - **Are the failures pre-existing?** Check failure logs to see if these
     tests failed before the patch. Pre-existing failures are not the patch's
     fault — note them but don't block the fix.
   - **Are the failures caused by the patch?** If the patch introduced new
     failures, the fix is incomplete or incorrect. Go to the fallback path.
   - **Are the failures revealing a test bug?** Only conclude this if you
     have concrete evidence that the test expectation is wrong, not just
     because the patch changed behavior.

### Step 7: Handle Failure — Fallback to Next Hypothesis

If the patch fails tests in a way that is caused by the patch (not
pre-existing or test-bug):

1. **Do not** try to salvage the patch with ad-hoc tweaks. That turns a
   hypothesis-driven fix into guesswork.
2. If the execution environment supports clean rollback (e.g., `git checkout`,
   `git stash`, or an external harness managing patch state), **revert** the
   failed patch before trying the fallback. If rollback is not supported,
   explicitly inspect and undo the failed edits before continuing — do not
   layer a new patch on top of a broken one.
3. **Select the next hypothesis** from the hypothesis note (the one with the
   next-best combination of evidence and risk).
4. **Document** in the Patch Decision Note: which hypothesis failed, why
   (what test broke), and which hypothesis is now being tried.
5. **Return to Step 2** with the fallback hypothesis.

If all hypotheses are exhausted and no patch passes tests, report that the
hypothesis set is insufficient — the root cause may not be captured by any
of the hypothesized explanations.

## Output: Patch Decision Note

At the end of the workflow (whether the patch succeeded or failed), produce
a Patch Decision Note using this exact structure:

```
patch_decision_note:
  selected_hypothesis: <which hypothesis was patched>
  patch_scope: <one-sentence summary of what the patch changes>
  modified_files:
    - <file path>: <reason this file had to change>
    - <file path>: <reason this file had to change>
  side_effect_checks:
    ready_valid: <findings or "no issues found">
    reset_flush_replay: <findings or "no issues found">
    fsm_state: <findings or "no issues found">
    stale_state: <findings or "no issues found">
    pipeline_ordering: <findings or "no issues found">
  linked_artifacts_checked:
    - <artifact type>: <findings or "not applicable">
    - <artifact type>: <findings or "not applicable">
  why_minimal: <explanation of why this is the smallest sufficient change>
  test_result: <"all pass" | "pre-existing failures: X" | "new failures: Y">
  fallback_used: <"none" | "H2 after H1 failed: <reason>">
  remaining_risk: <any residual risk not addressed by this patch>
```

## Completion Criteria

The patch is complete only when ALL of these are true:

- The patch corresponds to a specific hypothesis — either from a formal
  hypothesis note or a provisional hypothesis promoted from repair intent.
- Every modified file has a documented reason for the change.
- All five side-effect categories (ready/valid, reset/flush/replay, FSM state,
  stale state, pipeline ordering) have been evaluated for relevance. Categories
  directly touched by the patch get detailed analysis; categories with no
  relevance to the patch receive a brief "not applicable — [reason]" note. Do
  not force a long-form analysis on categories the patch does not affect.
- If the patch touches configuration, registers, or generated files, the
  artifact link check has been completed and documented.
- Tests have been run and the result is documented.
- If tests failed due to the patch, the fallback path has been followed and
  documented.

## Constraints (Non-Negotiable)

These are built into the workflow above, but bear repeating because they are
the most common failure modes in HDL patching:

1. **Never hardcode a test expectation.** If a test expects value X and the
   fix changes behavior to Y (correctly), do not change the test to expect Y
   without evidence that the test itself was wrong.
2. **Never modify a test file without evidence.** "The test fails now" is not
   evidence that the test is wrong. Concrete analysis is required.
3. **Never refactor while fixing.** The patch must be minimal. Refactoring
   obscures the fix and makes regression analysis harder.
4. **Always check both sides of signal pairs.** If you change `valid`, check
   the matching `ready`. If you change `request`, check `response`. If you
   change `reset` behavior, check `flush` and `replay`.
5. **Minimal patch does not mean single file.** If the fix requires updating
   a module, its instantiation, and a config file, change all three. That is
   minimal for the fix — just don't also rename variables in the config file
   "while you're there."

## Example

**Scenario:** A FIFO overflow bug. The hypothesis note identifies Hypothesis
H1: the almost-full threshold is miscalibrated, causing `ready` to deassert
one cycle too late.

**Agent workflow:**
1. Reads hypothesis note — H1 has strong evidence (waveform shows `ready`
   dropping after `data_count` already exceeds `ALMOST_FULL - 2`).
2. Selects H1 — strongest evidence, lowest risk (one parameter change).
3. Patch plan: change `ALMOST_FULL_THRESHOLD` from `DEPTH - 2` to `DEPTH - 4`
   in `fifo_params.sv`. Also check `top.sv` where the threshold is passed as
   a parameter override.
4. Side-effect scan: checks that the new threshold doesn't cause `ready` to
   deassert so early that throughput drops below spec. Checks the consumer's
   `valid` sampling against the earlier `ready` deassertion. FSM states all
   remain reachable.
5. Artifact link check: finds `config/fifo.hjson` where `ALMOST_FULL_MARGIN`
   is set to 2 — updates to 4 to match. No generated files or register maps
   reference this parameter.
6. Applies patch, runs `make sim`. All tests pass.
7. Produces Patch Decision Note with `fallback_used: none`.

## Summary

This skill enforces discipline in HDL bug fixing: entry is gated on prior
localization (whether from a formal hypothesis note, an explicit repair intent,
or an existing patch), every patch has a documented reason, every side effect
is checked, every linked artifact is verified, and failures trigger a
structured fallback rather than ad-hoc tweaking. The goal is not speed —
it's correctness.
