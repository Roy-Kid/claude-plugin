---
description: Restructure code while preserving all architectural invariants. Snapshot → incremental change → re-verify. Writes code.
argument-hint: "<what to refactor and why>"
---

# /mol:refactor — Refactor

Read CLAUDE.md. Parse `mol_project:` (`$META`).

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

5. **Verify invariants.**
   - Same tests pass (no regressions).
   - Public API surface matches the snapshot unless the user accepted an
     API change in step 2.
   - Delegate to the `architect` agent again to confirm post-refactor
     compliance.

End with a one-line summary: files moved / renamed, tests passing, API
changes (if any).
