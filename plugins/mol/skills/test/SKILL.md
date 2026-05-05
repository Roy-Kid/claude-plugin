---
description: Run the project's test suite and analyze test quality, coverage gaps, and missing categories.
argument-hint: "[module or test path]"
---

# /mol:test — Test & Coverage Analysis

Read CLAUDE.md. Parse `mol_project:` (`$META`).

## Procedure

1. **Run the suite.**
   - If `$ARGUMENTS` is given, prefer `$META.build.test_single` with
     `{path}` substituted. Otherwise run `$META.build.test`.
   - Capture pass / fail / skip counts and the list of failing tests.

2. **Delegate to the `tester` agent** for test-quality analysis:
   - Are the documented test categories present (happy path, edge cases,
     immutability if the project documents immutable flow, domain
     validation if `$META.science.required`)?
   - Are tests deterministic (seeded, no time-of-day dependency)?
   - Do assertions use appropriate tolerances for floating-point?

3. **Check coverage.** If `$META.build.coverage` is defined, run it and
   summarize by module. Flag modules below the project's documented
   target (if CLAUDE.md states one; otherwise 80%).

## Output

- Pass / fail / skip counts
- Per-module coverage (if coverage step ran)
- Missing test categories (from the `tester` agent)
- Failing tests grouped by likely root cause (classification only —
  fixing is `/mol:fix`)

End with a one-line summary.
