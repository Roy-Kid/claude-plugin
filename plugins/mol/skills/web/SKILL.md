---
description: Frontend runtime evaluator. Reads `<slug>.acceptance.md` from `.claude/specs/`, filters criteria of `type: ui_runtime`, optionally auto-starts the dev server via `mol_project.dev.command`, then verifies each criterion by driving the running web app through whatever Playwright MCP server / browser-automation plugin the user has installed. Returns a verdict per the evaluator-protocol contract. Read-only on source code; writes only artifact files (screenshots, console dumps) under `.claude/specs/<slug>.artifacts/`. Skip cleanly when no Playwright tools are available.
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

### 3. Resolve the target URL

Read the `mol_project.dev` block from CLAUDE.md (defined below
under § Project configuration). Pick the target URL in this order:

1. `mol_project.dev.url` if set.
2. A URL named under a "Dev environment" section of the spec.

If neither is present, stop and tell the user: the project has
not declared its dev URL — they should add `mol_project.dev.url`
to CLAUDE.md (see § Project configuration) or include a
"Dev environment" section in the spec.

Show the user the working set (id + summary + pass_when, one line
each) plus the resolved target URL. Then probe with the
browser-automation MCP `navigate` tool, 5 s timeout. If the page
loads, jump to Step 4.

### 3a. Auto-start the dev server (only if the probe failed)

If `mol_project.dev.command` is set, the skill brings the dev
server up itself instead of stopping at the probe:

1. Run the command in the background, capturing combined
   stdout/stderr to a log file under `/tmp/mol-web-<slug>.log`.
   Record the PID.
2. Tail the log every 1 s; consider the server ready as soon as
   the configured `ready_pattern` appears.
3. Parse the actual URL from the ready line using the configured
   `url_pattern` regex — many dev servers fall back to a
   different port when the configured one is busy (e.g.
   "port 3000 in use, using port 3001"), so the printed URL is
   the source of truth, not `mol_project.dev.url`.
4. Re-run the `navigate` probe against the parsed URL (5 s
   timeout). On success, continue to Step 4 and remember the PID
   for Step 6 cleanup. On failure or timeout (`ready_timeout`),
   kill the PID, stop, and tell the user the auto-start command
   did not produce a reachable URL.

If `mol_project.dev.command` is **not** set and the Step 3 probe
fails, stop and tell the user to either start the app manually or
declare `mol_project.dev.command` in CLAUDE.md. Do not invent a
command — CLAUDE.md is the source of truth for project-specific
HOW.

## Project configuration

This skill is project-agnostic; everything project-specific lives
in CLAUDE.md under `mol_project.dev`. None of the keys are
mandatory, but a project that wants the auto-start path must
declare at least `command` + `ready_pattern` + `url_pattern`.

```yaml
mol_project:
  dev:
    # required for Step 3 if no "Dev environment" section in the spec
    url: <project-specific dev URL>

    # required for Step 3a auto-start; the skill never invents commands
    command: <project-specific shell command that boots the dev server>

    # log line substring that means "the server is now listening"
    ready_pattern: <project-specific marker, e.g. a server's banner string>

    # regex with one capture group that extracts the actual URL from
    # the ready line (a dev server may pick a different port than url)
    url_pattern: <project-specific regex>

    # seconds to wait for ready_pattern before giving up (default: 90)
    ready_timeout: 90
```

Why per-project: dev-server stacks, workspace layouts (single-
package vs monorepo subdirectory), banner strings, and the choice
of mock vs live backend all vary across projects. The skill
states the procedural intent ("auto-start the dev server, navigate
to it, evaluate, clean up"); each project spells out its own HOW.

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

### 6. Cleanup

If Step 3a started a dev server in this run, stop it now:

```
kill <PID>
```

Do not kill a server you didn't start — if Step 3 found one
already reachable, it belongs to the user (or another tool) and
must be left running.

Always run cleanup, even on a skip / fail verdict, so a
half-finished evaluation does not leave a zombie dev server.

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
