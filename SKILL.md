---
name: task-splitter
description: >-
  Cost-optimized task segregation for building products and features. Captures the
  main task, saves it to a resumable ledger, splits it into atomic subtasks, scores
  each subtask's difficulty, routes it to the cheapest model + effort that can do it
  well (inline shell / haiku / sonnet / opus), batches subtasks that share the same
  model + effort into ONE subagent thread, runs independent batches in parallel waves,
  verifies with deterministic checks and escalates only failures one tier up.
  TRIGGER when: planning a product, feature or multi-step implementation; the user
  says "split this task", "segregate tasks", "break down and route", "optimize cost",
  "cheap models for easy parts", or invokes /task-splitter. Best used in plan mode.
  DO NOT TRIGGER for single-step tasks that need no decomposition.
argument-hint: "<main task description>"
---

# Task Splitter — cheapest correct execution

Goal: finish the main task with the **lowest total cost that still meets quality**.
Two levers: (1) route each piece of work to the cheapest capable tier, (2) pay each
subagent's cold-start context once by batching same-tier work into one thread.

Read `references/routing-rubric.md` before scoring and
`references/batch-brief-template.md` before writing any batch prompt.

## Hard rules

- **T0 work never spawns an agent.** Deterministic steps (install, mkdir, mv, a config
  value, running a generator) run inline in the main thread. A cold-start subagent costs
  more than the command.
- **Never use `subagent_type: "fork"` for cheap tiers.** A fork always runs on the parent
  model and ignores `model`. Use `general-purpose` (or `Explore` for read-only search) with
  an explicit `model` and `effort`.
- **This skill explicitly authorizes setting `effort`** on every Agent call it makes, using
  the tier table. Always pass both `model` and `effort`.
- **Verification by scripts, not models.** Tests, lint, typecheck and build are run by the
  main thread. Do not spend a model call on reviewing code a script can check.
- **Escalate only the failing subtask, one tier, once.** After one escalation, stop and
  ask the user.
- Never batch two subtasks that edit the same file into **different** batches of the same wave.

## Tier table (defaults — calibrate with the routing log)

| Tier | Score | Runs as | Typical work |
|---|---|---|---|
| T0 | 0 | main thread, inline shell | install deps, scaffold dirs, rename, set env/config value |
| T1 | 1–3 | `model: haiku`, `effort: low` | boilerplate following an existing pattern, copy edits, simple CRUD wiring, docs, fixtures |
| T2 | 4–6 | `model: sonnet`, `effort: medium` | normal feature code, components, endpoints, tests following existing patterns |
| T3 | 7–8 | `model: sonnet`, `effort: high` | multi-file logic, state management, integrations, tricky bugs, auth/payments/security |
| T4 | 9–10 | `model: opus`, `effort: high` | architecture, cross-cutting design, data model, hard debugging, perf/concurrency |

Relative cost weights for the preview: haiku = 1, sonnet = 3, opus = 5;
effort multiplier low = 1, medium = 1.5, high = 2.5. T0 = ~0.

## Workflow

### Phase 0 — Capture & persist
1. Restate the main task in ≤10 lines: goal, constraints, acceptance criteria, stack,
   out-of-scope.
2. Derive a slug. Look for an existing ledger in `<project>/.claude/task-splitter/`
   (`ls .claude/task-splitter/ 2>/dev/null`). If one matches this task and has unfinished
   subtasks → **resume from it** (skip to the first phase whose work is not done).
3. Otherwise create `<project>/.claude/task-splitter/<YYYY-MM-DD>-<slug>.md` from
   `templates/split-ledger.md` and fill the Context section. Update it after every phase.
   In plan mode, if writing files outside the plan file is not allowed, put the ledger
   content into the plan file under a `## Ledger` heading and write the real file as the
   first T0 step after approval.

### Phase 1 — Light scout
Read only what the split needs: manifest (package.json, pyproject, etc.), entry points,
the files the feature touches, existing utilities to reuse. If a broad search is needed,
use ONE `Explore` agent with `model: haiku`, `effort: low` and ask for paths + one-line
findings only. Record reusable utilities and exact file paths in the ledger — they become
the context packets that stop cheap agents from re-exploring.

### Phase 2 — Split
Break the work into atomic subtasks. Each subtask has:
`id` (S1, S2…), `goal` (one sentence), `files` (exact paths, new or existing),
`depends_on` (ids), `done_check` (a command like `pnpm test foo` or a precise criterion),
`type` (short tag: install, scaffold, component, endpoint, test, migration, refactor,
config, docs, debug, design…).
Atomic = one agent can finish it without decisions that belong to another subtask.
Pull design decisions out into their own early subtask (often T4) so later subtasks
are mechanical and cheap.

### Phase 3 — Score & route
Score each subtask with the 5-dimension rubric in `references/routing-rubric.md`
(ambiguity, reasoning depth, blast radius, context needed, novelty; 0–2 each), apply the
override rules, then map score → tier.
Then check `~/.claude/task-splitter/routing-log.md` (if it exists): if a `type` was
escalated in ≥2 of its last 3 runs at a tier, start it one tier higher; if it passed
first-time ≥3 times in a row one tier lower than you chose, consider dropping a tier.

### Phase 4 — Batch
1. Build dependency **waves**: wave 1 = subtasks with no unmet deps; wave N = those whose
   deps are all in earlier waves.
2. Inside each wave, group by `(model, effort)`, then by code area (same feature folder /
   module) so one agent loads related files once.
3. Limits per batch: ≤6 subtasks or one coherent area; no file edited by two batches in
   the same wave (move the conflicting subtask into the same batch or the next wave).
4. Merge tiny batches: a lone T1 subtask joins another T1 batch in the same wave if one
   exists; a lone T1 subtask that is nearly mechanical may be done inline instead.
5. T0 subtasks are grouped into a single inline step at the start of their wave.

### Phase 5 — Cost preview (approval gate)
Write into the plan file (or show the user, outside plan mode):

```
| Wave | Batch | Model / effort | Subtasks | Est. cost |
|------|-------|----------------|----------|-----------|
| 1    | inline| main / shell   | S1, S2   | ~0        |
| 1    | B1    | haiku / low    | S3, S4   | 2         |
| 1    | B2    | sonnet / medium| S5       | 4.5       |
| 2    | B3    | opus / high    | S6       | 12.5      |
Total ≈ 19 units  (all-opus/high baseline ≈ 75 units → ~75% cheaper)
```
Est. cost = number of subtasks × model weight × effort multiplier. Include the
all-opus/high baseline so the saving is visible. In plan mode, finish with ExitPlanMode;
do not execute before approval.

### Phase 6 — Execute
For each wave, in order:
1. Run the wave's T0 inline step.
2. Launch **all batches of the wave in ONE message** (parallel Agent calls):
   `Agent(subagent_type: "general-purpose", model: <tier model>, effort: <tier effort>,
   description: "<wave>/<batch>: <short>", prompt: <brief>)`.
   Build every brief from `references/batch-brief-template.md`. The shared preamble must
   be byte-identical across all batches (prompt-cache friendly) — put variable content
   after it.
3. Wait for every batch of the wave to report before starting the next wave. Never write
   up or guess a pending agent's result.
4. Update the ledger with each batch's status and reported files.

### Phase 7 — Verify & escalate
1. After each wave, the main thread runs the subtasks' `done_check` commands plus the
   project's cheap global checks (typecheck / lint / affected tests).
2. A failing subtask is re-run alone (or with other failures of the same new tier) at
   **one tier higher**, with the failing output and the previous agent's report attached.
3. Max one escalation per subtask. If it still fails, stop and ask the user with a short
   diagnosis. T4 failures go straight to the user.

### Phase 8 — Record & learn
1. Mark the ledger complete: per-subtask final tier, pass/escalated, files changed.
2. Append one line per subtask to `~/.claude/task-splitter/routing-log.md` (create the
   folder/file if missing; never inside the skill's git repo):
   `YYYY-MM-DD | <project> | <type> | <start tier> | <final tier> | first-pass|escalated|failed`
3. Report to the user in ≤10 lines: what was done, final cost units vs baseline, any
   escalations, anything left open.

## Output conventions
- Keep the main thread lean: agents return short reports, not file contents.
- Give the user tables, not prose, for the split and the cost preview.
- If the whole task scores ≤3 total, or has ≤2 subtasks, say so and just do it directly —
  splitting it would cost more than it saves.
