---
name: hdl-fault-localization
description: >
  Use this skill to systematically localize bugs in HDL/RTL designs (Verilog,
  SystemVerilog, VHDL) by generating multiple evidence-backed root-cause
  hypotheses. Trigger whenever the user reports simulation failures, testbench
  mismatches, synthesis/simulation mismatches, unexpected signal values,
  protocol violations (AXI/AHB/APB), state machine misbehavior, formal
  verification failures, or any RTL bug symptoms — including "my testbench
  fails", "signal X is stuck", "assertion fires", "why is this module
  outputting wrong data", "design doesn't match spec", "timing violation",
  "ready never goes high", and "state machine gets stuck in state Y". Use also
  when the user asks to "root cause", "narrow down", or "identify where the
  bug is" in a digital design. This skill focuses on localization (finding
  where and why) rather than patching. It is fully self-contained: it requires
  no other skills to run. It can optionally be used alongside waveform-debugging
  tools for richer signal data, and its structured output can feed into a
  downstream repair workflow — but neither is a prerequisite.
---

# HDL Multi-Hypothesis Fault Localization

A structured reasoning workflow for identifying possible root causes of
HDL/RTL bugs before writing any code changes. The core principle: **generate
multiple competing hypotheses, then rank them by evidence strength** — never
commit to a single theory prematurely.

**This skill is self-contained.** It requires no other skills, no external
tools, and no pre-labeled bug databases to function. All it needs is the
user's problem statement, failing logs or test output, and relevant source
files. Waveform data and other skill integrations are optional enhancements,
not prerequisites. The skill produces a structured `Root-Cause Hypothesis
Note` that can feed into any repair workflow, automated or manual.

## What This Skill Covers

**Primary scope**: RTL simulation failures and testbench mismatches in
Verilog/SystemVerilog designs. This is where the skill is most effective —
the evidence (logs, waveforms, source code) is most accessible and the
debugging workflow is most structured.

**Secondary scope**: Synthesis/simulation mismatches, protocol violations,
and formal verification failures are also supported, but the hypotheses
will rely more heavily on code inspection and inference than on runtime
evidence, so confidence may be lower. Timing violations and clock-domain
crossing issues are covered only to the extent that simulation-level
signals reveal the symptom.

1. **Extract the failure profile** — separate observed from expected behavior.
2. **Route the failure mode** — classify symptoms into known HDL bug patterns
   (timing, protocol, FSM, reset, pipelining, width mismatch, etc.) without
   relying on offline taxonomies.
3. **Back-trace from symptom to source** — follow signals, control paths, and
   data paths upstream from the point of observable failure.
4. **Generate 2-3 root-cause hypotheses** — each backed by specific evidence
   from code, logs, or waveforms.
5. **Rank and deliver a Root-Cause Hypothesis Note** — structured output ready
   for any downstream repair tool or manual fix work.

## When to Use This Skill

Use this skill whenever you need to diagnose an HDL/RTL bug from visible
evidence — no other tools or skills are required. The skill works with:
- A digital design simulation or synthesis failure
- Log output or testbench results showing unexpected behavior
- Source files for the design under test (Verilog, SystemVerilog, VHDL)
- Optionally, a waveform summary or VCD/FST/FSDB file

This skill is especially valuable when:
- The failure is not obvious from a quick code scan
- Multiple modules interact and the fault could be in any of them
- The user wants a systematic, evidence-based diagnosis (not a guess)
- The output needs to feed into a structured repair workflow (this skill
  produces machine-parseable output that any downstream tool can consume)

### Relationship to Other Skills (all optional)

This skill is self-contained — it can run with only the user's problem
statement, logs, and source files. The following integrations are available
but are NOT prerequisites:

- **Waveform data**: If the user provides waveform summaries or VCD/FST/FSDB
  files, this skill uses the signal-level data to strengthen hypotheses. The
  WAVES Debug Skill (if available) can generate or query waveform files, but
  this skill accepts waveform observations in any form — a text summary from
  the user is sufficient.
- **Downstream repair**: The `Root-Cause Hypothesis Note` output is designed
  to be consumed by automated repair tools (such as the HDL Minimal Repair
  Skill). This skill deliberately stops at hypothesis generation; the repair
  step is separate and optional.

## Core Constraints

These constraints define what this skill does NOT do, and why:

| Constraint | Rationale |
|---|---|
| **Do not use HWE-Bench offline bug type labels** | The skill must work from visible evidence alone, matching real-world debugging where bug categories are unknown upfront. |
| **Do not read original issue/PR as runtime input** | The debugging environment provides logs, code, and waveforms — not the bug report narrative. |
| **Do not generate patches** | Localization and repair are separate concerns. A hypothesis is not a fix. |
| **Never treat the highest-ranked hypothesis as the only answer** | Real debugging requires fallback paths. The repair workflow needs alternatives if the first hypothesis is wrong. |
| **Every localization conclusion must cite visible evidence** | Evidence can be log lines, waveform values, code constructs, or simulation messages — never pure speculation. |

## Input Requirements

Before starting the procedure, gather these inputs from the user's context:

### Required
1. **Problem statement** — what the user observed and what they expected.
   Can be extracted from the conversation context or provided explicitly.
2. **Failing logs / test output** — simulation log excerpts, testbench
   failure messages, assertion fire messages, or compiler/synthesis warnings.
3. **Relevant source files** — the RTL files involved in the failure. At
   minimum, the module(s) that produce the incorrect output and their
   immediate ancestors in the hierarchy.

### Optional
4. **Waveform summary** — key signal values at failure time, transition
   timing, or a text description of waveform observations. If the user has
   VCD/FST/FSDB files, this skill accepts summary text describing the
   relevant signal values; precise waveform queries can be done separately
   using WAVES tools if available, but a plain-text summary from the user
   is sufficient.

### If Inputs Are Incomplete

If the user provides only a problem statement without logs or source files,
first attempt to locate the relevant files in the workspace using file search
and directory exploration. If critical inputs are genuinely missing, ask the
user for:
- The failing log/test output (copy-paste is fine)
- Which source file(s) to examine first

Do not fabricate evidence. If you cannot find supporting data for a
hypothesis, mark that hypothesis with lower confidence and note the evidence
gap explicitly.

## Procedure

Follow these six steps in order. Each step builds on the previous one.
Do not skip steps, and do not jump to conclusions before completing the
back-tracing in step 3.

---

### Step 1: Extract Observed vs. Expected Behavior

From the problem statement and logs, identify and clearly separate:

- **Observed behavior** — what actually happened. Quote specific log lines,
  signal values, or simulation messages. Be precise: "signal `data_out` is
  32'h0000_0001" not "output is wrong".
- **Expected behavior** — what should have happened according to the spec
  or the user's intent. If the expected behavior is only partially specified,
  note which parts are assumptions vs. confirmed.

Record these in the output schema as `observed_behavior` and
`expected_behavior`.

**Why this matters**: A clear gap statement prevents scope creep. Later
hypotheses must explain the gap between these two specific things — not
explain every possible thing that could go wrong.

---

### Step 2: Failure-Mode Routing (Not Bug Classification)

Instead of matching the failure to a pre-defined bug category, route it by
asking these diagnostic questions:

1. **Timing vs. logic**: Does the failure depend on simulation time or clock
   cycles? If yes, suspect clock domain crossing, reset sequencing, or pipeline
   depth errors.
2. **Protocol vs. computation**: Does the failure involve interface handshakes
   (valid/ready, request/grant)? If yes, suspect protocol FSM errors.
3. **Control vs. data path**: Is the wrong value computed, or is the right
   value computed at the wrong time / under wrong conditions? If control,
   suspect FSM or enable signals; if data, suspect arithmetic or mux errors.
4. **Systematic vs. intermittent**: Does the failure happen every time or
   sporadically? Intermittent failures suggest initialization, race conditions,
   or uninitialized state.
5. **Single-bit vs. multi-bit**: Is one bit wrong (wiring/slicing error) or
   many bits (width mismatch, endianness, sign extension)?

Record the primary failure mode and any secondary modes in the output. Use
concrete evidence from logs or code to justify each classification.

**Why this matters**: The routing questions narrow the search space. A
protocol failure leads to different suspect modules than a computation
failure, even if the symptom looks similar.

---

### Step 3: Back-Trace from Symptom Point

Starting from the signal or output that exhibits the failure:

1. **Identify the symptom location** — the exact signal, register, or output
   port where the wrong value appears. Note the **file** (required). Include
   the **line number** and **hierarchical path** when you can determine them
   from the source code, but do not fabricate these if the code structure
   makes them difficult to pinpoint — a module name and signal name are
   sufficient to proceed.
2. **Walk upstream** — for each driving signal or controlling condition:
   - Who assigns this signal? (trace the driver)
   - What conditions must hold for this assignment? (trace the enable/condition)
   - Where does the driving data originate? (trace the data path)
3. **Mark candidates** — as you walk upstream, flag every module boundary,
   state machine transition, counter, FIFO, or conditional that could inject
   the error. A candidate is any point where a bug would produce the observed
   symptom.
4. **Stop when** you reach either:
   - A primary input (testbench stimulus, top-level port)
   - A module you've already visited (avoid loops)
   - A point where the signal value is correct (the bug is downstream of here)

For each candidate, note the file, module/signal name, and why it's suspicious.
These candidates become the basis for hypotheses in step 4.

**Why this matters**: Back-tracing ensures every hypothesis has a causal chain
from source to symptom. Without this step, hypotheses are speculation without
a mechanism.

---

### Step 4: Generate 2-3 Root-Cause Hypotheses

For each promising candidate from step 3, formulate a complete hypothesis.
Each hypothesis is a **falsifiable claim** about what specific bug caused the
observed failure.

A hypothesis must include:

- **`id`**: A short label (e.g., "H1-fsm-stall", "H2-width-mismatch")
- **`suspect_file`**: The file most likely containing the bug
- **`suspect_module_or_signal`**: The module instance or signal name
- **`upstream_source`**: The driving signal/module/input that feeds the
  suspect location (from step 3 back-trace)
- **`evidence`**: Specific observations that support this hypothesis. Must
  reference log lines, code snippets, or waveform observations. Use this
  format: `"[source]: [observation]"` — e.g., `"log line 42: data_out=0x01
  when addr=0x00, but addr selects index 0 not 1"`
- **`patch_direction`**: A one-sentence description of what kind of fix
  would address this hypothesis (NOT the actual patch). Use directional
  language: "change the FSM transition condition from X to Y", "widen the
  signal from N bits to M bits", "add a reset for register Z"
- **`risk`**: `low`, `medium`, or `high` — estimated risk of the fix
  causing side effects, based on the signal's fanout, module complexity,
  and spec criticality
- **`confidence`**: `low`, `medium`, or `high` — how strongly the evidence
  supports this hypothesis vs. alternatives

**Coverage requirements**:
- At least one hypothesis must explain the **symptom location** (where the
  failure is observed)
- At least one hypothesis must point to a **possible source location** (a
  different module or earlier pipeline stage than the symptom location)

**Quality over quantity**: 2 well-evidenced hypotheses are better than 3
weak ones. If you genuinely cannot find evidence for a third hypothesis,
deliver 2. The repair workflow needs at least one fallback option — so
default to 2+ hypotheses; see Edge Cases for the evidence-gap exception.

---

### Step 5: Rank Hypotheses

Sort hypotheses by this priority:

1. **Evidence strength** (most important) — direct log/waveform confirmation
   outranks inference from code structure, which outranks "could be" speculation
2. **Locality** — bugs in the same module as the symptom are easier to verify
   than bugs three modules upstream
3. **Repair risk** — lower-risk fixes are preferred as first attempts

The top-ranked hypothesis becomes `selected_hypothesis`. The remaining
hypotheses become `fallback_hypotheses` in rank order.

---

### Step 6: Produce the Root-Cause Hypothesis Note

Assemble all findings into the structured output below. The output must be
valid YAML (or equivalent structured text) so that it can be parsed by
automated tools or read directly by a human following up with a fix.

## Output Schema

The output of this skill is the **Root-Cause Hypothesis Note**. Use this
exact structure:

```yaml
root_cause_hypothesis_note:
  observed_behavior: >
    [Precise description of what was observed. Quote specific log lines,
    signal values, and timestamps.]

  expected_behavior: >
    [What should have happened. Note any assumptions vs. confirmed spec
    requirements.]

  primary_failure_mode: >
    [The dominant failure category from step 2 routing. E.g.,
    "protocol FSM error — valid deasserted during expected transfer",
    "data path width mismatch — upper 16 bits zero on 32-bit bus",
    "reset sequencing — counter starts before reset deassertion".]

  secondary_failure_modes:
    - [Additional contributing failure categories, if any]

  hypotheses:
    - id: "H1-[short-descriptor]"
      suspect_file: "path/to/file.sv"
      suspect_module_or_signal: "module_name.signal_name"
      upstream_source: "driving_module.driving_signal"
      evidence:
        - "log line N: [specific observation]"
        - "code at file:line: [specific code pattern]"
        - "waveform at T=X: [signal value observation]"
      patch_direction: >
        [One sentence describing the category of fix — not the actual code.]
      risk: "low|medium|high"
      confidence: "low|medium|high"

    - id: "H2-[short-descriptor]"
      suspect_file: "path/to/file.sv"
      suspect_module_or_signal: "module_name.signal_name"
      upstream_source: "driving_module.driving_signal"
      evidence:
        - "log line N: [specific observation]"
      patch_direction: >
        [One sentence describing the category of fix.]
      risk: "low|medium|high"
      confidence: "low|medium|high"

    # Add H3 if evidence supports it; otherwise deliver 2 hypotheses

  selected_hypothesis: "H1-[short-descriptor]"
  fallback_hypotheses:
    - "H2-[short-descriptor]"
    # - "H3-[short-descriptor]"  # if present
```

### Schema Field Reference

| Field | Type | Required | Description |
|---|---|---|---|
| `observed_behavior` | string | Yes | What was observed. Quote specific evidence. |
| `expected_behavior` | string | Yes | What should have happened. |
| `primary_failure_mode` | string | Yes | Dominant failure category from step 2 routing. |
| `secondary_failure_modes` | list | No | Additional contributing failure categories. |
| `hypotheses` | list | Yes | 2-3 hypothesis objects. |
| `hypotheses[].id` | string | Yes | Unique hypothesis identifier. |
| `hypotheses[].suspect_file` | string | Yes | File most likely containing the bug. |
| `hypotheses[].suspect_module_or_signal` | string | Yes | Module instance or signal name. |
| `hypotheses[].upstream_source` | string | Yes | Driving signal/module from back-trace. |
| `hypotheses[].evidence` | list | Yes | At least 1 evidence item per hypothesis. |
| `hypotheses[].patch_direction` | string | Yes | Category of fix (not the patch itself). |
| `hypotheses[].risk` | enum | Yes | `low`, `medium`, or `high`. |
| `hypotheses[].confidence` | enum | Yes | `low`, `medium`, or `high`. |
| `selected_hypothesis` | string | Yes | ID of the top-ranked hypothesis. |
| `fallback_hypotheses` | list | Yes | IDs of remaining hypotheses in rank order. |

## Completion Criteria

The task is complete when ALL of the following are true:

1. **At least 2 hypotheses** are generated. This is the default requirement.
   If the available evidence is genuinely insufficient for a second hypothesis,
   delivering 1 is permitted but requires: (a) an explicit `evidence_gap` note
   in the output explaining why a second hypothesis could not be formed, and
   (b) the single hypothesis must be marked `confidence: low`. Fabricating a
   weak second hypothesis is worse than honestly delivering 1 with a gap note.
2. **Every hypothesis has at least 1 evidence item** that cites a specific
   observation (log line, code construct, waveform value).
3. **At least one symptom location** is identified (the signal/output where
   the failure appears).
4. **At least one possible source location** is identified (a module or
   signal upstream from the symptom that could be the root cause).
5. **The output is structured** using the `root_cause_hypothesis_note` schema
   and is machine-parseable (valid YAML or equivalent).
6. **No patch code** is included in the output. Patch direction is a one-line
   description of the fix category, not code.

## Edge Cases

### Single-Hypothesis Scenarios

If evidence is genuinely insufficient to form even 2 hypotheses, do not
fabricate a second one. Instead:
- Deliver the single hypothesis with `confidence: low`
- Add an `evidence_gap` field to the output explaining why the evidence
  limits hypothesis formation
- Note what additional information would raise confidence (e.g., "waveform
  of signal X at time T would confirm or refute this hypothesis")

This is the exception, not the norm — most failures have enough visible
evidence for at least 2 plausible hypotheses. When in doubt, generate a
lower-confidence second hypothesis rather than defaulting to 1.

### Insufficient Evidence (General)

If the available logs and code do not provide enough information to generate
even 2 hypotheses, do not fabricate them. Instead:
- Explain the evidence gap in `observed_behavior`
- Generate what you can with explicit confidence=`low`
- Note what additional information would raise confidence (e.g., "waveform of
  signal X at time T would confirm or refute this hypothesis")

### Multi-Module Failures

If the failure involves a transaction passing through multiple modules (e.g.,
AXI read data path: master → interconnect → slave → memory), generate
hypotheses at different points in the chain. This ensures there are
options to check before rewriting the wrong module.

### Waveform Data Available

If the user provides waveform data (text summary or VCD/FST/FSDB files),
prioritize waveform evidence over log evidence — waveform values at the exact
failure time are the strongest possible evidence. The WAVES Debug Skill (if
installed) can help generate or query waveform files, but is not required:
a text description of observed signal values at key times is sufficient for
this skill to work. If waveform data could discriminate between two
hypotheses but is unavailable, note this as an evidence gap.

## Example

### Input

**Problem statement**: "My AXI4-Stream FIFO testbench fails. The `m_axis_tdata`
output is always zero, even though `s_axis_tdata` has valid data coming in."

**Log excerpt**:
```
[    0] TB: reset deasserted
[   10] TB: sending data 0xDEAD_BEEF on s_axis_tdata
[   10] TB: s_axis_tvalid=1, s_axis_tready=1
[   15] TB: sending data 0xCAFE_BABE on s_axis_tdata
[   15] TB: s_axis_tvalid=1, s_axis_tready=1
[   20] TB: checking m_axis_tdata
[   20] ERROR: m_axis_tdata=0x0000_0000, expected 0xDEAD_BEEF
```

**Source file**: `axis_fifo.sv` (simplified excerpt):
```systemverilog
module axis_fifo #(parameter WIDTH=32, DEPTH=8) (
  input clk, rst_n,
  input [WIDTH-1:0] s_axis_tdata,
  input s_axis_tvalid, output s_axis_tready,
  output [WIDTH-1:0] m_axis_tdata,
  output m_axis_tvalid, input m_axis_tready
);
  reg [WIDTH-1:0] mem [0:DEPTH-1];
  reg [2:0] wr_ptr, rd_ptr;  // NOTE: 2:0 = 8 entries, but DEPTH=8 means addr=0..7
  reg [3:0] count;            // NOTE: 4 bits = 0..15 range
  // ...
  assign m_axis_tdata = mem[rd_ptr];
endmodule
```

### Output (Root-Cause Hypothesis Note)

```yaml
root_cause_hypothesis_note:
  observed_behavior: >
    m_axis_tdata reads as 0x0000_0000 at time 20 in simulation, even though
    two data words (0xDEAD_BEEF and 0xCAFE_BABE) were successfully written
    with s_axis_tvalid=1 and s_axis_tready=1 at times 10 and 15 respectively.
    The wr_ptr and rd_ptr are 3 bits wide, limiting address range to 0..7,
    which is sufficient for DEPTH=8.

  expected_behavior: >
    m_axis_tdata should output 0xDEAD_BEEF (the first written word) after
    the read side consumes data. After reset, with valid writes followed by
    reads, the FIFO should produce written data in order.

  primary_failure_mode: >
    Read-before-valid — the read port reads `mem[rd_ptr]` but the entry at
    that index was never written by the write side. The write path populates
    `mem[wr_ptr]` correctly, but the read pointer has advanced past the
    write pointer (or started ahead of it), so the read side is sampling
    an unwritten slot. In Verilog simulation, reading an unwritten `reg`
    array element produces X (unknown); some simulators or display formats
    may render X as 0, but the root cause is the read/write pointer
    ordering, not memory initialization.

  secondary_failure_modes:
    - Possible read pointer misalignment — if rd_ptr increments before data
      is actually available at the read port, it could read an empty slot.

  hypotheses:
    - id: "H1-rd-ptr-ahead-of-wr-ptr"
      suspect_file: "axis_fifo.sv"
      suspect_module_or_signal: "axis_fifo.rd_ptr / axis_fifo.wr_ptr ordering"
      upstream_source: "axis_fifo.read FSM advances rd_ptr without checking that the entry at mem[rd_ptr] has been written"
      evidence:
        - "code at axis_fifo.sv: assign m_axis_tdata = mem[rd_ptr] — read is
          combinational; if rd_ptr has advanced past wr_ptr, it reads an entry
          that the write side has not yet populated, producing X (or 0 in some
          display formats)"
        - "log at time 20: m_axis_tdata=0x0000_0000 — reading from an unwritten
          slot; two writes at times 10 and 15 should have populated mem[0] and
          mem[1], so rd_ptr may already be at index 2+ or was never at 0 after
          reset"
        - "code at axis_fifo.sv: no valid-bit per entry — the read side has no
          mechanism to distinguish 'entry was written' from 'entry is stale/
          unwritten' other than comparing rd_ptr to wr_ptr"
      patch_direction: >
        Either: (a) add a per-entry valid bit that the write side sets and the
        read side checks before asserting m_axis_tvalid, or (b) gate reads so
        that rd_ptr cannot advance past wr_ptr (i.e., only read when count > 0
        using a properly synchronized count).
      risk: "low"
      confidence: "high"

    - id: "H2-rd-ptr-advance-before-write"
      suspect_file: "axis_fifo.sv"
      suspect_module_or_signal: "axis_fifo.rd_ptr"
      upstream_source: "axis_fifo.read FSM increments rd_ptr"
      evidence:
        - "code at axis_fifo.sv: rd_ptr is 3 bits (2:0), which is correct
          for DEPTH=8 — ruling out pointer width overflow"
        - "log at time 20: m_axis_tdata=0x0000_0000 — if rd_ptr advanced
          past wr_ptr before data settled, it reads an empty (uninitialized)
          entry"
        - "no log evidence of read-side activity: unclear if m_axis_tready
          was asserted and when rd_ptr incremented"
      patch_direction: >
        Check the read FSM timing — ensure rd_ptr only increments after
        the current read data is consumed. Add a condition that rd_ptr
        must be less than wr_ptr (accounting for wraparound) before
        asserting m_axis_tvalid or advancing the read.
      risk: "medium"
      confidence: "medium"

  selected_hypothesis: "H1-rd-ptr-ahead-of-wr-ptr"
  fallback_hypotheses:
    - "H2-rd-ptr-advance-before-write"
```

Note how H1 identifies the symptom location (`m_axis_tdata` outputting X/0
because `mem[rd_ptr]` reads an unwritten slot) and points upstream to the
rd_ptr/wr_ptr ordering in the read FSM. H2 identifies a possible source
location (`rd_ptr` advancement timing) that is upstream from the symptom.
Both cite specific evidence. The primary failure mode correctly avoids
claiming that uninitialized Verilog regs default to 0 — the root cause is
the read-side sampling an entry the write side hasn't populated, not a
missing reset on the memory array.

## Failure Mode Reference

When diagnosing, use these common HDL failure patterns as starting points
for your routing questions (Step 2). These are not exhaustive — use them to
prompt investigation, not to classify:

| Failure Pattern | Key Symptoms | Where to Look |
|---|---|---|
| Uninitialized state | Output is X, Z, or default value | `reg` without reset, `logic` without initial, memory arrays |
| Width mismatch | Upper bits zero, wrong values after arithmetic | Assignment LHS width vs. RHS width, concatenation ordering, parameter vs. hardcoded widths |
| Reset sequencing | First few cycles produce wrong output | Reset signal timing, reset domain crossing, async vs. sync reset |
| Protocol handshake | Valid/ready deadlock, data dropped | FSM transition conditions, missing `ready` check before `valid` assert |
| FSM stuck / dead state | Module stops responding | Unreachable states, missing default case, one-hot encoding errors |
| Pipeline depth / bubble | Data arrives too early or late | Shift register depth, counter-based delays, pipeline stage count |
| Clock domain crossing | Metastability, intermittent corruption | Missing synchronizers, insufficient CDC constraints, gray code errors |
| Endianness / byte ordering | Bytes reversed, word swapped | Little-endian vs. big-endian in multi-byte fields |
| Sign extension | Negative values misinterpreted | `$signed()` missing, unsigned by default, comparison with negative literals |
| Off-by-one | Counter wraps early/late, FIFO full/empty at wrong count | Counter size, comparison operator (`<` vs `<=`), flag assertion timing |
| Priority / arbitration | Wrong request granted, starvation | Priority encoder ordering, round-robin state corruption |
| Generate / parameter | Different parameter values produce different failures | Compile-time condition (`ifdef`, `generate if`), parameter override |

Use this table as a diagnostic lens — not as a labeling system. The routing
in Step 2 determines which patterns are relevant; then back-tracing in
Step 3 locates the specific code that could be causing the matched pattern.
