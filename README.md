# molcrafts claude plugin marketplace

A Claude Code plugin marketplace for the molcrafts workspace
(Atomiverse, molpy, molexp, molrs, molvis, molq, molnex), built around
**harness engineering**: small, well-shaped harnesses with principled
boundaries (public docs / passive internal context / runtime + active
artifacts / thin router) so the next agent that walks in succeeds
without re-deriving the rules.

## Layout

```
claude-plugin/
├── .claude-plugin/marketplace.json   # marketplace registry
├── plugins/
│   ├── mol/                          # 17 workflow skills + 14 single-axis agents
│   │   ├── .claude-plugin/plugin.json
│   │   ├── README.md
│   │   ├── docs/
│   │   │   ├── claude-md-metadata.md # mol_project frontmatter contract
│   │   │   ├── design-principles.md  # harness layering + design rules
│   │   │   ├── agent-design.md       # producer/reviewer split rationale
│   │   │   └── evaluator-protocol.md # planner/generator/evaluator contract
│   │   ├── skills/                   # 17 SKILL.md (incl. web, simplify)
│   │   └── agents/                   # 14 agent .md (incl. playwright-evaluator)
│   ├── mol-agent/                    # 3 harness-lifecycle skills
│   │   ├── .claude-plugin/plugin.json
│   │   ├── README.md
│   │   └── skills/                   # bootstrap, update, check
│   └── mol-plugin/                   # 3 marketplace-maintenance skills
│       ├── .claude-plugin/plugin.json
│       ├── README.md
│       └── skills/                   # new-skill, check, release
├── LICENSE
└── README.md
```

## Plugins

| Plugin | Purpose |
|---|---|
| [`mol`](plugins/mol/README.md) | Day-to-day project work, organized around the planner→generator→evaluator harness pattern: `/mol:spec` (planner — self-validates spec quality and negotiates the binding `<slug>.acceptance.md` contract) → `/mol:impl` (generator — refuses without both files) → `/mol:review [--axis=<name>]` (unified static evaluator; aggregates 10 single-axis reviewers via the `reviewer` agent) → `/mol:web` (runtime UI evaluator; uses whatever browser-automation MCP you installed). Plus `/mol:fix`, `/mol:refactor`, `/mol:simplify` (apply janitor hygiene), `/mol:ship`, git workflow chain, …. Adapts to each project via `mol_project:` frontmatter. |
| [`mol-agent`](plugins/mol-agent/README.md) | Harness lifecycle: `/mol-agent:bootstrap` (install), `/mol-agent:update` (idempotent re-bootstrap), `/mol-agent:check` (presence + design audit). |
| [`mol-plugin`](plugins/mol-plugin/README.md) | Maintaining this marketplace: `/mol-plugin:new-skill`, `/mol-plugin:check` (marketplace self-audit), `/mol-plugin:release`. |

## Install

```
/plugin marketplace add https://github.com/MolCrafts/claude-plugin
/plugin install mol@molcrafts
/plugin install mol-agent@molcrafts
```

`mol-plugin` is only needed if you are developing the plugins
themselves; most users skip it.

For local development, `/plugin marketplace add <path-to-this-checkout>`
works too. Restart the session or `/reload-plugins` to pick up new
skills.

## Adopt in a project

1. Install `mol` and `mol-agent` (above).
2. From the project root, run `/mol-agent:bootstrap`. It inspects the
   repo, asks what to add, and installs only what's justified
   (CLAUDE.md + `.agent/` for passive context + `.claude/specs/` for
   active work, plus the `mol_project:` frontmatter when you opt into
   the mol contract).
3. Smoke-test with `/mol-agent:check` (harness compliance) and
   `/mol:review --axis=arch` (architecture).

Each project's harness is rewritten in place rather than migrated in
phases — this is continuous iteration. When the plugin is upgraded
later, run `/mol-agent:update` to refresh templates and frontmatter.

## License

MIT — see [LICENSE](LICENSE).
