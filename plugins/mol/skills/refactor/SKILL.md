---
description: Restructure code while preserving all architectural invariants. Snapshot → incremental change → re-verify. Writes code.
argument-hint: "<what to refactor and why>"
---

# /mol:refactor — Refactor

Read CLAUDE.md. Parse `mol_project:` (`$META`). Read `$META.stage`
(default: `experimental`). Print `[mol] stage: <value>`.

Stage gate per `plugins/mol/rules/stage-policy.md` — apply before
any other step:

- `maintenance` — **refuse**. Print: *"`<project>` is in
  `maintenance`; refactors are out of scope. Bump the stage in
  CLAUDE.md if this is intentional."* and stop.
- `stable` — proceed, but public symbol renames require `pm` agent
  pre-review (delegate at Step 3 alongside `architect`). Internal
  renames are unaffected.
- `beta` — proceed; the post-refactor `architect` check at Step 5
  additionally verifies that README / tutorial code blocks still
  parse against the new public shape.
- `experimental` — proceed with full latitude; the snapshot in
  Step 1 records the *current* shape, and the public-API
  preservation clause in Step 5 is treated as informational rather
  than binding.

## Procedure

1. **Snapshot invariants.** Before any change:
   ```
   $META.build.check
   $META.build.test
   ```
   Record the passing test list and the public API surface you are about to
   restructure (function signatures, exported symbols, module layout). This
   snapshot is what the refactor must preserve.

2. **Understand scope.** What moves where / gets renamed / merges /
   splits? What interfaces change? What must stay the same?

3. **Architecture pre-check.** Delegate to the `architect` agent on the
   current state of the target area. The agent confirms the current
   structure is understood and the target structure satisfies
   `$META.arch.rules_section`.

4. **Execute incrementally.** One logical change at a time. After each
   change, run `$META.build.test`. If a change breaks more than the set
   of tests you expected, stop and investigate before continuing.
   **Type safety.** A refactor may only tighten types, never loosen
   them — it must not introduce `any` / `Any` / `interface{}` /
   `dyn Any` and must not strip existing annotations. Treat any
   surfaced loose type as a refactor opportunity to narrow it. The
   project's static type checker must remain green at every step.

5. **Verify invariants.**
   - Same tests pass (no regressions).
   - Public API surface matches the snapshot unless the user accepted an
     API change in step 2.
   - Delegate to the `architect` agent again to confirm post-refactor
     compliance.

End with a one-line summary: files moved / renamed, tests passing, API
changes (if any).
