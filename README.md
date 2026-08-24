# candor

Fidelity auditing for Claude Code. Does the thing you shipped still agree with the thing it came from?

## What it does

A unit of content moved from a SOURCE into an OUTPUT. `candor` finds every place that move went wrong.

The producer cannot audit itself, so the audit runs in subagents with their own context, given only
the source and the output, never the reasoning that produced it.

### The six modes

| Mode | Plain name | What it is |
|------|-----------|------------|
| **C** — Confabulation | Fabrication | In the output, not in the source |
| **A** — Alteration | Alteration | In both, but changed: rounding, unit/scale drift, reformatting, silent corrections |
| **N** — Neglect | Silence | In the source, not in the output. Includes truncation |
| **D** — Discord | Contradiction | Output conflicts with itself, or with the source |
| **O** — Overconfidence | — | Certainty that does not track correctness |
| **R** — Reassignment | Transposition | Right content, wrong place: wrong field, wrong row, wrong entity |

CAND_R is FACTS (Fabrication, Alteration, Contradiction, Transposition, Silence) plus the confidence
layer. The five content modes are exhaustive at the content level, minus duplication.

### The 2/4 split

Base rates are not flat. Neglect and Alteration dominate transformation work and both produce output
that reads clean, so they get a dedicated agent doing exhaustive unit-by-unit enumeration. The other
four are rarer, higher-severity, and cheap to spot once you look — one agent scanning covers them.

- **DEEP** — Neglect, Alteration. Works the source side.
- **BROAD** — Confabulation, Discord, Reassignment, Overconfidence. Works the output side.

Both run in parallel.

### Usage

```
/candor <output> <source> [--stability N] [--fix]
```

- `--fix` applies blocking and material findings, then re-audits once.
- `--stability N` regenerates N times and diffs the runs. Instability is a multiplier you measure
  across runs, not a label you tag on one output.

### Out of scope, on purpose

Duplication (cheap to catch programmatically), Noncompliance (schema-constrained decoding handles
it), Nonresponse (announces itself). Instability is handled by `--stability`.

## Install

```
claude plugin marketplace add benfoden/candor && claude plugin install candor@candor
```

Loads at next session start.
