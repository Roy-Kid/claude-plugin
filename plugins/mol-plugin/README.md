# mol-plugin

Maintenance toolkit for the molcrafts plugin marketplace itself.

`mol-plugin` is for **developing the plugins**, not for using them in a
project. If you want to *use* mol skills in a repo, install `mol`
and/or `mol-agent` instead.

## Install

```
/plugin marketplace add https://github.com/MolCrafts/claude-plugin
/plugin install mol-plugin@molcrafts
```

For local development:
`/plugin marketplace add <path-to-claude-plugin-checkout>`.

## Skills

| Skill | Purpose |
|---|---|
| `/mol-plugin:new-skill` | Scaffold a new skill in any plugin (`mol`, `mol-agent`, `mol-plugin`). Creates one SKILL.md with the project's frontmatter shape and a skeletal procedure. Does not edit READMEs or `plugin.json`. |
| `/mol-plugin:check` | The marketplace's self-audit. Structural check across the whole marketplace: `marketplace.json`, every `plugin.json`, every `SKILL.md`, every agent. Parallel to `/mol-agent:check`, but for the plugin source rather than a project's harness. Read-only. |
| `/mol-plugin:release` | Version bump (patch/minor/major) + changelog generation (grouped by conventional-commit type) + git commit + tag. Does not push. |

## Workflow

```
/mol-plugin:new-skill mol:bench "Microbenchmark hot paths"
# author the skill body
/mol-plugin:check
# fix anything red
/mol-plugin:release mol minor
# review the diff, push when ready
```

## License

MIT — see the root [LICENSE](../../LICENSE).
