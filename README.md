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

### 2. HDL Repair Review — V2 Draft

**Status**: Draft (`2.0.0-draft.1`); not yet experimentally evaluated.

This skill supports the agent's current repair idea or candidate patch with
focused review of the changed logic and directly related behavior. A concrete
contradiction triggers deeper analysis. It favors local correction when the
repair direction remains valid, and allows a new hypothesis when evidence
contradicts the current one.

The agent decides implementation scope, environment preparation, validation,
response to test failures, and when to finish. The skill uses general review
principles rather than a fixed HDL checklist or a mandatory localization step.

Main output: a brief explanation of the changes, their rationale, and actual
validation results.

The existing `hdl-minimal-repair` identifier is retained. See
[`hdl-minimal-repair/SKILL.md`](hdl-minimal-repair/SKILL.md) for the draft.
Here, V2 denotes the skill revision, independently of experimental model-cohort
labels.

## Design

The original skills follow the LEGO-style skill abstraction:

S = (N, F, C, P, IO, Σ, G)

where each skill defines:

- N: name
- F: function
- C: constraints
- P: procedure
- IO: input and output
- Σ: output schema
- G: completion criteria

The Repair V2 draft instead uses flexible review principles and a brief summary,
without a fixed procedure, output schema, or completion checklist.

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

The localization skill has an `evals/` directory with test prompts and assertions.
Its eval runs use the skill-creator framework: subagents execute prompts with and
without the skill, and outputs are graded against assertions. The Repair V2 draft
has not yet undergone behavioral or benchmark evaluation.

Workspace outputs (`*-workspace/`), simulation artifacts (`*.vcd`), and test fixture files are git-ignored.

## License

MIT License. See [LICENSE](LICENSE.md) for details.