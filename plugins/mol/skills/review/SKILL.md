---
description: Multi-axis static code review — fans out to single-axis review agents, collects their findings, hands them to the `reviewer` agent for aggregation + verdict. **Static only**; runtime evaluation (Playwright / benchmarks / numerical oracles) is suggested via `acceptance.md` `type` fields and dispatched manually to runtime-evaluator skills like `/mol:web`, per `plugins/mol/docs/evaluator-protocol.md`. Read-only. Supports `--axis=<name>` to scope to a single review dimension.
argument-hint: "[<path-or-spec-slug>] [--axis=<name>]"
---

# /mol:review — Multi-Axis Code Review

Read CLAUDE.md. Parse `mol_project:` (`$META`).

## Step 1 — Parse arguments

`$ARGUMENTS` may contain any combination of:

- A path / list of modified files. Otherwise default to
  `git diff --name-only` for recent uncommitted / unpushed changes.
- A spec slug (no slash, no dot, matches a file under
  `$META.specs_path`). When present, additionally read
  `{$META.specs_path}{slug}.acceptance.md` and use its
  `type: code` criteria to focus the static reviewers.
- `--axis=<name>` — scope the review to a single dimension.
  Recognized values:

  | Axis flag | Agent invoked |
  |---|---|
  | `arch` | `architect` |
  | `perf` | `optimizer` |
  | `docs` | `documenter` (Mode A: audit) |
  | `ux` | `undergrad` |
  | `api` | `pm` |
  | `science` | `scientist` (gated on `$META.science.required`) |
  | `numerics` | `compute-scientist` (gated on `$META.science.required`) |
  | `visual` | `web-design` |
  | `security` | `security-reviewer` |
  | `hygiene` | `janitor` |

  If `--axis` is absent, the working set is **all axes** (the gating
  rules below still apply).

If `--axis=<name>` names an axis whose gate is not satisfied (e.g.
`--axis=science` on a project with `science.required: false`), refuse
and explain.

## Step 2 — Fan out to the working set (single message, parallel)

Delegate in parallel to every agent in the working set. The full set
(when `--axis` is absent) is:

1. **Architecture** — `architect` agent.
2. **Performance** — `optimizer` agent.
3. **Documentation** — `documenter` agent (Mode A: audit).
4. **User experience** — `undergrad` agent (user-facing API,
   onboarding, extension story).
5. **Product / public-API** — `pm` agent (public-surface naming,
   breaking-change risk, downstream integration contracts).
6. **Scientific correctness** — `scientist` agent **only if**
   `$META.science.required` is `true`.
7. **Numerical stability / HPC readiness** — `compute-scientist`
   agent **only if** `$META.science.required` is `true`. The agent
   self-detects HPC relevance per file and silently skips files
   where neither numerics nor HPC apply.
8. **Visual design** — `web-design` agent. Always delegated; the
   agent self-skips per file when no frontend code is present.
9. **Security** — `security-reviewer` agent. Always delegated; the
   agent self-skips per file when no attack surface is present.
10. **Hygiene / continuous tech-debt** — `janitor` agent. Always
    runs. Reads `.claude/notes/` for captured aesthetic rules and applies
    them to the diff; emits hygiene findings plus suggestions for
    new rules to capture via `/mol:note`.

Each agent returns its findings in `<emoji> file:line — message`
form (🚨 Critical / 🔴 High / 🟡 Medium / 🟢 Low).

## Step 3 — Aggregate via the `reviewer` agent

Collect every agent's raw findings (verbatim, including any
"N/A for this file" passes from `web-design` / `security-reviewer`,
and any rule-capture suggestions from `janitor`). Pass the
collected block to the `reviewer` agent with the dimensions
covered, the file scope, and any acceptance-criteria slug used to
focus the reviewers. The `reviewer` agent is responsible for:

- deduplication by `(file, line, category)`,
- conflict resolution between contradicting agents,
- the severity table,
- listing 🚨 / 🔴 findings verbatim,
- the verdict line (APPROVE / REQUEST CHANGES / BLOCK),
- the rule-capture suggestions section (when `janitor` returned any).

`/mol:review` does **not** re-aggregate the output — it renders
exactly what the `reviewer` agent returns.

## Step 4 — Runtime evaluator handoff (skill-level)

This skill is **static-only**. If an `acceptance.md` is in scope
(either passed explicitly as a slug, or found alongside a spec
referenced in the diff), inspect its `criteria:` list:

- For every criterion with `type ∈ {ui_runtime, scientific,
  performance, runtime}`, emit a *suggestion line* — never an
  invocation.
- Use the `evaluator_hint` field if set; otherwise look up the type
  in the **Known evaluator plugins** table in
  `plugins/mol/docs/evaluator-protocol.md`.
- If no plugin is registered for that type, say so plainly:
  *"ac-007 (performance): no evaluator plugin registered; verify
  manually."*

Format the section as:

```
## Runtime evaluators (manual)

- ac-004 (ui_runtime) → `/mol:web <slug> ac-004`
- ac-005 (ui_runtime) → `/mol:web <slug> ac-005`
- ac-007 (performance) → no evaluator skill; verify by hand
```

The user — or their external orchestrator — decides whether and
when to run any of these. `/mol:review` does not auto-dispatch.

If runtime evaluators are pending, append a note to the verdict
line: *"static APPROVE; 3 runtime criteria still unverified."*

## Step 5 — One-line summary

End with: files reviewed, axes covered, verdict, count of findings
by severity, count of pending runtime criteria (if any).
