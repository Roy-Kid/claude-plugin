---
description: Multi-axis static code review — fans out to single-axis review agents and aggregates findings into one verdict. Use after writing or modifying code; static only — runtime evaluation is dispatched separately via `/mol:web` and other evaluator skills per `acceptance.md` `type` fields.
argument-hint: "[<path-or-spec-slug>] [--axis=<name>]"
---

# /mol:review — Multi-Axis Code Review

Read CLAUDE.md → parse `mol_project:` (`$META`).

## Step 1 — Parse arguments

`$ARGUMENTS` may contain:

- Path / list of modified files; else default to `git diff --name-only`.
- Spec slug (no slash, no dot, matches a file under `$META.specs_path`). Present → also read `{$META.specs_path}{slug}.acceptance.md` and use its `type: code` criteria to focus reviewers.
- `--axis=<name>` — scope to one dimension:

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

  No `--axis` → working set is **all axes** (gating still applies).

`--axis=<name>` whose gate fails (e.g. `--axis=science` with `science.required: false`) → refuse and explain.

## Step 2 — Fan out to working set (single message, parallel)

Delegate in parallel. Full set (no `--axis`):

1. **Architecture** — `architect`.
2. **Performance** — `optimizer`.
3. **Documentation** — `documenter` (Mode A: audit).
4. **User experience** — `undergrad` (user-facing API, onboarding, extension story).
5. **Product / public-API** — `pm` (public-surface naming, breaking-change risk, downstream contracts).
6. **Scientific correctness** — `scientist`, **only if** `$META.science.required: true`.
7. **Numerical stability / HPC readiness** — `compute-scientist`, **only if** `$META.science.required: true`. Self-detects HPC relevance per file; silently skips.
8. **Visual design** — `web-design`. Always delegated; self-skips per file with no frontend.
9. **Security** — `security-reviewer`. Always delegated; self-skips per file with no attack surface.
10. **Hygiene** — `janitor`. Always runs. Reads `.claude/notes/` for captured aesthetic rules; emits hygiene findings + suggestions for new rules via `/mol:note`.

Findings format: `<emoji> file:line — message` (🚨 Critical / 🔴 High / 🟡 Medium / 🟢 Low).

## Step 3 — Aggregate via `reviewer` agent

Collect every agent's raw findings verbatim (including "N/A for this file" passes from `web-design` / `security-reviewer`, and rule-capture suggestions from `janitor`). Pass to `reviewer` with dimensions covered, file scope, and any acceptance-slug. `reviewer` handles:

- deduplication by `(file, line, category)`
- conflict resolution between contradicting agents
- severity table
- listing 🚨 / 🔴 findings verbatim
- verdict line (APPROVE / REQUEST CHANGES / BLOCK)
- rule-capture suggestions section (when `janitor` returned any)

`/mol:review` does **not** re-aggregate — renders exactly what `reviewer` returns.

## Step 4 — Runtime evaluator handoff (skill-level)

Static-only. If `acceptance.md` is in scope (slug passed, or found alongside diff-referenced spec), inspect its `criteria:`:

- For every criterion with `type ∈ {ui_runtime, scientific, performance, runtime}` **whose `status` is `pending` or `failed`**, emit a *suggestion line* — never an invocation. Skip `verified` (evaluator already ran).
- Use `evaluator_hint` if set; else look up the type in **Known evaluator plugins** in `plugins/mol/rules/evaluator-protocol.md`.
- No registered plugin for that type → say so plainly: *"ac-007 (performance): no evaluator plugin registered; verify manually."*

Section format:

```
## Runtime evaluators (manual)

- ac-004 (ui_runtime) → `/mol:web <slug> ac-004`
- ac-005 (ui_runtime) → `/mol:web <slug> ac-005`
- ac-007 (performance) → no evaluator skill; verify by hand
```

User decides whether/when to run any. `/mol:review` does not auto-dispatch.

Pending runtime evaluators → append note to verdict: *"static APPROVE; 3 runtime criteria still unverified."*

## Step 5 — One-line summary

Files reviewed, axes covered, verdict, count of findings by severity, count of pending runtime criteria (if any).
