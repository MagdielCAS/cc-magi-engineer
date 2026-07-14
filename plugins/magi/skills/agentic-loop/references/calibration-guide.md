# Calibration Guide — anchoring the Evaluator to the user's standard

Run this in Step 1, before the autonomous loop. The output is 2–3 graded
examples that you pass to **every** Evaluator call as a few-shot anchor. This is
the "Principal Domain Expert" step from Hamel's Critique Shadowing: a judge with
a few of the expert's own graded examples judges far more like the expert than
an instruction-only judge does.

## Why calibration matters (and why before, not after)

The Evaluator's job is to apply *the user's* standard, not a generic one. Two
engineers can read the same criterion ("functions have docstrings") and disagree
on whether a one-word docstring counts. Calibration resolves that disagreement
**up front** by showing the judge real graded examples. Doing it up front (not
after seeing outputs) is also what prevents criteria drift: the standard is fixed
by the user's examples before the loop produces anything to rationalize around.

## What to collect

Ask the user for **2–3 examples**. Each example has three parts:

1. **A candidate output** — a realistic example of the kind of thing the
   Implementer will produce for this goal (a function with a docstring, a diff, a
   generated file, a command's output).
2. **The verdict** — the user's binary call: PASS or FAIL. No scores.
3. **The critique** — *why*, in the user's own words, detailed enough that a new
   engineer could apply the same judgment. This is the most valuable part.

Aim for a mix: at least one PASS and at least one FAIL. The FAIL example is often
the most informative — it draws the line.

## How to ask

Keep it light and concrete. Something like:

> Before I start the loop, I want to judge work the way *you* would. Can you give
> me 2–3 quick examples? For each: a sample output, whether it PASSES or FAILS,
> and a sentence or two on why. One passing and one failing example is ideal —
> the failing one tells me where your line is.

If the user has examples in mind, capture them. If they're vague, offer to draft
candidate examples from the goal and let them correct the verdicts/critiques —
correcting is faster than authoring.

## If the user declines or has no examples

Two acceptable fallbacks — pick based on how much the user cares about judgment
precision:

- **Draft-and-confirm (preferred):** You write 2–3 plausible graded examples from
  the goal and criteria, present them, and ask the user to fix any verdict or
  critique they disagree with. Even a 30-second correction sharply improves
  alignment.
- **Proceed uncalibrated:** Skip calibration, but record `"calibrated": false`
  in `state.json` and tell the user the Evaluator is running on the stated
  criteria alone — judgments may be stricter or looser than they'd like, and they
  should spot-check early verdicts.

## Format to store (`calibration.md`)

```markdown
# Calibration examples (Evaluator few-shot anchor)

## Example 1 — PASS
**Candidate:**
def parse(text: str) -> list[Turn]:
    """Parse a raw transcript into ordered conversation turns."""
    ...
**Verdict:** PASS
**Critique:** One-line summary docstring that says what the function does and
returns. Imperative mood, fits on one line, PEP 257 compliant. This is the bar:
a docstring must describe behavior, not just restate the name.

## Example 2 — FAIL
**Candidate:**
def parse(text: str) -> list[Turn]:
    """parse"""
    ...
**Verdict:** FAIL
**Critique:** The docstring just echoes the function name. It tells a reader
nothing they couldn't get from the signature. A docstring that adds no
information fails — it must describe what the function does and what it returns.

## Example 3 — PASS
...
```

## How the examples flow into the loop

- Every **Evaluator** dispatch includes the full `calibration.md` as few-shot
  context (see `loop-protocol.md` Section 5). The examples are fixed for the
  whole run — they are the anchor.
- This is distinct from the **PASS critique bank**, which *grows* during the run
  as subtasks pass. Calibration = the user's standard, fixed up front. PASS bank
  = the run's own accumulating exemplars. The Evaluator receives both: the anchor
  and the run history.

## Verifying calibration worked

After the first 1–2 Evaluator verdicts, sanity-check them against the calibration
examples. If a verdict contradicts the user's grading style (e.g. it passes
something resembling a calibration FAIL), surface it: the criterion or the
calibration may need tightening before more steps burn budget. This quick check
early is far cheaper than discovering misalignment at the end of the run.
