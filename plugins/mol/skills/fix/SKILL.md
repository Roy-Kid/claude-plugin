---
description: Minimal-diff bug fix — reproduce, diagnose, patch the smallest surface, verify. Writes code.
argument-hint: "<bug description, error message, or failing test>"
---

# /mol:fix — Quick Fix

Read CLAUDE.md. Parse `mol_project:` (`$META`).

## Procedure

1. **Reproduce.** Run `$META.build.check` and `$META.build.test` (or the
   single-test form if the failure is a specific test). Confirm the
   reported symptom.

2. **Diagnose.** Follow the `/mol:debug` procedure inline — classify the
   failure (build / test / runtime), gather evidence, root-cause. Stop once
   the root cause is identified.

3. **Fix.** Make the minimal change that resolves the issue:
   - If the fix touches architecture boundaries, consult
     `$META.arch.rules_section` in CLAUDE.md first.
   - If the root cause suggests a missing test, delegate to the `tester`
     agent to write a regression test BEFORE the fix (RED), then fix
     (GREEN).

4. **Verify.** Run the full `$META.build.test` suite to confirm no
   regressions. Run `$META.build.check` for format / lint.

5. **Report.** One-line summary: root cause, files changed, test added
   (if any).
