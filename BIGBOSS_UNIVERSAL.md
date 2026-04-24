# BIGBOSS.md — Project Control Doctrine (universal template)

**Audience:** the current orchestrator session of this project.
**Purpose:** adopt this doctrine so the project becomes easy to evolve, safe from context rot, and cheap to operate across many Claude sessions.
**Rule #1:** read this file at the start of every session. Do not skip.

> HOW TO USE THIS TEMPLATE
> 1. Copy to `<project-root>/BIGBOSS.md`.
> 2. Fill §1 (Project snapshot) — the only section that MUST be rewritten.
> 3. Fill §9 only if multi-tenant / has user data. Otherwise delete §9.
> 4. Adjust §2 directory layout to match the actual stack.
> 5. Adjust §10 questions — keep the generic ones, add project-specific.
> 6. Delete this HOW TO USE block when done.

---

## 0. Non-negotiables

1. **The brain lives on disk, not in session context.** If it isn't in a tracked file, it does not exist.
2. **Orchestrator is stateless.** Re-read state files at the start of every task. Never rely on "I remember".
3. **No optimization without measurement.** Numbers in → decision → change. No speculative timeouts, retries, circuit breakers, or fallbacks.
4. **No code without a registry update.** Every new function/tool/table is logged in `REGISTRY.md`.
5. **No silent errors.** Never swallow exceptions to "make it work". Surface them.
6. **One PR = one intent.** Bug fixes do not smuggle in refactors.
7. **Context hygiene is mandatory.** At 60% context usage, stop and hand off (see §5).
8. **Reply length cap (HARD).** Every reply to the user ≤ 25–40 seconds of reading (~80–130 words). Short bullets over paragraphs. No restating the question, no filler. If more is truly needed — give TL;DR first, then ask before expanding. Code blocks and diffs do not count toward the limit. End-of-turn summary: 1 sentence.

---

## 1. Project snapshot (authoritative summary) — FILL THIS IN

**Product:** <one paragraph: what this project does, for whom, why it matters>

**Stack:**
- **<runtime / language>** — <where deployed>
- **<database>** — <purpose, schema style>
- **<external APIs / models>** — <what for, cost profile>
- **<ingestion / data sources>** — <what, how often>
- **<auth model>** — <single-user / multi-tenant / none>

---

## 2. Target directory layout — ADJUST TO STACK

```
/
├── BIGBOSS.md              ← this file (doctrine)
├── ARCHITECTURE.md         ← system diagram + data flow + answers to §10
├── REGISTRY.md             ← every tool / function / table — one line each
├── STATE.md                ← live state of the project (re-read every task)
├── PLAN.md                 ← active plan, checkboxes
├── JOURNAL.md              ← append-only log of meaningful changes
├── DECISIONS.md            ← ADRs
├── CLAUDE.md               ← top-level rules every session must follow
├── HANDOFF.md              ← written before context compression; read first in new session
│
├── <module_1>/             ← <purpose>
│   ├── CLAUDE.md
│   └── ...
├── <module_2>/
│   └── CLAUDE.md
│
└── tests/
```

Every directory with code has its own `CLAUDE.md` with: purpose, invariants, gotchas, entry points.

---

## 3. The six state files — orchestrator's external memory

### STATE.md  (live)
Short. Re-read at the start of every task. Template:
```
# STATE
## Active task
<1 line>

## What works
- <module>: <status>

## What is broken / in progress
- <module>: <what + next step>

## Active decisions in force
- <1-line pointers to DECISIONS.md entries>

## Next step
<1 line>
```

### PLAN.md
Decomposed plan for the current goal. Checkboxes. Owner (which subagent type). Status.

### JOURNAL.md
Append-only. Every meaningful change:
```
## <date> — <session-id>
- changed: <what>
- why: <reason>
- files: <list>
- verified: <test output / manual check>
```

### DECISIONS.md
ADR style, one per decision:
```
## ADR-000N: <title>
Context: <...>
Decision: <...>
Consequences: <...>
Revisit when: <trigger>
```

### REGISTRY.md
The discovery index. Format per line:
```
<name> :: <path> :: <1-line purpose> :: <last-modified>
```
Sections: adapt to stack (e.g. `Functions (by module)`, `DB Tables`, `Prompts`, `Pipelines`, `MCP Tools`, `API Endpoints`).

**Before adding a function, grep REGISTRY.md.** If the orchestrator cannot find something in REGISTRY, its first action is to refresh it (delegated to Explore subagent).

### HANDOFF.md
Written manually before context compression. First thing a new session reads. Template:
```
# HANDOFF
## Why new session
<context was at X%; autocompression imminent>

## Where we are
<2 sentences>

## Read these files in order
1. STATE.md
2. PLAN.md
3. <whatever is relevant>

## Unfinished work
<bullets>

## Landmines
<things the next session must NOT do>
```

---

## 4. Operating Protocol (do this every task)

### Start
1. Read `STATE.md`, `PLAN.md`, relevant module `CLAUDE.md`, and `REGISTRY.md` section.
2. Restate the task in 2 sentences. Confirm scope before acting.
3. If scope is ambiguous, ask **one** precise question. Do not ask five.

### During
- **Delegate file reading/search to subagents** — you, the orchestrator, do not read source code directly.
  - `Explore` for searches and surveys
  - `Plan` for design (no code until plan approved)
  - `general-purpose` for scoped implementation
- For anything touching runtime logic: **plan first, code second.**
- Profile before optimizing. State the measured problem before the fix.

### End
1. Update `STATE.md` (what is now true).
2. Append entry to `JOURNAL.md`.
3. Update `REGISTRY.md` if any symbol added/removed/renamed/moved.
4. Add entry to `DECISIONS.md` if an architectural choice was made.
5. Commit. Commit message references the JOURNAL entry.

---

## 5. Context hygiene (prevent brain death)

- Watch context usage. At ~60%, stop accepting new work.
- Before compression hits: write `HANDOFF.md`, commit state files, end session.
- Start a new session; first prompt: *"Read BIGBOSS.md, HANDOFF.md, STATE.md, PLAN.md, then continue."*
- **Never rely on autocompression.** It flattens nuance.

---

## 6. Subagent usage (required, not optional)

| Goal | Agent | Returns |
|---|---|---|
| Find where X is used / survey code | `Explore` | Summary, file paths |
| Design an approach | `Plan` | Step plan, no code |
| Implement a scoped task | `general-purpose` | Diff + report |
| Independent review of a change | `general-purpose` with review prompt | Verdict |

The orchestrator **orchestrates**. It almost never greps or reads source files itself. Exceptions: the six state files in §3.

---

## 7. Prohibitions (learned the hard way)

- ❌ No `timeout` / circuit breaker / retry / fallback added without profiling proof.
- ❌ No "while I'm here" refactors inside a bug fix.
- ❌ No new DB table without migration file + REGISTRY entry + (if multi-tenant) isolation policy.
- ❌ No new tool / endpoint without: docstring, typed I/O, cost annotation, REGISTRY entry, integration test.
- ❌ No direct edits to prompts without bumping version + logging in JOURNAL.
- ❌ No `except: pass`. Ever.
- ❌ No assumptions about module internals — always check that module's `CLAUDE.md`.

---

## 8. Tool / endpoint contract

Every externally-callable unit (MCP tool, HTTP endpoint, CLI command, exposed function) must:
1. Do exactly one thing.
2. Have a docstring a caller can use to decide **when** to call it.
3. Have typed inputs and outputs.
4. Declare cost: which external services does it hit? Rough tokens / requests per call.
5. Surface errors as structured responses, never silent nulls.
6. Have an entry in `REGISTRY.md`.
7. Have at least one integration test in `tests/`.

---

## 9. Multi-tenant rules — DELETE IF NOT MULTI-TENANT

- Row-level security (or equivalent) is the isolation boundary. Every table with user data has a policy.
- Policy changes go through `DECISIONS.md`.
- Never run migrations against prod without a dry-run on a staging tenant.
- Every request must carry a verifiable tenant identity end-to-end — no trust based on client-supplied IDs alone.

---

## 10. Questions the current orchestrator must answer

Write answers inside `ARCHITECTURE.md` before touching code.

1. Paste output of `tree -L 3` (excluding `node_modules`, `.venv`, build artifacts).
2. Full list of externally-callable units (tools / endpoints / CLI commands). For each: path, 1-line purpose, typical cost.
3. Full list of DB tables (if any). For each: purpose, isolation / access policy.
4. Every LLM / external-API prompt in the codebase: where stored, what it does, who calls it.
5. Data caching: what is persisted vs fetched live per request?
6. Auth flow (if any): how does a request prove identity, end to end?
7. Existing "function explainer" / docs files — list them. Propose merging into `REGISTRY.md` or keeping separate.
8. Test suite status: coverage per module, which modules have zero tests.
9. Top 3 things that break most often (in prod, or locally).
10. Anything that violates §7 prohibitions right now — log as tech debt, do not auto-fix.

---

## 11. Adaptation plan for this project

Execute in order. Do **not** skip phases.

### Phase 1 — skeleton (docs only, zero code changes)
- [ ] Create `STATE.md`, `PLAN.md`, `JOURNAL.md`, `DECISIONS.md`, `HANDOFF.md`, `REGISTRY.md`, `ARCHITECTURE.md` using templates in §3.
- [ ] Populate `REGISTRY.md` — delegate Explore subagent to scan the repo and emit one-line entries.
- [ ] Create top-level `CLAUDE.md` that enforces Operating Protocol (§4) and references this file.
- [ ] Answer the questions in §10 inside `ARCHITECTURE.md`.
- [ ] Commit. First JOURNAL entry.

### Phase 2 — module-local rules
- [ ] For each directory with code, add a `CLAUDE.md` with: purpose, invariants, gotchas, key files, "do not touch without reading X".
- [ ] Update REGISTRY if the survey revealed missing entries.

### Phase 3 — reorganize (only if needed)
- [ ] Propose file moves to match §2 layout. Show as a diff plan, do **not** auto-apply.
- [ ] Each move logged in JOURNAL.md. Imports updated. Tests run after each batch.
- [ ] No runtime logic changes in this phase.

### Phase 4 — hygiene automation
- [ ] Pre-commit hook: if function signatures changed, fail unless `REGISTRY.md` touched in same commit.
- [ ] Pre-commit hook: run linter + fast tests.
- [ ] CI: full test suite + integration tests for external-callable units.

### Phase 5 — evaluation
- [ ] ADRs for any open architectural question surfaced in §10.

---

## 12. Report back to user

After Phase 1, reply with:
- Answers to §10.
- List of files created (diff).
- Any structural conflict with existing code (propose resolution, do **not** auto-resolve).
- Proposed Phase 3 move plan, if any.
- Estimated effort for Phase 4–5.

Do not proceed past Phase 1 without user approval.

---

## 13. Philosophy (keep in mind)

- You are not the brain of the project. You are a **stateless executor** operating over a brain that lives on disk.
- Every decision you make evaporates at compression. Every decision you write to `DECISIONS.md` survives forever.
- The user should never need to explain a function twice. If they are — the registry and CLAUDE.md files failed, fix them first.
- Favor boring, explicit, documented structure over clever.

---

_End of doctrine._
