---
name: agentic-loop
description: >-
   Run a bounded, verifiable agentic loop that plans, implements, and evaluates
   work until it passes or the step budget is exhausted. Use when the user asks
   for an "agentic loop", "autonomous loop", "self-correcting loop",
   "plan/implement/evaluate loop", "run until it passes", "keep iterating until
   X", "loop until all tests pass", or "loop until lint is clean", or when they
   want LLM-as-a-judge style PASS/FAIL gating with detailed critiques and clear
   stopping conditions for a concrete goal.
---

# Agentic Loop Orchestrator

You are about to act as the **orchestrator** of an agentic loop. You receive a
high-level goal, break it into verifiable subtasks, and drive each one to a
verified PASS by dispatching three specialized subagents in a tight
Reason -> Act -> Observe -> Repeat cycle — without asking the user to approve
every step. You stop when the goal is met, the step budget is exhausted, or a
destructive action needs human sign-off.

This is a **meta-skill**: you are the only stateful actor. The Planner,
Implementer, and Evaluator are **stateless leaf subagents** — they know nothing
except what you put in their prompt. You hold the state, assemble each prompt,
read each result, and decide the next move. Subagents do not spawn other
subagents; every dispatch comes from you.

## Why this loop is shaped the way it is

A plain LLM call is request -> response: one shot, no feedback. An agentic loop
adds the missing primitive — **the feedback loop** — so the agent acts, observes
*real* results (a test run, a file diff, a judge's critique), and corrects. The
design choices below come from hard-won practice; each exists to defeat a
specific failure mode. Read them as reasoning for why the loop works; the
operational requirements live in the Execution protocol and Guardrails recap
and must be followed.

1. **The Evaluator returns binary PASS/FAIL plus a detailed critique — never a
   number.** A "7/10" tells the next step nothing actionable: nobody can say what
   separates a 7 from an 8. A binary verdict forces a real judgment, and the
   *critique* carries all the nuance. The critique — not the verdict — is the
   signal that drives the loop. (Hamel, "LLM-as-a-Judge".)
2. **Success criteria are fixed at the start, before any output exists.** If a
   judge invents criteria *after* seeing the output, the evaluation is circular —
   it just rationalizes whatever was produced ("criteria drift",
   arXiv:2404.12272). Criteria are an input to the loop, immutable during it.
3. **A FAIL critique is injected verbatim into the next Implementer prompt.** Not
   summarized, not replaced with "try again" — the exact critique becomes a
   precise work order. This is what makes the correction tight.
4. **PASS critiques accumulate as few-shot examples for later Evaluator calls.**
   Early judgments teach the standard for later ones, so the judge gets *more*
   consistent as the run proceeds (continual in-context learning — no
   fine-tuning). The bank keeps growing on disk, but each Evaluator dispatch gets
   only a capped slice (~5 most relevant/recent blocks) — never the whole file —
   so token cost stays flat instead of growing with the run.
5. **The step budget is set by the caller, not by you.** This is the hard guard
   against runaway loops and cost explosions.
6. **A human checkpoint precedes any destructive action** (delete, overwrite,
   force-push, deploy, schema migration, `rm`, dropping data). Autonomy stops at
   irreversibility.
7. **Every subagent is stateless.** Pass all context explicitly. Never assume a
   subagent can see the outer conversation, a previous subagent's output, or
   prior iterations — it cannot. State that must survive lives in files and in
   the task list (TaskUpdate), which also survive context compaction.

## When NOT to run a loop

Loops add cost and non-determinism. Skip the loop and just do the work directly
when: the task is a single step with no feedback to observe; the output must be
deterministic; the goal is vague ("improve the codebase") rather than verifiable;
or the context is so large that a long loop would degrade. If the user's goal is
not verifiable or not bounded, say so and help them sharpen it before looping —
a good loop goal has a clear "done" signal (tests pass, file exists, lint clean)
and a clear "not done" signal (error output, failing assertion, missing file).

## The contract — inputs you need before looping

Gather these four before the autonomous phase. Derive what you safely can from
the conversation; ask the user only for what is genuinely missing.

| Input | What it is | If missing |
|---|---|---|
| **Goal** | One concrete, verifiable end-state. | Ask. Refuse vague goals; help sharpen. |
| **Success criteria** | The checks that define "done", per subtask and overall. Immutable during the loop. | Propose a draft from the goal; get the user's sign-off. |
| **Step budget** | Max loop steps. One step = one Implementer+Evaluator attempt — count it once per attempt in **every** branch (PASS, FAIL-retry, and the escalating attempt); a FAIL-retry is not free. Planner dispatches (initial and re-decomposition) and the final overall Evaluator pass do not count. A dispatch that fails again after one retry counts as one step. | Suggest an initial default based on rough complexity: 20 for likely small goals, 30-50 for broader multi-file goals. After Step 3 planning, compare planned subtasks to remaining budget (estimate 1-2 steps per subtask) and confirm whether to keep the budget, increase it, or prioritize a subset. |
| **Calibration examples** | 2–3 graded examples (a candidate output + the user's PASS/FAIL + their critique) that anchor the Evaluator to the user's standard. | Offer to draft examples for the user to correct, or proceed uncalibrated and say so. See `references/calibration-guide.md`. |

Also establish **constraints** (files/dirs that are off-limits or read-only, e.g.
"never edit test files"; style requirements) and the **working directory** the
loop is scoped to. Scoping the working directory is the main defense against
scope creep.

## Execution protocol

Follow these steps in order. Steps 0–2 are setup (some user interaction); steps
3–5 are the autonomous loop.

### Step 0 — Intake and goal-shaping

Confirm the goal is loop-appropriate (verifiable + bounded). State the goal,
criteria, budget, and constraints back to the user in 3–6 lines and get a quick
confirmation. If the goal isn't verifiable, fix that first.

### Step 1 — Calibration (anchor the Evaluator)

Run the calibration step in `references/calibration-guide.md`. The output is a
small set of graded examples you will pass to **every** Evaluator call as a
few-shot anchor. This is the single highest-leverage step for judgment quality —
do not skip it silently. If the user declines, proceed but note that the
Evaluator is uncalibrated.

If a reference file cannot be found or read, inform the user that the
agentic-loop reference files are missing and ask them to verify the
installation path. Do not proceed past Step 1 without access to
`references/loop-protocol.md`, as it contains the required dispatch templates.

If a reference file is found but its content is empty, unparseable, or
contradicts this prompt, treat this prompt as authoritative. Inform the user
that the reference file appears corrupted, proceed using this document's
protocol, and log the discrepancy in `state.json`.

### Step 2 — Initialize loop state

Create a run directory and the critique bank, and create tasks so progress
survives context compaction. Exact file layout, the `state.json` schema, and the
`critique-bank.md` format are in `references/loop-protocol.md` (read it now if
you have not). At minimum:

- Generate `<run-id>` with `date -u +%Y-%m-%dT%H%M%SZ` (a real UTC timestamp,
  seconds precision — you have no internal clock, so use the shell). Create
  `.agentic-loop/<run-id>/` in the working directory (suggest adding
  `.agentic-loop/` to `.gitignore`). Write `state.json` (goal, criteria, budget,
  step=0, constraints) and an empty `critique-bank.md`.
- `TaskCreate` one task per planned subtask once the Planner returns (Step 3), or
  a single tracking task now; use `TaskUpdate` after every iteration to persist
  the verdict, critique pointer, and step count.

If the run directory cannot be created (permission denied, disk full, or a
read-only filesystem), inform the user of the specific error and ask for an
alternative writable directory or for the permission issue to be resolved
before proceeding.

### Step 3 — Plan (decompose the goal)

Dispatch the **Planner** subagent (`subagent_type: magi:agentic-loop-planner`). Give
it the goal, the full criteria, the constraints, and the working directory. It
returns an ordered list of small, independently verifiable subtasks, each tagged
with the criterion it satisfies, its dependencies, and whether it is potentially
destructive. Persist the list to the run directory and create a task per subtask.
Use the exact dispatch template in `references/loop-protocol.md`.

Validate the Planner output before proceeding: if it returns zero subtasks, any
subtask has unresolvable circular dependencies, or a re-decomposition after
escalation produces a plan identical to one that already failed, stop and
escalate to the user with the Planner output and failure context.

If a re-decomposed plan contains more subtasks than the remaining step budget
can likely complete (estimate 1-2 steps per subtask), inform the user of the
budget shortfall and ask whether to increase the budget, prioritize a subset,
or exit with a partial report.

## Mid-loop user intervention

If the user provides new instructions or changes the goal during the autonomous
phase:

1. Pause the loop immediately.
2. Confirm the new goal or constraint.
3. If criteria changed, note that the Evaluator standard has shifted and
   re-run calibration if needed.
4. Update `state.json` and resume or restart planning as appropriate.

Never silently ignore user input during the loop.

### Step 4 — The loop

While there is an unfinished subtask AND `step < budget`:

```mermaid
stateDiagram-v2
   [*] --> Pick
   Pick --> Checkpoint
   Checkpoint --> Act
   Act --> Observe
   Observe --> Persist: step += 1 (every attempt)
   Persist --> Pass: VERDICT PASS
   Persist --> Fail: VERDICT FAIL
   Pass --> Pick
   Fail --> Retry: fail count 1-2 (no intervening PASS)
   Retry --> Act
   Fail --> Escalate: 3rd FAIL (no intervening PASS)
   Escalate --> Replan
   Replan --> Pick
   Pick --> [*]: no ready subtask or budget reached
```

1. **Pick** the next ready subtask (dependencies satisfied).
2. **Checkpoint if destructive** — if the subtask (or the Implementer's plan for
   it) involves a destructive action, pause and get explicit user approval before
   proceeding. Never delete/overwrite/deploy autonomously.
3. **Act** — dispatch the **Implementer** (`subagent_type:
   magi:agentic-loop-implementer`) with: the subtask, its success criterion, the
   constraints, the working directory, the **result file path**
   (`iterations/<NNN>-<id>-impl.md`), and — on a retry — the previous FAIL
   critique injected **verbatim** as the work order. It writes the full diff/trace
   to that result file and returns only a **compact** block (summary, files
   touched, result path, brief self-check, status) — the large diff stays in the
   file, out of your context.
4. **Observe** — first confirm the result file exists and is non-trivial; if it
   is missing or empty, treat it as a dispatch failure (retry the Implementer
   once, then escalate). Then dispatch the **Evaluator** (`subagent_type:
   magi:agentic-loop-evaluator`) with: the subtask criterion (upfront, immutable),
   the calibration examples, **at most ~5** PASS blocks from `critique-bank.md`
   (not the whole file), and the pointer to the Implementer's result file. The
   Evaluator **independently verifies against the repo** (`git diff`, re-reads the
   files, re-runs the test) — it does not trust the self-report or the result
   file alone — and returns `VERDICT: PASS|FAIL` plus a detailed critique.
5. **Persist step** — increment `step` and update `state.json` now, once per
   attempt, **before** branching on the verdict — so PASS, FAIL-retry, and the
   escalating attempt each cost exactly one step (a retry is never free).
6. **Decide:**
   - **PASS** -> append the critique to `critique-bank.md`, mark the subtask done
     (`TaskUpdate`), advance.
   - **FAIL** -> keep the critique. If this subtask has failed 1 or 2 times in
     the current attempt sequence (no intervening PASS), loop back to the
     **Act** step (4.3) for the same subtask, injecting this critique verbatim.
     On the 3rd FAIL without an intervening PASS, stop retrying: log the root
     cause from the critique and **escalate to the Planner** to re-decompose the
     remaining work with the failure history as context. Mark all dependent
     subtasks as BLOCKED; after re-decomposition, update dependency pointers for
     blocked subtasks to reference the new plan, and unblock any task whose
     dependency no longer applies.

If a subagent dispatch fails for reasons other than an unknown type (for
example timeout, malformed response, context overflow, or a missing/empty result
file), retry the same dispatch once with simplified or truncated context. If it
fails again, count it as one attempt (one `step`) for budget purposes, log the
error in `state.json`, and escalate to the user with the error details.

Prompt-assembly templates for all three dispatches are in
`references/loop-protocol.md`. Use them.

### Step 5 — Exit and report

The final Evaluator pass against the overall goal criteria (with a capped slice
of the PASS bank — ~5 blocks, not the whole file) is part of Step 4 and follows
the same retry/escalation rules. Step 5 executes only when no further iteration
is needed or possible.

After that final pass is complete and no subtask can be re-opened, the loop
ends in exactly one of three ways:

- **Goal met** -> success report: what was accomplished, the final verdict, and a
  short summary of the accumulated critiques (the standard that was enforced).
- **Budget exhausted** -> partial report: completed subtasks, remaining subtasks,
  and **FAIL root causes derived from the actual critiques** (never invented).
- **Goal not met but budget remains and no path forward** (e.g. blocked, or final
  eval fails) -> gap report: what's missing, why, and concrete next steps.

Exit-report templates are in `references/loop-protocol.md`.

For the exit double-check, **read the iteration files on demand** (the diffs live
in `iterations/<NNN>-impl.md`, not in your context) rather than relying on
anything you held in memory. If your exit review finds that a subtask previously
marked PASS does not actually meet its criterion, re-open that subtask and return
to Step 4 under the same rules if budget remains. If budget is exhausted, note the
discrepancy in the exit report under a "Post-loop review findings" heading.

## Subagent dispatch map

| Role | `subagent_type` | What it does |
|---|---|---|
| Planner | `magi:agentic-loop-planner` | Decompose goal into ordered, verifiable subtasks; re-decompose on escalation. |
| Implementer | `magi:agentic-loop-implementer` | Execute exactly one subtask; consume a FAIL critique as a work order on retry. |
| Evaluator | `magi:agentic-loop-evaluator` | Independently verify output; return binary PASS/FAIL + detailed critique. |
| Explorer | `Explore` (built-in) | Optional read-only codebase research to inform planning. Use the built-in agent. |

The three custom subagents ship with this skill in the **magi** plugin and are
namespaced `magi:<agent>`. If a dispatch fails because a `subagent_type` is
unknown, the `magi` plugin is not installed (or not enabled) — tell the user to
install it (`/plugin install magi@cc-magi-engineer`) and run `/reload-plugins`
before proceeding. Do not attempt a `general-purpose` fallback.

## Reference files

- `references/loop-protocol.md` — the full algorithm, `state.json` and
  `critique-bank.md` formats, the exact dispatch prompt templates for each
  subagent, retry/escalation rules, the destructive-action checkpoint protocol,
  and the three exit-report templates. Read this before Step 2.
- `references/calibration-guide.md` — how to run the pre-loop calibration:
  what to ask, how to format graded examples, and how to proceed if the user
  declines. Read this before Step 1.

## Guardrails recap (do not violate)

- Binary verdicts only — no numeric scores, no middle tiers.
- Criteria are fixed at intake; the Evaluator judges, it does not design criteria.
- FAIL critique -> next Implementer prompt, verbatim.
- PASS critiques accumulate on disk, but only a capped slice (~5 blocks) is fed to
  any Evaluator call — never the whole bank.
- The Implementer writes its full diff to its iteration file and returns only a
  compact pointer; large diffs never enter the orchestrator's context.
- Never exceed the caller's step budget; count one step per Implementer+Evaluator
  attempt (PASS, FAIL-retry, and escalation attempt alike).
- Human checkpoint before any destructive/irreversible action.
- Treat every subagent as stateless; pass all context explicitly; externalize
  state to files + TaskUpdate so it survives compaction.
- Review the actual diffs — do not trust a subagent's self-assessment; that is
  the Evaluator's entire job (it re-verifies against the repo, not the result
  file), and you double-check at exit by reading the iteration files on demand.
