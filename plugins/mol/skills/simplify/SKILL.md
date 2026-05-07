---
description: Apply hygiene cleanup to the current diff — opportunistic post-impl simplification. Delegates to the `janitor` agent for findings (read-only), then applies the minimal patch for each accepted finding under a build/test gate. Behavior-preserving by contract; reverts any change that regresses the test suite. Pairs with `/mol:review`'s hygiene axis as the write-mode counterpart.
argument-hint: "[path or list of files]"
---

# /mol:simplify — Apply Hygiene Cleanup

Read CLAUDE.md. Parse `mol_project:` (`$META`).

Write-mode counterpart to the `janitor` review axis: `janitor`
stays read-only and emits hygiene findings; this skill applies
them under a build/test gate with regression revert. See
`plugins/mol/docs/agent-design.md` § "Producer vs reviewer".

## Scope contract

`/mol:simplify` only applies changes that are **provably
behavior-preserving**:

- delete an unused import / unreachable branch / unused local
- delete commented-out code
- delete a debug `print` / `console.log` / `dbg!`
- inline-replace a magic literal with a named constant **already
  defined** in the file (do not introduce new constants without
  user direction)
- rename a local symbol to the project's captured naming rule
  (e.g. `natoms` → `n_atoms`) — **only** when the name is local
  to one file and grep confirms no external caller
- whitespace / import-order fixes the formatter should have
  caught
- delete a stale `TODO` / `FIXME` whose reference is dead code

`/mol:simplify` **refuses** to apply, and surfaces as
"manual fix"  instead:

- copy-paste duplication extraction (cross-file judgment — that's
  `/mol:refactor`)
- function-too-long splits (`/mol:refactor`)
- public-API renames (cross-file caller updates — `/mol:refactor`
  with `pm` agent pre-check)
- any finding labeled `→ defer to <agent>` by `janitor`
- any finding without a captured rule citation (the
  agent-emitted "suggested rule capture" — those go through
  `/mol:note` first, then a future `/mol:simplify` run picks
  them up)

This is the safety contract. If a finding doesn't fit the
"provably behavior-preserving" bar, it is the user's call (or
`/mol:fix` / `/mol:refactor`'s) — not this skill's.

## Procedure

### 1. Determine scope

- If `$ARGUMENTS` is given, treat it as the path / file list.
- Else use `git diff --name-only` for recent uncommitted /
  unpushed changes.
- If the working tree has unrelated uncommitted changes,
  **stop** and tell the user to commit or stash first. This skill
  needs a clean diff to revert cleanly on regression.

### 2. Snapshot the test gate

Before any change:

```
$META.build.check
$META.build.test
```

Record the passing test list and any pre-existing failures. The
revert criterion in Step 5 is "no new failures vs. this
snapshot," not "all tests pass" — pre-existing red tests are
out of scope.

If `$META.build.test` is already failing in unrelated ways, ask
the user whether to proceed with that baseline.

### 3. Delegate to `janitor`

Invoke the `janitor` agent on the scope from Step 1. Capture its
output verbatim — findings + rule-capture suggestions + deferred-
to-other-agent items.

### 4. Triage findings against the scope contract

For each finding the agent returned:

- If it matches the **scope contract** (apply list above) →
  candidate for batch apply.
- If it matches the **refuses list** → label *"manual: route to
  `/mol:refactor` (or `/mol:fix`)"* and skip.
- If `janitor` left a `Fix:` line, propose that exact patch. If
  the fix is multi-line or requires file-level reorganization,
  treat as out of contract and skip.

Show the user the triage table:

```
[apply]    src/foo.py:42  unused import os               → delete line
[apply]    src/foo.py:88  debug print() residue          → delete line
[apply]    src/bar.ts:17  literal "#fff"                 → replace with TOKENS.surface (already in scope)
[manual]   src/baz.py:12  function 142 lines             → /mol:refactor (split)
[manual]   src/baz.py:55  copy-paste of foo.py:120-140   → /mol:refactor (extract)
[skipped]  src/qux.py:9   naming drift (no captured rule)→ /mol:note first, then re-run
```

Wait for explicit user approval. The user may de-select any
`[apply]` row.

### 5. Apply, verify, revert on regression

For each approved row:

1. Apply the minimal patch (a single Edit per finding when
   possible).
2. After the **whole batch** is applied, run
   `$META.build.test`.
3. If any test that was green in the Step 2 snapshot is now red,
   **revert the entire batch** (`git checkout -- <files>`) and
   tell the user which finding was the suspected trigger
   (re-applying findings one-by-one to bisect is the user's
   call, not automatic).
4. If everything stays green, run `$META.build.check` to confirm
   format / lint also pass.

Never partial-apply. The contract is "this batch was
behavior-preserving" — a green-after-revert state is the only
acceptable failure mode.

### 6. Report

```
/mol:simplify: applied N hygiene fixes across K files
  - N_unused imports / debug residue / commented code
  - N_naming drift fixes (captured rules)
  - N_token-or-constant substitutions

Manual handoffs (M):
  - <finding> → /mol:refactor / /mol:fix
Suggested rule captures (S):
  - <suggestion> → /mol:note
```

If `janitor` emitted rule-capture suggestions, surface them in
their own section. Don't auto-route — the user decides which
become rules.

End with a one-line summary: files touched, fixes applied,
manual handoffs queued, tests still green.

## Guardrails

- **Behavior-preserving only.** Anything that could change
  runtime behavior, public API, or test outcomes belongs in
  `/mol:fix` or `/mol:refactor`.
- **Whole-batch atomicity.** A test regression reverts the
  entire batch. Never leave the tree in a half-cleaned state.
- **No new abstractions.** Do not introduce new helpers,
  constants, or modules. Only delete or rename.
- **No CLAUDE.md / `.claude/notes/` writes.** Rule capture is
  `/mol:note`'s job.

## Idempotency

Running `/mol:simplify` twice in a row on the same scope: the
second run finds zero `[apply]` candidates (everything from the
first run is gone) and either reports zero changes or surfaces
manual / rule-capture handoffs that the first run also flagged.

## When to invoke

Explicit only — the user-side rule "a bug fix doesn't need
surrounding cleanup" makes cleanup a separate operation.
Reasonable triggers: after `/mol:impl` finishes (before
`/mol:commit`), or after `/mol:review` flagged hygiene findings.
