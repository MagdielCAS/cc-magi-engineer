# Loop Protocol — mechanics, state, prompt templates, exit reports

This is the operational detail behind `SKILL.md`. Read it before initializing
state (Step 2). It covers: the loop algorithm, the on-disk state layout, the
exact prompt templates for dispatching each subagent (with correct context
injection), retry/escalation, the destructive-action checkpoint, and the three
exit reports.

## Table of contents

1. Loop algorithm (pseudocode)
2. On-disk state — run directory, `state.json`, `critique-bank.md`
3. Dispatch template — Planner
4. Dispatch template — Implementer
5. Dispatch template — Evaluator
6. Retry and escalation
7. Destructive-action checkpoint
8. Exit reports (success / budget-exhausted / gap)
9. Context-window hygiene

---

## 1. Loop algorithm (pseudocode)

```
GOAL               = <verifiable end-state>
CRITERIA           = <per-subtask + overall checks>     # immutable during the run
BUDGET             = <max steps, set by the caller>
CALIBRATION        = <2-3 graded examples + critiques>  # few-shot anchor, fixed
PASS_CRITIQUE_BANK = []                                  # grows during the run
STEP               = 0

# Step 3: plan
SUBTASKS = Planner(GOAL, CRITERIA, CONSTRAINTS, WORKDIR)   # ordered, verifiable

# Step 4: loop
while (any subtask not done) and (STEP < BUDGET):
    t = next ready subtask
    if t.destructive: human_checkpoint(t); if not approved: skip/abort
    impl_path = "iterations/<NNN>-<t.id>-impl.md"                     # Implementer writes its full diff here
    result = Implementer(t, t.criterion, CONSTRAINTS, WORKDIR, impl_path,
                         fail_critique = last_fail_critique_for(t))   # verbatim, or none
    if impl_path missing or trivial: treat as dispatch failure (see §6)
    verdict, critique = Evaluator(t.criterion, CALIBRATION,
                                  recent(PASS_CRITIQUE_BANK, 5), impl_path)  # independent repo re-verify
    STEP += 1; persist(state.json)      # every attempt costs one step — PASS, retry, or escalate
    if verdict == PASS:
        PASS_CRITIQUE_BANK.append(critique)
        mark t done; TaskUpdate(t, completed)
    else:  # FAIL
        record critique for t
        retries[t] += 1
        if retries[t] < 3:
            pass                # retry SAME subtask next iteration, inject this critique verbatim
        else:
            log root_cause(critique)
            SUBTASKS = Planner(remaining GOAL, CRITERIA, CONSTRAINTS, WORKDIR,
                               failure_history = critiques_for(t))     # re-decompose (Planner is free)

# Step 5: exit
final_verdict, final_critique = Evaluator(GOAL CRITERIA, CALIBRATION,
                                          recent(PASS_CRITIQUE_BANK, 5), whole-workspace)
if all done and final_verdict == PASS: success_report()
elif STEP >= BUDGET:                    budget_report()
else:                                   gap_report()
```

`STEP += 1` runs once per Implementer+Evaluator attempt, in **every** branch
(PASS, FAIL-retry, escalation) — so the budget genuinely bounds total work.
Re-planning after the 3rd FAIL uses a Planner dispatch, which does not count
(only Implementer+Evaluator attempts do). `recent(PASS_CRITIQUE_BANK, 5)` passes
at most the 5 most-relevant/most-recent PASS blocks, not the whole bank (§9).

Note: **you** (the orchestrator / main agent) run this loop. The Planner,
Implementer, and Evaluator are stateless one-shot dispatches via the Agent tool.
They never call each other.

---

## 2. On-disk state

Externalize everything that must survive context compaction. Use a per-run
directory in the working directory:

```
.agentic-loop/<run-id>/         # run-id from `date -u +%Y-%m-%dT%H%M%SZ`, e.g. 2026-06-29T143017Z
├── state.json                  # loop state (below)
├── subtasks.json               # Planner output, with status per subtask
├── critique-bank.md            # accumulated PASS critiques (few-shot bank)
├── calibration.md              # the 2-3 graded examples (anchor)
└── iterations/
    ├── 001-<subtask-id>-impl.md    # Implementer result + diff/trace
    ├── 001-<subtask-id>-eval.md    # Evaluator verdict + critique
    ├── 002-...
```

Suggest adding `.agentic-loop/` to `.gitignore`. Mirror key state into the task
list with `TaskUpdate` so the user sees live progress and it survives compaction.

### `state.json`

```json
{
  "run_id": "2026-06-29T143017Z",
  "goal": "Every .py file in src/ has a module docstring and per-function docstrings",
  "overall_criteria": ["..."],
  "budget": 20,
  "step": 0,
  "working_dir": "src/",
  "constraints": ["do not edit files under tests/", "follow PEP 257"],
  "calibrated": true,
  "current_subtask": null,
  "status": "running"
}
```

### `subtasks.json`

```json
[
  {
    "id": "st-1",
    "description": "Add a module docstring to src/parser.py",
    "criterion": "src/parser.py starts with a triple-quoted module docstring that names the module's responsibility",
    "depends_on": [],
    "destructive": false,
    "status": "pending",
    "retries": 0
  }
]
```
`status`: `pending | in_progress | done | escalated`.

### `critique-bank.md`

Append one block per PASS. These become few-shot exemplars of "what good looks
like" for later Evaluator calls.

```markdown
## PASS — st-1 — Add module docstring to src/parser.py
**Criterion:** src/parser.py starts with a module docstring naming its responsibility.
**Verdict:** PASS
**Critique:** The first statement is a triple-quoted docstring: "Parse raw call
transcripts into structured turns." It names the responsibility, is one line as
the summary, and matches PEP 257. Verified by reading lines 1-3 of the file. A
new engineer would understand what this module is for from the docstring alone.

---
```

---

## 3. Dispatch template — Planner

Send via Agent tool, `subagent_type: magi:agentic-loop-planner`. Fill every `<...>`.
The Planner is stateless: include everything.

```
You are decomposing a goal into an ordered list of small, independently
verifiable subtasks for an agentic loop.

GOAL:
<goal>

SUCCESS CRITERIA (immutable — your subtasks must collectively satisfy these):
<criteria, as a list>

CONSTRAINTS (hard limits — off-limits/read-only paths, style rules):
<constraints>

WORKING DIRECTORY (scope — do not plan work outside this):
<workdir>

<IF RE-DECOMPOSING after escalation, include:>
FAILURE HISTORY — a previous subtask failed 3x. Re-plan the REMAINING work with
this in mind. Do not repeat the approach that failed.
Remaining goal state: <what's left>
FAIL critiques for the stuck subtask (verbatim):
<the 3 FAIL critiques>

Return ONLY the subtask list as JSON matching this schema (no prose):
[{ "id", "description", "criterion", "depends_on": [ids], "destructive": bool }]
Rules: each subtask is one focused change; each maps to a concrete, checkable
criterion; order by dependency; mark destructive any subtask that deletes,
overwrites, force-pushes, deploys, or migrates data.
```

Persist the returned JSON to `subtasks.json` and `TaskCreate` one task per
subtask.

---

## 4. Dispatch template — Implementer

Send via Agent tool, `subagent_type: magi:agentic-loop-implementer`. One subtask only.

```
Execute exactly ONE subtask. You are stateless — everything you need is below.

SUBTASK:
<subtask.description>

SUCCESS CRITERION (what "done" means for this subtask):
<subtask.criterion>

CONSTRAINTS (must not violate):
<constraints>   # e.g. never edit tests/, follow the project's style

WORKING DIRECTORY:
<workdir>

RESULT FILE PATH (write your full diff/trace HERE, then return only the compact
block below — do not paste the diff into your reply):
iterations/<NNN>-<subtask-id>-impl.md

<ON RETRY ONLY — inject the previous Evaluator FAIL critique verbatim:>
A previous attempt FAILED evaluation. This critique is your work order — address
it specifically and completely. Do not start over conceptually; fix what it
identifies:
"""
<verbatim FAIL critique>
"""

DESTRUCTIVE-ACTION RULE: if completing this requires deleting, overwriting,
force-pushing, deploying, or migrating data, do NOT do it. Stop and return
status "needs_checkpoint" describing the destructive action and why it's needed.

Write the full DIFF / TRACE (actual diff + commands + their real output) to the
RESULT FILE PATH. Then return ONLY this compact block:
- SUMMARY: one line — what you changed and how it satisfies the criterion
- FILES TOUCHED: paths
- RESULT_PATH: the RESULT FILE PATH you wrote
- SELF-CHECK: one or two lines of evidence (exit code, key line) — not the diff
- STATUS: done | needs_checkpoint | blocked(reason)
```

The Implementer writes the full result to
`iterations/<NNN>-<subtask-id>-impl.md`; you keep only its compact return. Before
dispatching the Evaluator, confirm that file exists and is non-trivial — if it is
missing or empty, treat it as a dispatch failure (§6) and retry the Implementer.

---

## 5. Dispatch template — Evaluator

Send via Agent tool, `subagent_type: magi:agentic-loop-evaluator`. This is the
judgment that gates the loop. Inject criterion + calibration + a capped slice of
the PASS bank (at most ~5 blocks — see §9; never the whole file).

```
You are an impartial Evaluator. Return a BINARY verdict (PASS or FAIL) and a
detailed critique. No numeric scores. No middle tiers. The critique is the
product — make it good enough that a new engineer could act on it.

SUCCESS CRITERION (the ONLY standard — do not invent new criteria):
<subtask.criterion>

HOW THE USER GRADES (calibration anchor — match this judgment style):
<contents of calibration.md: each example = candidate + PASS/FAIL + critique>

WHAT GOOD LOOKS LIKE SO FAR (up to ~5 recent/relevant PASS critiques from this
run — use as additional few-shot guidance for consistency):
<at most ~5 PASS blocks from critique-bank.md>   # may be empty early in the run

OUTPUT UNDER EVALUATION:
<RESULT_PATH: iterations/<NNN>-...-impl.md (a pointer — may be compact), plus the
files/paths it changed>

YOUR PROCESS:
1. INDEPENDENTLY VERIFY against the criterion USING THE REPO as source of truth —
   re-diff the changed files (`git diff`), re-read them, re-run the test/linter
   yourself. The result file is only a pointer; do NOT trust the Implementer's
   self-report. Missing changed files or a missing result file is a FAIL.
2. Decide PASS or FAIL strictly against the criterion (not against your own
   preferences, not against criteria the user didn't state).
3. Write the critique: cite specific evidence (file lines, command output). If
   FAIL, the critique must be a precise, actionable work order — exactly what is
   wrong and what would make it pass. If PASS, explain why it satisfies the
   criterion (this becomes a teaching example for later evaluations).

Return exactly:
VERDICT: PASS | FAIL
CRITIQUE: <detailed, evidence-cited, no scores>
```

Save to `iterations/<NNN>-<subtask-id>-eval.md`. On PASS, append the critique to
`critique-bank.md` (Section 2 format). On FAIL, keep it for the next Implementer
prompt.

---

## 6. Retry and escalation

- A FAIL retries the **same** subtask. Inject the FAIL critique verbatim into the
  next Implementer prompt (Section 4). Do not paraphrase; do not substitute a
  generic "please try again".
- Max **2 retries** (i.e. up to 3 Implementer attempts) per subtask.
- On the **3rd consecutive FAIL**: stop retrying. Extract the root cause from the
  critiques, set the subtask `status: escalated`, and re-dispatch the **Planner**
  with the failure history (Section 3, re-decompose branch). The Planner re-plans
  the remaining work given what didn't work. Resume the loop on the new plan.
- Each Implementer+Evaluator attempt is one iteration; increment `STEP` exactly
  once per attempt, immediately after the Evaluator call and before the PASS/FAIL
  branch (§1) — so PASS, FAIL-retry, and the escalating attempt each cost one
  step and the budget genuinely bounds total work. The Planner re-dispatch on
  escalation is free (it is not an Implementer+Evaluator attempt).
- **Dispatch failure** (Implementer/Evaluator times out, returns malformed
  output, or the result file is missing/empty): retry the same dispatch once with
  simplified context. If it fails again, count it as one attempt (one `STEP`), log
  the error in `state.json`, and escalate to the user.

---

## 7. Destructive-action checkpoint

Before executing any subtask flagged `destructive`, OR whenever an Implementer
returns `STATUS: needs_checkpoint`, pause the loop and get explicit user
approval. Present:

```
HUMAN CHECKPOINT — destructive action required
Subtask: <id> — <description>
Action: <delete/overwrite/deploy/migrate — be specific about what and where>
Why it's needed: <reason from the Implementer or plan>
Reversible? <yes/no; if yes, how to undo>
Proceed? (yes / no / modify)
```

Continue only on explicit "yes". On "no" or no response, skip the subtask and
record it as blocked in the exit report. This is the one place the loop is not
autonomous — by design (autonomy stops at irreversibility).

---

## 8. Exit reports

### Success (goal met, final Evaluator PASS)

```markdown
# Agentic Loop — SUCCESS
**Goal:** <goal>
**Steps used:** <step> / <budget>
**Subtasks completed:** <n>/<n>

## Final verdict
PASS — <one-line from the final Evaluator critique>

## What was done
- <subtask> — <one line>

## Standard enforced (critique summary)
<2-4 themes distilled from critique-bank.md — the bar each subtask had to clear>
```

### Budget exhausted

```markdown
# Agentic Loop — STOPPED (budget exhausted)
**Goal:** <goal>
**Steps used:** <budget>/<budget>

## Completed
- <subtask> — done

## Remaining
- <subtask> — <status: pending/escalated/blocked>

## FAIL root causes (from the actual critiques — not invented)
- <subtask>: <root cause quoted/derived from its FAIL critique(s)>

## Recommended next step
<the single most useful thing to do next, e.g. raise the budget, re-scope, or
fix a blocking dependency>
```

### Gap (goal not met, budget remains, no path forward)

```markdown
# Agentic Loop — INCOMPLETE
**Goal:** <goal>
**Why stopped:** <blocked dependency / checkpoint declined / final eval FAIL>

## Gap
<what's missing vs. the goal criteria>

## Why
<evidence from the relevant critiques or blockers>

## Concrete next steps
- <step 1>
- <step 2>
```

---

## 9. Context-window hygiene

Long loops degrade as early observations fall out of context. Mitigations,
already baked into this protocol:

- Subagents are stateless one-shots — they never inherit the growing transcript;
  you pass them only the slice they need.
- The Implementer writes its full diff/trace to `iterations/<NNN>-impl.md` and
  returns only a compact pointer — so large diffs never enter your (the
  orchestrator's) context. Read an iteration file on demand (e.g. for the exit
  double-check); don't hold every diff.
- All durable state is in files (`state.json`, `subtasks.json`,
  `critique-bank.md`, `iterations/`) and the task list — re-read them instead of
  relying on memory.
- The step budget bounds total length by design.
- **Hard cap on the PASS bank:** pass **at most ~5** PASS blocks to any Evaluator
  dispatch — the most relevant to the current subtask if easily identified, else
  the most recent. Never send the whole `critique-bank.md`; resending the full
  (growing) bank on every dispatch is O(n²) token growth across the run. The bank
  keeps accumulating on disk; only what you *send* is capped.
