---
description: Diagnose build, test, or runtime failures with structured root-cause analysis. Read-only — never writes code. Delegates to the `debugger` agent. Use /mol:fix to actually patch.
argument-hint: "<error message or symptom>"
---

# /mol:debug — Diagnosis Only

Read CLAUDE.md. Parse `mol_project:` (`$META`).

This skill **never** edits files. It produces a diagnosis only.
To patch, hand the report off to `/mol:fix`.

## Procedure

Delegate the entire diagnosis to the `debugger` agent with
`$ARGUMENTS` as the failure symptom. The agent classifies the
failure (build / test / runtime), runs `$META.build.*` to gather
evidence, and returns:

- **Classification.**
- **Root cause** — one paragraph, file:line precise.
- **Fix recommendation** — what to change, not the change.
- **Preventive measure** — the test category + assertion shape
  that would catch a regression.
- (when evidence is inconclusive) **Open questions** — specific
  evidence the user should gather before re-invoking.

Render the agent's output verbatim. Do not re-summarize — the
agent's structured report is the contract `/mol:fix` reads.

End with the one-liner: *"diagnosis only — invoke `/mol:fix
<bug>` to apply the recommendation."*
