---
name: hdl-minimal-repair
description: >
  Review an HDL repair idea or candidate patch for concrete inconsistencies
  in the changed logic and directly related behavior. Use while developing
  or reviewing a bug fix, with deeper guidance when a specific contradiction
  emerges.
metadata:
  version: "2.0.0-draft.1"
---

# HDL Repair Review

Support the agent's ongoing repair with focused review. Adapt to the current
repair state rather than restarting a fixed workflow.

## Start from the Current Repair State

Work with the current repair idea or candidate patch. If neither exists,
continue ordinary diagnosis and develop a repair idea. A formal hypothesis
note or a separate localization skill is not a prerequisite.

## Review the Change and Its Direct Effects

Check whether the proposed change addresses the reported problem and is
consistent with directly related behavior. Use the task requirements,
relevant source, and available execution evidence to assess the change.

Keep the review focused on the proposed changes and their direct dependencies.
Deepen the analysis or expand the review when a concrete contradiction emerges,
such as a conflict with a task requirement, an inconsistent dependent use, or
an observed failure that contradicts the proposed behavior. Missing test
results alone are not such a contradiction.

## Respond to Concrete Contradictions

Prefer a local correction when the repair direction still explains the bug.
Revisit the root-cause hypothesis when evidence contradicts it, and form a new
hypothesis from that evidence. Alternatives need not come from a prewritten
list. Checking a related location does not by itself require changing it.

## Use Agent Judgment

Choose the implementation scope, environment preparation, validation approach,
response to test failures, and completion point according to the task and
available environment. These review principles do not impose a fixed sequence
or an additional completion checklist.

## Summarize the Repair

Briefly explain what changed, why it addresses the problem, and what validation
actually ran and showed. If validation was not run, say so. Include additional
review details when a concrete issue was found; a category-by-category report
is unnecessary.
