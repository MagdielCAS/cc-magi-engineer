# Improve `agentic-loop` — token efficiency + correctness (prose-only)

## Context

The `agentic-loop` skill (now shipped in the `magi` plugin) drives a stateful
orchestrator that dispatches three stateless subagents (Planner, Implementer,
Evaluator) in a Reason→Act→Observe loop, externalizing state to
`.agentic-loop/<run-id>/`. It works, but a review (plus an independent red-team
pass) surfaced three concrete problems worth fixing and one latent bug:

1. **Biggest token sink — diffs in the orchestrator transcript.** The Implementer
   both *saves* its result to `iterations/<NNN>-impl.md` **and** *returns* the full
   DIFF/TRACE to the orchestrator (`loop-protocol.md` §4, lines 219 & 224;
   Implementer return format lines 69–84). So the largest payload lands in the
   orchestrator's context every iteration and never leaves.
2. **Cumulative O(n²) sink — the PASS bank is resent whole.** Every Evaluator
   dispatch includes the *entire* growing `critique-bank.md` (`loop-protocol.md`
   §5, lines 244–246). §9's "pass the most relevant exemplars" (lines 381–382) is
   only advisory, so in practice the whole file is resent each time.
3. **Step-accounting bug.** In `loop-protocol.md` §1 pseudocode, a FAIL-retry hits
   `continue` (line 50) which jumps back to the `while` and **skips** `STEP += 1`
   (line 55). Retries are therefore free against the budget — directly
   contradicting §6 ("increment STEP once per attempt so the budget genuinely
   bounds total work", lines 283–284) and `SKILL.md` Step 4.6.

**Decided scope:** prose-only, targeted edits. No helper script (a `loop.py` was
evaluated and rejected: in a distributed plugin it needs unreliable
install-path resolution, triggers a permission prompt on every call — breaking
autonomy — and forces a second code path the prose must mirror byte-for-byte).
The reasoning prose ("why the loop is shaped this way") stays intact.

**Intended outcome:** the orchestrator's per-iteration context stops growing with
run length, the Evaluator prompt stops growing quadratically, and the budget
guard actually bounds total work — with zero new dependencies.

## Changes

### Change 1 — Slim the orchestrator's context (the big token win)

Move large diffs out of the orchestrator entirely. The Implementer already has
the `Write` tool; the Evaluator already independently re-verifies against the
real repo.

- **`agents/agentic-loop-implementer.md`** — rewrite the "Return format —
  structured" section (lines 69–84): the Implementer **writes** its full
  `DIFF / TRACE` to the iteration file it is given, and **returns only** a compact
  block: `SUMMARY` (one line), `FILES TOUCHED`, `RESULT_PATH`, a brief
  `SELF-CHECK` pointer, and `STATUS`. The full diff no longer travels back to the
  orchestrator. Keep `needs_checkpoint` / `blocked` statuses and the
  destructive-action rule unchanged.
- **`references/loop-protocol.md` §4 (Implementer dispatch template)** — add a
  `RESULT FILE PATH:` line to the template instructing the Implementer to write
  its full diff/trace there and return only the compact summary + that path.
  Update the trailing "Save the full result to `iterations/...`" note (line 224)
  to state the *Implementer* saves it, not the orchestrator.
- **`agents/agentic-loop-evaluator.md`** — tighten "Independently verify" (lines
  46–53) to make repo re-diffing **mandatory**: the impl file is a pointer; the
  **repo is the source of truth** (`git diff` / re-read changed files / re-run the
  test yourself). A thin or stale impl file must not be able to cause a false
  PASS.
- **`references/loop-protocol.md` §5 (Evaluator dispatch template)** — point
  `OUTPUT UNDER EVALUATION` at the impl file **and** require the independent repo
  verification; note the impl file may be compact.
- **`SKILL.md` Step 4 (lines ~197–208) + Step 5 (lines ~234–254) + Guardrails
  recap (lines ~290–292)** —
  - Reflect the compact Implementer return (orchestrator no longer holds the
    diff).
  - **Add an impl-file guard:** before dispatching the Evaluator, verify the
    iteration file exists and is non-trivial; if missing/empty, treat as a
    dispatch failure and use the existing retry path (lines 224–228).
  - Exit double-check now **reads iteration files on demand** rather than relying
    on diffs held in context.

### Change 2 — Cap the PASS bank sent to the Evaluator (kills O(n²))

Keep `critique-bank.md` accumulating (the few-shot principle is sound); only cap
what is *passed* per dispatch.

- **`references/loop-protocol.md` §9 (lines 381–382)** — turn the advisory into a
  hard cap: pass **at most ~5** PASS blocks to any Evaluator dispatch (most
  relevant to the current subtask if easily identified, else most recent).
- **`references/loop-protocol.md` §5 template** — change "WHAT GOOD LOOKS LIKE SO
  FAR" to inject at most ~5 blocks, not the whole file.
- **`SKILL.md`** — one-line note where the PASS bank is described (the "why" point
  4, and Step 4.4) that the bank is capped when sent; the accumulate-as-few-shot
  behavior is unchanged.

### Change 3 — Fix step-accounting + reconcile budget semantics

- **`references/loop-protocol.md` §1 pseudocode (lines 37–55)** — move
  `STEP += 1; persist(state.json)` to run **immediately after the Evaluator
  call**, before the `if/else`, and **delete the `continue`**. Add the missing
  explicit `retries[t] += 1` on FAIL. Result: every Implementer+Evaluator attempt
  counts once, in all three branches (PASS / retry / escalate). Planner
  re-dispatch on escalation stays free (consistent with `SKILL.md` line 83).
- **`references/loop-protocol.md` §6** — align wording so it matches the corrected
  pseudocode (it already says "once per attempt"; ensure no residual
  contradiction).
- **`SKILL.md` Step 4.6 + the Step-budget row of the contract table (line 83)** —
  state once, unambiguously: increment step after **every** attempt (PASS,
  FAIL-retry, or escalation attempt); Planner dispatches and the final overall
  Evaluator pass do not count.

### Change 4 — Real run-id clock (one line, no script)

- **`SKILL.md` Step 2 + `references/loop-protocol.md` §2 (line 78)** — specify the
  orchestrator generate `run-id` via `date -u +%Y-%m-%dT%H%M%SZ` (Bash), using
  **seconds** precision to avoid same-minute collisions (current example is
  minute precision). LLMs have no clock; this is the one deterministic bit that
  genuinely needs a shell call, and Bash is already in scope.

## Files to modify

- `plugins/magi/skills/agentic-loop/references/loop-protocol.md` — §1, §2, §4, §5,
  §6, §9 (the bulk of the mechanics).
- `plugins/magi/skills/agentic-loop/SKILL.md` — Step 2, Step 4, Step 5, contract
  table, Guardrails recap.
- `plugins/magi/agents/agentic-loop-implementer.md` — return format.
- `plugins/magi/agents/agentic-loop-evaluator.md` — mandatory repo re-diff.

`calibration-guide.md` and the Planner agent are unchanged.

## Verification

**Static (read-back + grep after edits):**
- `references/loop-protocol.md` §1 has no `continue` on the FAIL-retry branch and
  `STEP += 1` sits above the `if/else`; an explicit `retries` increment exists.
- Implementer return format no longer contains a full `DIFF / TRACE` payload —
  only `SUMMARY / FILES TOUCHED / RESULT_PATH / SELF-CHECK / STATUS`.
- Evaluator states repo re-diff is mandatory ("repo is the source of truth").
- §5 and §9 both cap the PASS bank at ~5 blocks.
- run-id uses `%H%M%SZ` (seconds).

**Dynamic (real bounded run):** reinstall and reload the plugin
(`/plugin marketplace add .` → `/plugin install magi@cc-magi-engineer` →
`/reload-plugins`), then in a scratch dir run a tiny goal, e.g. *"add a module
docstring to two throwaway .py files, budget 6, criteria = each file starts with
a triple-quoted docstring naming its responsibility."* Confirm:
- the run-id in `.agentic-loop/` includes seconds;
- Implementer dispatches return compact summaries + a path (no full diffs echoed
  into the orchestrator transcript); the diffs live in `iterations/*-impl.md`;
- the Evaluator re-reads/re-checks the files itself;
- forcing one FAIL (e.g. an intentionally empty docstring) increments the step
  count and injects the FAIL critique verbatim into the retry.

**Structural (no regressions):** run the repo's CI-equivalent checks — validate
`marketplace.json` and each `plugin.json` with `jq`, confirm the skill `name`
still matches its directory and agent `name:` fields match filenames.
