# HDL Repair Skills

This repository contains experimental Codex Skills for HDL/RTL bug repair.

The skills are designed for evaluating whether structured debugging procedures can improve AI Agent performance on HDL code repair tasks.

## Skills

### 1. HDL Multi-Hypothesis Fault Localization

This skill guides the agent to identify possible root causes before editing code.

It asks the agent to:

- extract observed and expected behavior
- infer likely failure modes from visible evidence
- trace suspicious signals, modules, and files
- generate multiple root-cause hypotheses
- rank hypotheses by evidence, locality, and repair risk

Main output:

```text
Root-Cause Hypothesis Note
```

### 2. HDL Minimal Repair and Consistency Check

This skill guides the agent to produce a small, evidence-backed patch and check its side effects.

It asks the agent to:

- select a root-cause hypothesis
- make the smallest necessary patch
- check ready/valid, reset, flush, replay, and FSM side effects
- check related configuration or generated artifacts when needed
- record the patch decision and remaining risks

Main output:

```text
Patch Decision Note
```

## Design

The skills follow the LEGO-style skill abstraction:

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
├── README.md
├── hdl-fault-localization/
│   └── SKILL.md
└── hdl-minimal-repair/
    └── SKILL.md
```

## Status

Work in progress.
