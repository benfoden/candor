---
name: candor
description: "Fidelity audit of an output against its source, run by fresh-context subagents. Trigger on `/candor [output] [source]`, or when the user says 'check fidelity', 'audit this against the source', 'did I drop anything', 'verify this summary/extraction/translation/migration', or asks whether a generated artifact can be trusted before shipping. Also fires before handing off any transformed content (summary, extraction, table, port, refactor, transcript, spec, diff, translation) where the receiver will act on it without re-reading the source. Splits the six CANDOR failure modes across two independent subagents with a deliberate priority bias: one agent takes the 2 highest-base-rate modes, another takes the remaining 4."
argument-hint: "[output path/desc] [source path/desc] [--stability N] [--fix]"
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
  - Write
  - Edit
---

<objective>
A unit of content moved from a SOURCE into an OUTPUT. Find every place that move went wrong.

Producer cannot audit itself. Whoever wrote the output already believes it. So the audit runs in
**subagents with their own context**, given only the source and the output — never the reasoning,
plan, or justification that produced it. Fresh eyes, no priors to defend.
</objective>

## CANDOR — the six modes

Five content modes plus the confidence layer. Exhaustive at the content level.

| Mode | Plain name | What it is |
|------|-----------|------------|
| **C** — Confabulation | Fabrication | In the output, not in the source |
| **A** — Alteration | Alteration | In both, but changed: rounding, unit/scale drift, reformatting, silent "corrections" |
| **N** — Neglect | Silence | In the source, not in the output. Includes truncation |
| **D** — Discord | Contradiction | Output conflicts with itself, or with the source |
| **O** — Overconfidence | — | Certainty that does not track correctness |
| **R** — Reassignment | Transposition | Right content, wrong place: wrong field, wrong row, wrong entity |

Neglect = Silence, Reassignment = Transposition, Confabulation = Fabrication. Same things, letters
chosen to spell the word. Say the plain name to a human.

Overconfidence is not a content error. It decides whether the other five are actionable. A system
wrong 8% of the time that knows which 8% is usable. One wrong 3% of the time that cannot say where
is not.

## Priority bias — why 2 and 4

Base rates in transformation work are not flat. **Neglect and Alteration dominate** — omission and
truncation are the default failure of any lossy compression, and silent rounding / unit drift /
reformatting are the default failure of any re-serialization. They are also the quietest: both
produce output that reads as clean.

So split 2 / 4, not 3 / 3:

- **Agent DEEP** — 2 modes (Neglect, Alteration). Fewer modes, more focus per mode. This agent is
  expected to do exhaustive unit-by-unit enumeration, because that is what catches these.
- **Agent BROAD** — 4 modes (Confabulation, Discord, Reassignment, Overconfidence). Lower base
  rate, higher severity, and each is cheap to spot once you look for it. Scanning suffices.

Both run in parallel, in a single message.

## Procedure

### 1. Pin the pair

Name the SOURCE and the OUTPUT explicitly before spawning anything. If either is ambiguous, ask.

If the source is not available (no file, no upstream artifact, model-internal knowledge), say so
and stop — there is nothing to audit against. Report that instead of guessing.

### 2. Pick the unit

Say what a "unit of content" is for this pair: a claim, a row, a field, a number, a requirement, a
function, a citation, a bullet. Every finding must attach to one. Ban vague findings.

### 3. Spawn both agents in parallel

One message, two `Agent` calls. Give each: paths (or verbatim text) of source and output, the unit
definition, and its mode list. Give neither the rationale behind the output.

Both return the same shape:

```
MODE | unit id / locator | source says | output says | severity | confidence
```

Severity: **blocking** (a reader acting on this is wrong), **material** (changes meaning, not the
decision), **cosmetic**. Confidence: certain / likely / suspected.

**DEEP prompt skeleton:**

> Audit OUTPUT against SOURCE for exactly two failure modes. Ignore all others.
> **Neglect** — content in the source absent from the output, including truncation, dropped tail,
> dropped qualifiers/caveats/exceptions, dropped edge cases, dropped rows/fields.
> **Alteration** — content present in both but changed: rounding, precision loss, unit or scale
> drift (%, k/M, currency, timezone, base), reformatting that loses information, hedges hardened
> into assertions or assertions softened into hedges, silent correction of something you judged
> wrong in the source.
> Method: enumerate every unit in the SOURCE, then find its counterpart in the OUTPUT. Work the
> source side, not the output side — reading the output first makes omissions invisible. Report
> every unmatched or mismatched unit. Do not summarize; list.

**BROAD prompt skeleton:**

> Audit OUTPUT against SOURCE for four failure modes.
> **Confabulation** — any claim, number, name, citation, or detail in the output with no source
> support. Check every specific: numbers, proper nouns, dates, quotes, file paths, API names.
> **Discord** — output contradicting the source, or contradicting itself between two places.
> **Reassignment** — correct content attached to the wrong entity, row, field, column, date, or
> speaker. Check joins, tables, lists, attributions, and anything ordered.
> **Overconfidence** — statements whose stated certainty exceeds their support: hedged source
> rendered flat, inferred content presented as extracted, gaps papered over instead of marked
> unknown, uniform confident tone across items of unequal support.
> Method: work the OUTPUT side. For each unit, demand the source line that licenses it. No line,
> no license.

### 4. Merge

Dedupe overlaps (one defect can trip two modes — keep the most severe label, note the other).
Report as one table sorted by severity, mode second.

Then a one-line verdict:

- **Ship** — no blocking, no overconfidence findings.
- **Ship with the flags attached** — findings exist, and the output can be made to say where it is
  weak. This is the state Overconfidence buys you.
- **Do not ship** — blocking findings, or overconfidence findings that mean you cannot tell which
  parts are wrong. Full review required.

State counts per mode. A clean sweep on all six is a real result — say it plainly, do not invent
findings to look thorough.

### 5. `--fix` (optional)

Only with the flag. Apply blocking + material findings to the output, one edit per finding, cite
the source locator in each. Do not touch cosmetic. Re-run step 3 after, once.

## `--stability N` (optional)

Instability is not a mode you tag on an output — it is a multiplier you measure by running the
same input N times. It is the failure people find last and most expensively.

With `--stability N`: regenerate the output N times (default 3) from the same input, then diff the
runs. Any unit that varies across runs is unstable — report it separately from CANDOR findings,
because an unstable unit is untrustworthy even when today's run happens to be correct.

## Deliberately out of scope

Say this if the user asks why something is missing.

- **Duplication** — uncommon, and cheap to catch programmatically. No slot needed in a human-facing
  taxonomy.
- **Noncompliance** (schema/format violation) — largely engineered away by schema-constrained
  decoding. Let the validator catch it.
- **Nonresponse** — announces itself. Nobody misses it.
- **Instability** — real omission from the taxonomy, handled by `--stability` above, because it is
  a property of the system across runs, not of one output.

## Scale

Small pair (one page vs one page): both agents still run, they just finish fast. Don't skip the
split — the split is what buys independence.

Large pair (a repo port, a 200-row extraction, a long transcript): chunk the SOURCE into ranges and
give DEEP one agent per chunk, keeping BROAD whole-artifact so it can still see cross-chunk Discord
and Reassignment. Keep the 2/4 mode split inside each chunk.
