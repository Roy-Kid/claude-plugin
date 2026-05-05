---
description: Pre-commit / pre-push / pre-merge CI-parity gate. Runs a tiered set of checks (format, lint, tests, full CI equivalent) and reports PROCEED or BLOCK. Never writes code.
argument-hint: "<commit | push | merge>"
---

# /mol:ship — CI Parity Gate

Read CLAUDE.md. Parse `mol_project:` (`$META`).

This skill is **read-only**. It does not edit code, write tests, or
run `git` state-changing commands. It executes the appropriate
pre-flight checks and reports PROCEED or BLOCK so the caller (human or
another skill) knows whether it is safe to `git commit`, `git push`,
or merge.

## Tier selection

The first positional argument selects the gate. The tiers are
cumulative — `merge` ⊇ `push` ⊇ `commit`.

- `commit` — format + lint + pre-commit hooks. Run before every
  `git commit`. Fast (~60s).
- `push` — adds the full test suite. Run before every `git push`.
  Medium (5–10 min).
- `merge` — mirrors the remote CI pipeline locally. Run before
  merging into `main`/`master`. Heavy; budget the full CI wall-clock.

If `$ARGUMENTS` is empty, default to `commit` and state the assumption
in the output.

## Procedure

1. **Resolve the tier** from `$ARGUMENTS`.

2. **Delegate to the `ci-guard` agent** with the tier as input. The
   agent detects CI config, runs the tier's commands, interprets
   failures, and reports CI-only drift modes (platform, matrix,
   secrets, cache, services).

3. **Aggregate** the agent's findings into a severity table:

   | 🚨 Critical | 🔴 High | 🟡 Medium | 🟢 Low |
   |-------------|---------|-----------|--------|
   | N           | N       | N         | N      |

4. **Decide** the verdict from the tier and the findings:

   - `commit` tier + any 🚨 → **BLOCK COMMIT**
   - `push` tier + any 🚨 → **BLOCK PUSH**
   - `merge` tier + any 🚨 or 🔴 → **BLOCK MERGE**
   - No blocker at the requested tier → **PROCEED**
   - 🟡 / 🟢 are informational; they never block.

5. **Route fixes.** For each blocker, name the `/mol:*` skill that
   should address it (not this skill — this skill refuses to edit):

   - lint / format → `/mol:fix <file>` with the failing rule
   - failing test → `/mol:fix` (if the test was expected to pass) or
     `/mol:impl` (if the feature is incomplete)
   - architecture violation surfaced by CI → `/mol:review --axis=arch`
     then `/mol:refactor`
   - doc drift → `/mol:docs`
   - CI-config drift (matrix / secrets / runner) → ask the user to
     update `$META.ci` or the workflow file itself — never edit
     workflows automatically

6. **Never auto-fix.** This skill refuses to edit code even when the
   fix is trivial. Hand the user a concrete next action instead.

## Output

```
Gate: <tier>
Verdict: PROCEED | BLOCK

(severity table)

(ci-guard findings verbatim, severity-sorted)

Next steps:
- <action 1>
- <action 2>
```

End with a one-line summary suitable for scanning:

```
/mol:ship <tier>: PROCEED (N findings, none blocking)
```

or

```
/mol:ship <tier>: BLOCK — <top blocker in one phrase>
```
