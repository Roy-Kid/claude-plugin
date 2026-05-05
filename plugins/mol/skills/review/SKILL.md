---
description: Multi-axis static code review — aggregates architecture, performance, documentation, user-experience, product (public-API / breaking-change), hygiene (continuous tech-debt servicing via the janitor agent), and (when applicable per-file detection) visual design, security, scientific-correctness, and numerical-stability findings into a severity table with verdict. **Static only**; runtime evaluation (Playwright / benchmarks / numerical oracles) is suggested via `acceptance.md` `type` fields and dispatched manually to runtime-evaluator skills like `/mol:web`, per `plugins/mol/docs/evaluator-protocol.md`. Read-only.
argument-hint: "[path or list of modified files] | [<spec-slug>]"
---

# /mol:review — Multi-Axis Code Review

Read CLAUDE.md. Parse `mol_project:` (`$META`).

Determine the files to review: use `$ARGUMENTS` if given; otherwise
`git diff --name-only` for recent uncommitted / unpushed changes.

If `$ARGUMENTS` looks like a spec slug (no slash, no dot, matches a
file under `$META.specs_path`), additionally read
`{$META.specs_path}{slug}.acceptance.md` and use its criteria of
`type: code` to focus the static reviewers.

## Delegate in parallel (single message, multiple tool calls)

1. **Architecture** — delegate to the `architect` agent.
2. **Performance** — delegate to the `optimizer` agent.
3. **Documentation** — delegate to the `documenter` agent (Mode A: audit).
4. **User experience** — delegate to the `undergrad` agent (reviews
   user-facing API, onboarding docs, extension story from a new user's
   perspective).
5. **Product / public-API** — delegate to the `pm` agent (reviews
   public-surface naming, breaking-change risk, downstream integration
   contracts).
6. **Scientific correctness** — delegate to the `scientist` agent
   **only if** `$META.science.required` is `true`.
7. **Numerical stability / HPC readiness** — delegate to the
   `compute-scientist` agent **only if** `$META.science.required` is
   `true`. The agent additionally self-detects HPC relevance per
   file (CUDA / PyTorch DDP / xsimd signals) and silently skips
   files where neither numerics nor HPC apply.
8. **Visual design** — delegate to the `web-design` agent. Always
   delegated; the agent self-skips per file when no frontend code
   is present. Covers design-token consistency, info density,
   empty/error/loading states, accessibility, responsive behavior.
9. **Security** — delegate to the `security-reviewer` agent.
   Always delegated; the agent self-skips per file when no attack
   surface is present (no subprocess + user input, no web handler,
   no LLM tool dispatch, no path/network operations on untrusted
   data, etc.). Covers shell / SQL / path / SSRF / prompt
   injection, deserialization hazards, secret leakage, missing
   authorization on mutating endpoints.
10. **Hygiene / continuous tech-debt** — delegate to the `janitor`
    agent. Always runs. Reads `.agent/` for captured aesthetic
    rules and applies them to the diff; emits hygiene findings
    plus suggestions for new rules to capture via `/mol:note`.

Each agent returns its findings in `<emoji> file:line — message` form,
where emoji is 🚨 Critical / 🔴 High / 🟡 Medium / 🟢 Low.

## Aggregate

Emit a summary table:

| Dimension          | 🚨 | 🔴 | 🟡 | 🟢 |
|--------------------|----|----|----|----|
| Architecture       |    |    |    |    |
| Performance        |    |    |    |    |
| Documentation      |    |    |    |    |
| User experience    |    |    |    |    |
| Product / API      |    |    |    |    |
| Science            |    |    |    |    |
| Numerics / HPC     |    |    |    |    |
| Visual design      |    |    |    |    |
| Security           |    |    |    |    |
| Hygiene (janitor)  |    |    |    |    |

Below the table, list every 🚨 and 🔴 finding verbatim from the agents,
grouped by dimension.

If `janitor` returned any **rule-capture suggestions** (patterns it
saw without a captured `.agent/` rule), surface them in their own
section before the verdict so the user can lock them in via
`/mol:note`.

## Runtime evaluator handoff

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

## Verdict

The static verdict is independent of runtime criteria:

- **APPROVE** — zero 🚨, zero 🔴.
- **REQUEST CHANGES** — zero 🚨, some 🔴.
- **BLOCK** — any 🚨.

If runtime evaluators are pending, append a note to the verdict
line: *"static APPROVE; 3 runtime criteria still unverified."*

End with a one-line summary: files reviewed, verdict, count of
findings by severity, count of pending runtime criteria (if any).
