---
description: Batch-implement a chain of specs from start to finish. Given a spec prefix, discovers all matching specs in `.claude/specs/` sorted by numeric suffix, runs `/mol:impl` on each non-interactively, and auto-commits between specs. Thin orchestrator — all gates, TDD, simplify, and verdict logic lives in `/mol:impl`. Never stops to ask questions.
argument-hint: "<spec-prefix>"
---

# /mol:impl-all — Batch Spec Chain Driver

Drive an entire spec chain (`<base>-01-<phase>`, `<base>-02-<phase>`, …) by iterating `/mol:impl` on each spec in order, committing between specs. All gating, testing, simplifying, and status advancement is `/mol:impl`'s job — this skill only discovers the chain and keeps it moving.

```
/mol:impl-all morse-bond
→ finds morse-bond-01-potential, morse-bond-02-gradient, morse-bond-03-optimization
→ /mol:impl 01 → /mol:commit → /mol:impl 02 → /mol:commit → /mol:impl 03 → /mol:commit
→ reports chain verdict
```

**Never asks questions.** Fully autonomous.

---

## Procedure

### 1. Discover the chain

Read `CLAUDE.md` → parse `mol_project:` (`$META`); else emit adoption hint and stop.

Scan `$META.specs_path` (default `.claude/specs/`) for files matching `<prefix>-<NN>-<phase>.md` where `<NN>` is a two-digit number. Include a bare `<prefix>.md` at position 00 if present. Sort ascending by `<NN>`.

At least 1 spec must exist. If none: *"no specs matching '<prefix>' found"* and stop.

Print the chain:

```
[mol:impl-all] chain: morse-bond (3 specs)
  01-potential      status: approved
  02-gradient       status: approved
  03-optimization   status: approved
```

Any spec with `status: draft` → refuse and stop (*"approve draft specs first: <list>"*).

### 2. Drive the chain

For each spec, in sorted order:

```
═══ [mol:impl-all] 2/3: morse-bond-02-gradient ═══
```

Invoke `/mol:impl <slug>`. `/mol:impl` owns all guardrails (stage gate, science gate, scope classification, TDD, simplify, acceptance criteria, status advancement). Do not duplicate them here.

After `/mol:impl` completes, check the result:

- Spec `status: done` (deleted by `/mol:impl`) → record done, run `/mol:commit` with message `feat(<base>): implement <phase> (<NN>/<total>)`.
- Spec `status: code-complete` → `/mol:impl` parked it for runtime evaluators. Record parked, run `/mol:commit`, continue to next spec.
- Spec still `in-progress` or `approved` (didn't advance) → `/mol:impl` hit a blocker. Stop the chain.

### 3. Report

```
═══ [mol:impl-all] chain verdict ═══

  morse-bond-01-potential      done
  morse-bond-02-gradient       done
  morse-bond-03-optimization   code-complete (pending: ui_runtime → /mol:web)

  2 done, 1 parked, 0 failed
```

If any specs are parked, print the runtime evaluator command to run next.

---

## Guardrails

- **Never ask questions.** Autonomous batch mode — drive forward, report at end.
- **Don't duplicate `/mol:impl`.** All gates, TDD, simplify, acceptance, and status logic belongs to `/mol:impl`. This skill only iterates and commits.
- **Stop on stall.** If `/mol:impl` doesn't advance a spec's status, stop the chain — later specs may depend on it.
- **Commit every spec.** Each spec gets its own checkpoint so the chain is reviewable mid-flight.
