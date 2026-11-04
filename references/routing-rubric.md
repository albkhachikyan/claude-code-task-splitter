# Routing rubric

Score every subtask on five dimensions, 0–2 each. Total 0–10 → tier.

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| **Ambiguity** | Exact steps known (a command, a value) | Clear goal, small choices left | Open-ended; needs product or design judgment |
| **Reasoning depth** | No reasoning, just mechanical | Local logic in one function or file | Multi-step logic, algorithms, tricky edge cases |
| **Blast radius** | Easy to undo, isolated | Affects one feature | Shared code, data, auth, money, prod config |
| **Context needed** | None beyond the brief | A few files | Many files, or an understanding of the whole system |
| **Novelty** | Copies an existing pattern in the repo | Adapts a pattern | No precedent in the repo, or a new library/API |

## Score → tier

| Score | Tier | model / effort |
|---|---|---|
| 0 | T0 | inline shell in main thread |
| 1–3 | T1 | haiku / low |
| 4–6 | T2 | sonnet / medium |
| 7–8 | T3 | sonnet / high |
| 9–10 | T4 | opus / high |

## Override rules (applied after scoring)

- **Floor T3:** auth, permissions, payments/billing, secrets, security fixes, data
  migrations, concurrency/locking, anything irreversible in production.
- **Floor T4:** architecture, the data model or schema design, the public API contract,
  cross-cutting refactors (more than about 10 files), perf work that needs profiling, bugs
  with no known repro.
- **Cap T2:** tests that follow an existing test pattern, CRUD that copies an existing
  endpoint, UI that copies an existing component.
- **Cap T1:** docs, comments, README, changelog, fixtures/seed data, i18n strings,
  renames that a codemod can't do safely.
- **Force T0:** anything one deterministic command can do (package install, codegen,
  formatter, `sed` rename, mkdir/mv, setting a known env/config value).
- If the user marks a subtask as critical, raise it one tier.

## Worked examples

| Subtask | A | R | B | C | N | Score | Tier |
|---|---|---|---|---|---|---|---|
| `pnpm add zustand` | 0 | 0 | 0 | 0 | 0 | 0 | T0 inline |
| Create `src/features/theme/` folder + index barrel | 0 | 0 | 0 | 0 | 0 | 0 | T0 inline |
| Add 12 i18n keys to `en.json` from a given list | 0 | 0 | 0 | 1 | 0 | 1 | T1 haiku/low |
| New `ThemeToggle` button that copies `LangToggle` | 0 | 1 | 0 | 1 | 0 | 2 | T1 haiku/low |
| Zustand theme store with persistence + system pref | 1 | 1 | 1 | 1 | 1 | 5 | T2 sonnet/medium |
| Unit tests for theme store following existing tests | 0 | 1 | 0 | 1 | 0 | 2 | T1 haiku/low |
| Stripe webhook handler with idempotency | 1 | 2 | 2 | 1 | 1 | 7 | T3 sonnet/high (floor T3) |
| Decide multi-tenant data model + migration plan | 2 | 2 | 2 | 2 | 2 | 10 | T4 opus/high |
| Fix flaky race in WebSocket reconnect, no repro | 2 | 2 | 1 | 2 | 1 | 8 | T4 (floor: no repro) |

## Calibration from the routing log

`~/.claude/task-splitter/routing-log.md` lines look like this:
`YYYY-MM-DD | project | type | start tier | final tier | first-pass|escalated|failed`

- A `type` escalated in at least 2 of its last 3 runs at tier X: start it at X+1.
- A `type` that passed first time at least 3 runs in a row at tier X: you may start it at X-1 (never below
  the override floor).
- Mention any calibration you applied in the cost preview, e.g. "S5 endpoint: T2→T3
  (log: escalated 2/3)".
