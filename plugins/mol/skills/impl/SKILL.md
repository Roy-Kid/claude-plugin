---
description: Implementation workflow — scope → spec → acceptance gate → TDD → implement → verify. Use after `/mol:spec` has produced an approved spec + acceptance file; writes code, tests, and docs while ticking the spec's Tasks/acceptance checkboxes.
argument-hint: "<feature description or path to spec file>"
---

# /mol:impl — Implementation

Read CLAUDE.md → parse `mol_project:` (`$META`); else print adoption hint and stop. Read `$META.stage` (default `experimental`, with warning per `plugins/mol/rules/stage-policy.md`). Print `[mol] stage: <value>`.

Stage hooks owned by this skill (every other stage decision delegates to `/mol:simplify` Step 6.5 — the single backward-compat gatekeeper, per `plugins/mol/rules/stage-policy.md`):

- **Pre-work gate** — `maintenance` refuses unless spec frontmatter has `kind: bugfix`. Print refusal from rule doc and stop.
- **Commit-message decoration** (`beta`) — Step 7 auto-commit body must include a one-paragraph migration note when the diff modifies a public signature.

## Step 1 — Assess scope

Classify against `$META.arch.style`:

- **SMALL** — single new symbol, existing pattern, < 3 files. Skip to Step 3.
- **MEDIUM** — 3–8 files, new pattern, fits existing layers. Start at Step 2.
- **LARGE** — new top-level concept (driver / package / layer / frame field). Also triggers the large-spec split rule (`plugins/mol/rules/large-spec-split.md`); if the spec is monolithic LARGE, stop and re-invoke `/mol:spec` to produce a chain. Otherwise continue from Step 2 on the current sub-spec.

State the classification and confirm with user.

## Step 1.5 — Auto-branch (chained / LARGE)

Per `plugins/mol/rules/large-spec-split.md`, when slug matches `<base>-<NN>-<phase>` **or** scope is LARGE, ensure work happens on a feature branch without prompting.

1. Default branch = `upstream`'s default if it exists, else `main`, else `master`.
2. If on default branch, create/checkout `feat/<base>` (or check it out if it exists and is reachable). No prompt; print `/mol:impl: switched to branch feat/<base>`.
3. If on any other branch, stay (user positioned deliberately).

Silent for SMALL/MEDIUM on a single non-chained spec.

## Step 2 — Domain & spec (MEDIUM / LARGE)

- Equations or physics + `$META.science.required: true` → invoke `/mol:litrev`. Abort if no credible basis.
- `$ARGUMENTS` is a path under `$META.specs_path` → read that file as spec.
- Else → invoke `/mol:spec`, then read the result.

### 2a. Status gate (HARD)

Read spec frontmatter `status:`:

- `draft` — refuse. User deferred approval mid-`/mol:spec`; tell them to re-run `/mol:spec <slug>`.
- `approved` — first run; continue to 2a-bis.
- `in-progress` — previous run interrupted **or** out-of-band work; continue to 2a-bis (sync at 2c reconciles).
- `code-complete` — code done; spec parked for runtime evaluators. **Skip Steps 1–6**, jump to Step 7 finalize: re-read every criterion's `status`. All `verified` → advance to `done` and run close-out deletion. Any non-`code`/`runtime` still `pending` → list with `evaluator_hint` and exit cleanly. A `code`/`runtime` reverted to `pending` (impl modified out-of-band) → drop back to `in-progress` and continue from 2c.
- `done` — Step 7 should have deleted this. Warn and stop; user deletes by hand or runs `/mol:spec` for a new one.

### 2a-bis. Acceptance contract gate (HARD)

Read `{$META.specs_path}{slug}.acceptance.md`. If absent, refuse — spec was approved without acceptance file (impossible from `/mol:spec`, so manual tampering or partial supersede). User restores from git or re-runs `/mol:spec <slug>`.

Parse `criteria:` from frontmatter. Each carries `status:` per `plugins/mol/rules/evaluator-protocol.md` (`pending` / `verified` / `failed`; default `pending`):

- Every Step 4 RED test MUST trace to ≥1 criterion of `type: code | runtime` (verifiable by tests). State the trace per test.
- `type ∈ {ui_runtime, scientific, performance, docs}` → out of scope for `/mol:impl` verification. Don't skip — surface at Step 7 close-out so user knows which evaluator (`/mol:web` for `ui_runtime`, etc.).
- Criterion with no plausible mapping to any task → stop, tell user the spec/acceptance pair has a gap; re-run `/mol:spec` to refine.

Display the ledger before any new work:

```
acceptance ledger:
  ac-001 (code)        verified   ← skip during Step 5–7 trace
  ac-002 (code)        pending
  ac-003 (runtime)     pending
  ac-004 (ui_runtime)  pending    → /mol:web <slug> ac-004 after close-out
  ac-005 (performance) failed     → /mol:fix or /mol:spec to refine
```

`code`/`runtime` already at `verified` are not re-traced; trust the prior verdict unless 2c resume sync finds the test path missing/red — then drop back to `pending` and re-trace.

### 2b. Read the Tasks section

If missing, refuse and tell user to re-run `/mol:spec` (Tasks section mandatory).

### 2c. Resume sync — reconcile against codebase

A spec's checkboxes drift whenever:

- previous `/mol:impl` was interrupted between change and tick;
- user/another tool implemented a task by hand;
- a `main` refactor subsumed a task.

For every `- [ ]`, **cheap inspection**:

| Task shape | Inspection |
|---|---|
| "Write failing tests for X" / "Add tests covering Y" | grep test names; run `$META.build.test_single`. Pass = done. |
| "Implement X in `<file>`" / "Add `<symbol>` to `<module>`" | `Glob` file, `Grep` symbol. Reachable from non-test caller = done. |
| "Add new `<type/field/route>` `<name>`" | symbol/file existence check. |
| "Wire X into Y" | grep wire-up call site (e.g. `Y.register(X)`). |
| "Update docstrings via documenter" | spot-check 2–3 named symbols against `$META.doc.style`. |
| "Run full check + test suite" | leave unchecked — Step 6 re-runs. |
| Anything not cheaply verifiable | leave unchecked. **Do not guess.** |

Show one-line-per-task verdict before writing:

```
[done — verified]      Implement Foo in src/foo.py        ← will tick
[done — verified]      Add type Bar in src/types.py       ← will tick
[no evidence]          Wire Bar into Service              ← leave
[ambiguous]            Update docs for Baz                ← leave (note why)
```

Wait for confirmation. Replace `- [ ]` with `- [x]` for verified tasks **in one batch** — only place batch ticking is allowed (every other tick is one-at-a-time post-work).

Report concisely: *"`<spec-slug>`: 3 of 7 tasks already done; resuming from 'Wire Bar into Service'."* If 0, *"no out-of-band progress detected; starting from the top."* If **all** tasks already `[x]`, skip to Step 6.

### 2d. Promote status

Set spec frontmatter to `status: in-progress` (no-op if already). Begin Step 3.

If a later step proves the spec wrong (infeasible task, design gap), stop and re-invoke `/mol:spec` for supersede/refine. Resume with the recomputed task list — Step 2c re-runs.

## Step 3 — Architecture check

Read CLAUDE.md section named by `$META.arch.rules_section`. Find the closest existing pattern:

- `layered` — module that imports only from allowed layers.
- `crate-graph` — workspace `Cargo.toml` for target crate's deps.
- `backend-pillars` — backend subtree for a sibling.
- `package-tree` / `monorepo` — target package for a peer.

LARGE → delegate to `architect` agent for full validation before any test.

## Code quality rules (Steps 4–6)

- **Type safety.** Every line — production or test — satisfies `$META.build.check` (`mypy --strict` / `tsc --strict` / `cargo check` / etc.). Forbid escape-hatch top types: `any` / `unknown` (TS), `Any` / untyped sigs (Python), `interface{}` / bare `any` (Go), `dyn Any` (Rust). Replace with explicit union, generic, or trait/protocol bound. Only exception: deserialization at a system boundary — narrow immediately to typed shape.

## Step 4 — TDD (RED)

Find first **Write failing tests …** task. Delegate to `tester` agent:

- Write failing tests at the location `$META.arch.style` dictates.
- Required categories: happy path, edge cases, immutability (if project documents immutable data flow), domain validation (if `$META.science.required`).
- Run `$META.build.test_single` (or `build.test`); confirm tests fail for the right reason.

When tests are written and verified failing, **tick the corresponding box** (`- [ ]` → `- [x]`) immediately. No batching.

## Step 5 — Implement (GREEN), one task at a time

For each remaining unchecked task:

1. Make the change.
2. Run `$META.build.test` (or relevant subset); confirm green.
3. Stay inside the layer from Step 3.
4. **Tick that task's box** before moving on.

Task can't be done as written → stop, re-invoke `/mol:spec` (supersede/refine). Resume with recomputed list.

MEDIUM/LARGE: after impl tasks checked, re-delegate to `architect` for post-impl layer check. Tick the corresponding spec task if present.

## Step 6 — Verify

Run in parallel:

- `$META.build.check` (format/lint)
- `$META.build.test` (full suite)

MEDIUM/LARGE: delegate to `documenter` agent for docstrings in `$META.doc.style`. Tick the docs task when complete.

Tick the **Run full check + test suite** task (or equivalent) once both green.

## Step 6.5 — Hygiene & backward-compat (mandatory)

Invoke `/mol:simplify` on the file list touched in Steps 4–6 (pass explicitly as `$ARGUMENTS` so it doesn't re-derive scope from `git diff`). **Mandatory for every spec, every scope, every stage** — `/mol:simplify` is the single backward-compat gatekeeper per `plugins/mol/rules/stage-policy.md`. It decides:

- legacy newly orphaned by this diff: delete (`experimental`) / shim (`stable`) / migration-note flag (`beta`) / leave (`maintenance`);
- dead imports / debug residue / commented-out code introduced or revealed.

`simplify` runs its own build/test gate and reverts on regression. If revert:

1. Leave spec `status: in-progress`.
2. Do **not** advance to Step 7.
3. Surface suspected trigger and stop. User resolves, re-runs `/mol:impl`; Step 2c recovers.

`simplify` manual handoffs (`/mol:refactor`) or rule-capture suggestions (`/mol:note`) → carry into Step 7 close-out as advisories (non-blocking).

If a "Hygiene cleanup" task exists in the spec, tick it; else add one line to Tasks recording that simplify ran clean (self-documents that backward-compat was reviewed).

## Step 7 — Close out

Re-read spec **and** `{$META.specs_path}{slug}.acceptance.md`.

### 7a. Update acceptance ledger

For every criterion with `type ∈ {code, runtime}`, write back its verdict to `acceptance.md` (the carved-out exception in `plugins/mol/rules/evaluator-protocol.md` § *Field semantics*):

- `status: verified` if traced test path was green in Step 6 (or in 2a-bis if already `verified` and 2c confirmed still green).
- `status: failed` if traced path is red.
- Set `last_checked: <today's ISO date>`.

Do **not** touch criteria with `type ∈ {ui_runtime, scientific, performance, docs}` — those flip only when their runtime evaluator runs. Existing `status` (almost always `pending` on first close-out) stays.

### 7b. Decide next status

Three conditions:

1. Every Tasks item is `[x]`.
2. Latest `$META.build.check` and `$META.build.test` were both green.
3. Every criterion with `type ∈ {code, runtime}` is `status: verified` (no `failed`, no `pending`).

Any failing → leave `status: in-progress` and go to 7e.

All hold → branch on runtime-typed criteria:

- **No runtime-typed, OR all already `verified`** → advance to `status: done`. Run 7c (close-out path).
- **≥1 runtime-typed still `pending` or `failed`** → set `status: code-complete`. **Do not commit via `/mol:commit`; do not delete spec/acceptance/INDEX** — spec is parked. Surface the deferred list (see 7d) and exit cleanly. User invokes each named evaluator; on next `/mol:impl <slug>` (or future `mol:close <slug>`), 2a's `code-complete` branch re-runs 7b and advances to `done` if evaluators flipped everything.

### 7c. Done path

1. Mark spec `status: done` in frontmatter.
2. **Auto-commit on close-out (every spec, every scope).** Invoke `/mol:commit` (which gates on `/mol:ship commit`) before deleting any artifacts. Conventional-commit message:

   ```
   feat(<scope>): <one-line summary> (<slug>)
   ```

   `<scope>` derives from spec's primary layer; summary is the first sentence of spec's Summary, trimmed. Chained sub-specs (`<base>-<NN>-<phase>`) → exactly the per-stage checkpoint required by `plugins/mol/rules/large-spec-split.md` (one commit per sub-spec, no bundling).

   `/mol:commit` BLOCK → **do not delete** spec/acceptance/INDEX; leave `in-progress`, surface failure, stop. User fixes, re-runs `/mol:impl`; Step 2c recovers.

   Never push, never PR — that's `/mol:push` / `/mol:pr`.
3. *"<slug> done — every criterion verified. Deleting spec + acceptance + INDEX entry."*
4. Delete `{$META.specs_path}{slug}.md`.
5. Delete `{$META.specs_path}{slug}.acceptance.md` if present.
6. Remove the entry from `{$META.specs_path}INDEX.md`.
7. If anything from the spec is non-obvious context worth keeping (unusual design, hidden-constraint workaround, deliberate tolerance), suggest user run `/mol:note`. Don't auto-capture.

### 7d. Code-complete path

When 7b decided `code-complete`:

1. Mark spec `status: code-complete` (acceptance.md stays as-is — runtime evaluators mutate the `status:` fields they handle).
2. **Auto-commit the code work.** Invoke `/mol:commit` exactly as the done path (same conventional-commit message). The commit's existence is what makes runtime verification meaningful (evaluators run against committed state, not dirty tree). `/mol:commit` BLOCK → drop to `status: in-progress` and stop.
3. **Surface still-pending runtime criteria**, grouped by `evaluator_hint`:

   ```
   <slug>: code-complete. 3 criteria still pending verification:
     ui_runtime  → run `/mol:web <slug>`
       - ac-004 — first paint of project tree under 200ms
       - ac-005 — error toast appears on save failure
     performance → no evaluator skill registered; verify by hand
       - ac-007 — list endpoint p95 < 50ms

   Re-run `/mol:impl <slug>` after the evaluators have flipped each criterion to `status: verified` to advance the spec to `done`.
   ```

   **Do not auto-invoke.** Orchestration is the user's call (see `plugins/mol/rules/evaluator-protocol.md`).
4. **Do not delete** spec/acceptance/INDEX. Acceptance file MUST stay alive for evaluators.

For chained features, after either close-out path commits, **do not auto-advance** to the next sub-spec. Exit cleanly so user can inspect the stage commit before invoking `/mol:impl <base>-NN+1-<phase>`.

### 7e. Failure branch

If 7b's three conditions don't all hold:

1. Leave `status: in-progress`.
2. Do **not** delete spec or acceptance file.
3. Report unchecked tasks, failing checks, unmet (`pending` / `failed`) criteria. Print the updated ledger from 7a — canonical "where this spec stands."

## Step 8 — Summary

One line: scope, files changed, tests passing, resulting spec `status:` (`done` → deleted; `code-complete` → parked with N runtime criteria pending; `in-progress` → N tasks remaining), and acceptance-ledger tally (e.g. *"3 verified, 1 failed, 2 pending"*).
