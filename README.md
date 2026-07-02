# HDL Repair Skills

This repository contains experimental Codex Skills for HDL/RTL bug repair.

The skills are designed for evaluating whether structured debugging procedures can improve AI Agent performance on HDL code repair tasks.

## Skills

### 1. HDL Multi-Hypothesis Fault Localization

**Status**: Complete, tested.

A self-contained skill for systematically localizing bugs in HDL/RTL designs from visible evidence. It does not depend on any other skill and does not generate patches.

6-step procedure:

1. Extract observed vs. expected behavior
2. Route failure mode (timing / protocol / control / data-path)
3. Back-trace from symptom to source (signals, modules, files)
4. Generate 2-3 evidence-backed root-cause hypotheses
5. Rank by evidence strength, locality, and repair risk
6. Output a machine-parseable **Root-Cause Hypothesis Note**

The skill operates on problem statements, failing logs, and source files. Waveform data is optional. It deliberately does not:
- Use HWE-Bench offline bug type labels
- Read original issue/PR text
- Generate patches

See `hdl-fault-localization/SKILL.md` for the full procedure, constraints, output schema, and example.

Eval results (iteration 2): **18/18 (100%)** with-skill vs 10/18 (56%) without-skill across 3 HDL bug scenarios (FIFO pointer, AXI pipeline, FSM stuck).

### 2. HDL Minimal Repair and Consistency Check

**Status**: In progress.

This skill guides the agent to produce a small, evidence-backed patch and check its side effects.

It asks the agent to:

- select a root-cause hypothesis (from Skill 1's output)
- make the smallest necessary patch
- check ready/valid, reset, flush, replay, and FSM side effects
- check related configuration or generated artifacts when needed
- record the patch decision and remaining risks

Main output: **Patch Decision Note**.

## Design

Skills follow the LEGO-style skill abstraction:

S = (N, F, C, P, IO, Σ, G)

where each skill defines:

- N: name
- F: function
- C: constraints
- P: procedure
- IO: input and output
- Σ: output schema
- G: completion criteria

## Repository Layout

```
.
├── .gitignore
├── README.md
├── hdl-fault-localization/
│   ├── SKILL.md
│   └── evals/
│       └── evals.json
├── hdl-minimal-repair/
│   └── SKILL.md
└── test/
    └── fixtures/          # scratch files for manual testing
```

## Testing

Each skill has an `evals/` directory with test prompts and assertions. Eval runs use the skill-creator framework: subagents execute prompts with and without the skill, outputs are graded against assertions.

Workspace outputs (`*-workspace/`), simulation artifacts (`*.vcd`), and test fixture files are git-ignored.