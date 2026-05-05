# mol-agent

Per-project agent-harness lifecycle skills, paired with the `mol`
plugin.

`mol-agent` installs and maintains the *scaffolding* a `mol`-style
project needs (CLAUDE.md, `.agent/`, `.claude/specs/`). It never
writes project source — that is `mol`'s job.

## Install

```
/plugin marketplace add https://github.com/MolCrafts/claude-plugin
/plugin install mol-agent@molcrafts
```

For local development:
`/plugin marketplace add <path-to-claude-plugin-checkout>`.

Most users will install `mol` and `mol-agent` together.

## Skills

| Skill | Purpose |
|---|---|
| `/mol-agent:bootstrap` | First-time agent-harness initializer. Inspects the repo, classifies what already exists, installs only the minimum guardrails (CLAUDE.md + `.agent/` scaffolding + `.claude/specs/`, optionally `.claude/` skills/agents). Idempotent. **Not** a template copier. Writes harness files only; never project source. |
| `/mol-agent:update` | Idempotent re-bootstrap on an already-installed harness. Pulls latest plugin templates into managed sections, rescans the repo to refresh `mol_project:` frontmatter, migrates older harness layouts (e.g. `.agent/specs/` → `.claude/specs/`) to current conventions. Writes only managed sections and the frontmatter block. |
| `/mol-agent:check` | Inspect the harness and report what's installed, what's missing, what's misaligned. Combines a quick presence/health pass (CLAUDE.md present, `mol_project:` parseable, spec INDEX consistent) with the full harness-engineering design audit (layering + two-layer model + orthogonality + knowledge + capability + workflow + output + idempotency). Read-only. |

## Four-zone layering (active vs passive)

The harness `mol-agent` installs separates four kinds of content:

| Zone        | Purpose                                                         |
|-------------|-----------------------------------------------------------------|
| `docs/`     | public-facing documentation (tutorials, API, user guides)       |
| `.agent/`   | passive internal context (notes, decisions, contracts, handoffs, rubrics, debt, open questions) — outlives any single feature |
| `.claude/`  | Claude Code runtime + active artifacts (skills, agents, hooks, settings, AND `specs/` — alive, ticked off as `/mol:impl` works, deleted on completion) |
| `CLAUDE.md` | thin entry router — points to where things live; no manual     |

Full rules in [`docs/design-principles.md`](../mol/docs/design-principles.md).
Run `/mol-agent:check` to verify compliance.

## Adopt in a project

1. `/mol-agent:bootstrap` — first time, in the project root.
2. `/mol-agent:check` — verify.
3. `/mol-agent:update` — when the plugin is upgraded or the repo's
   stack changes.

## License

MIT — see the root [LICENSE](../../LICENSE).
