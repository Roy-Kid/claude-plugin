---
description: Frontend runtime evaluator. Reads `<slug>.acceptance.md` from `.claude/specs/`, filters criteria of `type: ui_runtime`, and verifies each by driving the running web app via whatever Playwright MCP server / browser automation plugin the user has installed. Returns a verdict per the evaluator-protocol contract. Read-only on source code; writes only artifact files (screenshots, console dumps) under `.claude/specs/<slug>.artifacts/`. Skip cleanly when no Playwright tools are available.
argument-hint: "<spec-slug> [<criterion-id>]"
---

# /mol:web — Frontend Runtime Evaluator

Verify `ui_runtime` acceptance criteria against a running
application. This is the runtime half of the harness — `/mol:review`
covers the static half — so projects that ship a UI can close the
loop end-to-end.

## Detect (run only when applicable)

This skill is meaningful only when **both** of the following hold:

- The spec has at least one `ui_runtime` criterion in
  `<slug>.acceptance.md`.
- A Playwright-capable MCP server is reachable in this session.
  Look for any of these tool prefixes: `mcp__plugin_playwright_*`,
  `mcp__claude-in-chrome__*`, or any other browser-automation MCP
  the user has installed. **`mol` does not bundle one** — the user
  installs whichever Playwright MCP / plugin they prefer.

If no `ui_runtime` criteria exist, return *"no `ui_runtime`
criteria for `<slug>` — nothing to do"* and stop.

If `ui_runtime` criteria exist but no Playwright MCP is available,
return *"no browser-automation MCP installed; install one (e.g.
the official Playwright MCP) and re-run, or verify these
criteria manually"* — list the criteria — and stop. Do not try to
shell out to `npx playwright`; this skill operates strictly through
MCP.

## Procedure

### 1. Resolve the slug

`$ARGUMENTS` parses as `<slug> [<criterion-id>]`. Read CLAUDE.md
to resolve `$META.specs_path` (default `.claude/specs/`).

Read:

- `{$META.specs_path}{slug}.md` — the spec.
- `{$META.specs_path}{slug}.acceptance.md` — the contract.

If either is missing, stop. Tell the user the spec is not approved
yet — they should run `/mol:spec <slug>`.

### 2. Filter criteria

Parse `criteria:` from the acceptance frontmatter. Build the
working set:

- if `<criterion-id>` given: only that one (refuse if its `type`
  is not `ui_runtime`; explain that this skill handles only that
  type).
- otherwise: every criterion with `type: ui_runtime`.

If the working set is empty, see § Detect above.

### 3. Confirm the target environment

Show the user:

- the working set (id + summary + pass_when, one line each);
- a question: *"Which URL is the running app at? (e.g.
  http://localhost:5173)"*. Default to a value declared in the
  spec under a section titled "Dev environment" or similar if
  present.

Wait for confirmation. Cheap check: invoke the Playwright MCP
`navigate` tool against the URL with a 5s timeout to verify it
loads. If it does not, stop and ask the user to start the app.

### 4. Delegate per criterion

For each criterion in the working set, delegate to the
`playwright-evaluator` agent with the criterion verbatim plus the
target URL. The agent translates `pass_when` into a navigate /
interact / observe / assert sequence and returns:

- a verdict (`✅ pass` / `❌ fail` / `⏭ skip`)
- one-line evidence
- artifact paths (screenshots, console dump) under
  `.claude/specs/<slug>.artifacts/<criterion-id>/`

Run the per-criterion delegations **sequentially** (not in
parallel) — Playwright sessions interfere when run concurrently
against the same target.

### 5. Aggregate and emit

Print the verdict in the evaluator-protocol shape:

```markdown
## /mol:web — <slug>

| Criterion | Verdict | Evidence |
|-----------|---------|----------|
| ac-004    | ✅ pass | first paint at 142ms (target <200ms) → screenshot |
| ac-005    | ❌ fail | error toast did not appear after save 500 → screenshot + console |

### Artifacts

- `.claude/specs/<slug>.artifacts/ac-004/state.png`
- `.claude/specs/<slug>.artifacts/ac-005/before.png`
- `.claude/specs/<slug>.artifacts/ac-005/after.png`
- `.claude/specs/<slug>.artifacts/ac-005/console.log`
```

End with a one-line user-facing summary: *"<slug>: 4 ui_runtime
criteria, 3 pass, 1 fail. See artifacts under
.claude/specs/<slug>.artifacts/."*

## Guardrails

- **Read-only on source.** This skill never edits code. If a
  verdict is `fail`, it is the user's (or orchestrator's) job to
  decide whether to feed the failure back into `/mol:fix` or
  `/mol:spec` (refine the criterion).
- **Artifacts under specs/, not docs/.** Artifacts are scratch —
  delete them when the spec is closed (`/mol:impl` Step 7 deletes
  the spec; deleting `.artifacts/` is a manual cleanup the user
  can run, since impl does not know about runtime artifacts by
  design).
- **Do not auto-loop.** If a criterion fails, do not re-run after
  a code change. The harness pattern explicitly leaves the
  loop-back decision to the orchestrator.
- **Browser dialog hazard.** Per the Playwright MCP guidance,
  avoid clicking elements that trigger native `alert/confirm/prompt`
  dialogs without first acknowledging the user — they block the
  session and break the next criterion.

## Why this lives in `mol`

The runtime evaluator was originally drafted as a separate
`mol-web` plugin, but a separate plugin only makes sense when a
plugin owns a non-trivial dependency. `mol:web` does not bundle
Playwright — it consumes whatever browser-automation MCP the user
has installed — so its only "asset" is the SKILL.md procedure and
the `playwright-evaluator` agent. That fits cleanly inside `mol`
as a sibling to `/mol:review`, behind a self-detect gate.

The same shape can host other runtime evaluators when a project
needs them: `/mol:bench` for `performance`, `/mol:numeric` for
`scientific`, etc. Each follows the contract in
`plugins/mol/docs/evaluator-protocol.md` and self-skips when its
prerequisites are not present.
