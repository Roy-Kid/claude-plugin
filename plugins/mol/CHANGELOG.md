# Changelog

## 0.1.1 — 2026-05-05

Aligns `mol` with the planner / generator / evaluator harness
pattern from
[Anthropic's harness-design guide](https://www.anthropic.com/engineering/harness-design-long-running-apps).
Spec quality, the binding "done" contract, and runtime UI
evaluation are first-class steps in the workflow.

### Features

- **`/mol:spec` now produces both `<slug>.md` and
  `<slug>.acceptance.md` in one pass.** Spec validation
  (sections / atomic tasks / RED-before-GREEN / Files↔Tasks
  cross-reference) is an internal quality gate that runs before
  the user sees the draft. Acceptance negotiation walks the spec
  section-by-section and produces structured observable criteria
  with `id` / `summary` / `type` / `pass_when`. The user signs off
  on both files together; `/mol:impl` refuses without both.
- **New `/mol:web` skill.** Frontend runtime evaluator. Reads
  `<slug>.acceptance.md`, filters `type: ui_runtime` criteria, and
  verifies each by driving whatever browser-automation MCP server
  the user has installed (Playwright MCP, claude-in-chrome, …).
  Self-skips cleanly when no Playwright MCP is reachable. Returns
  a verdict in the evaluator-protocol shape (pass / fail / skip
  per criterion + artifact paths under
  `.claude/specs/<slug>.artifacts/`). `mol` does not bundle a
  Playwright distribution — install whichever MCP you prefer.
- **New `playwright-evaluator` agent.** Per-criterion sub-agent
  invoked by `/mol:web`. Translates one `pass_when` into
  navigate / interact / observe / assert sequences. Captures
  evidence proportional to verdict (one screenshot on pass;
  before/after + console + network on fail). Never decides
  whether to retry — the orchestrator does.
- **New evaluator-protocol document
  (`plugins/mol/docs/evaluator-protocol.md`).** Specifies the
  three artifact paths (`<slug>.md` / `<slug>.acceptance.md` /
  `<slug>.artifacts/`), the `acceptance.md` schema (yaml
  frontmatter + markdown body, structured for external
  orchestrators), the `criteria[].type` taxonomy
  (`code` / `runtime` / `ui_runtime` / `scientific` /
  `performance` / `docs`), and the contract a runtime evaluator
  skill must satisfy.

### Refactors

- **`/mol:spec` supersede/refine flow regenerates `acceptance.md`
  from scratch.** When the conflict-check path updates an existing
  spec body, the previously negotiated criteria are dropped and a
  fresh `acceptance.md` is produced against the new Tasks list
  (criteria ids restart at `ac-001` because the spec itself is a
  new design).
- **`/mol:impl` adds two HARD gates at Step 2.** Refuses to start
  unless `status: approved` AND `<slug>.acceptance.md` exists.
  Step 4 RED tests must trace back to a `code`/`runtime`
  criterion; criteria of other types are surfaced at Step 7
  close-out as runtime-evaluator handoffs (no auto-invocation).
- **`/mol:impl` Step 7 cleanup.** On completion, deletes the spec
  file, the acceptance file, and the INDEX entry together.
  Reports deferred runtime criteria grouped by `evaluator_hint`
  so the user knows which runtime evaluator to invoke next.
- **`/mol:review` is now static-only.** Adds a "Runtime
  evaluators (manual)" section that maps `ui_runtime` /
  `scientific` / `performance` / `runtime` criteria to suggested
  evaluator-skill invocations (e.g. `/mol:web <slug> ac-004`).
  Never auto-dispatches — orchestration is the user's call (or
  their external orchestrator's).

### Lifecycle

The spec status flow is simpler than it would have been with a
separate spec-check skill:

```
draft       (user deferred approval mid-/mol:spec)
approved    (signed off both <slug>.md and <slug>.acceptance.md)
in-progress (/mol:impl running)
done        (/mol:impl cleaned up; spec + acceptance deleted)
```

### Docs

- README updated: 14 skills, 14 agents, new `evaluator-protocol.md`
  reference, `/mol:web` mention.

## 0.1.0

- Initial release. 13 skills (impl, spec, litrev, arch, review,
  test, perf, docs, note, fix, debug, refactor, ship) + 13
  single-axis agents (architect, tester, scientist,
  compute-scientist, optimizer, documenter, undergrad, pm,
  ci-guard, web-design, security-reviewer, janitor, reviewer).
  Four-zone layering (docs / .agent / .claude / CLAUDE.md) with
  active-vs-passive split. Adapts per project via `mol_project`
  frontmatter.
