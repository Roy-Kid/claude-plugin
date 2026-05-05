# mol

Shared molcrafts project-workflow skills and common-axis agents,
built around **harness engineering**: give the repository a small,
well-shaped harness — principled boundaries, predictable layers, and
just enough scaffolding to make safe defaults the obvious move — so
the *next* agent that walks in succeeds without re-deriving the rules.

`mol` is the day-to-day toolbox: implement, spec, review, test, fix,
refactor, ship. The harness *itself* (CLAUDE.md, `.agent/`,
`.claude/specs/`) is installed and maintained by the sibling plugin
[`mol-agent`](../mol-agent/README.md).

Skills adapt to each project by reading a `mol_project:` YAML
frontmatter block at the top of the project's `CLAUDE.md` — so one
plugin serves Atomiverse, molpy, molexp, molrs, molvis, molq, and
molnex without per-project forks.

## Install

```
/plugin marketplace add https://github.com/MolCrafts/claude-plugin
/plugin install mol@molcrafts
/plugin install mol-agent@molcrafts
```

For local development:
`/plugin marketplace add <path-to-claude-plugin-checkout>`.

## Four-zone layering (active vs passive)

Every well-shaped repository the `mol` plugin works on separates four
kinds of content:

| Zone        | Purpose                                                         |
|-------------|-----------------------------------------------------------------|
| `docs/`     | public-facing documentation (tutorials, API, user guides)       |
| `.agent/`   | passive internal context (notes, decisions, contracts, handoffs, rubrics, debt, open questions) — outlives any single feature |
| `.claude/`  | Claude Code runtime + active artifacts (skills, agents, hooks, settings, AND `specs/` — alive, ticked off as `/mol:impl` works, deleted on completion) |
| `CLAUDE.md` | thin entry router — points to where things live; no manual     |

The split between `.agent/` and `.claude/` is **active vs passive**:
notes are kept; specs are intentionally ephemeral. Full rules in
[`docs/design-principles.md`](docs/design-principles.md). Run
`/mol-agent:check` to verify compliance.

## Skills

All `mol` skills require a `mol_project:` frontmatter in CLAUDE.md
(see [`docs/claude-md-metadata.md`](docs/claude-md-metadata.md)) and
fail fast with an adoption hint when it is missing. To create
CLAUDE.md and the surrounding harness, run `/mol-agent:bootstrap`
first.

| Skill | Purpose |
|---|---|
| `/mol:impl` | Full implementation workflow: scope → litrev → spec → arch → TDD → impl → verify. Writes code, tests, docs. Reads the spec's Tasks checklist, ticks each box as work completes, and deletes the spec (plus its INDEX entry) on completion. |
| `/mol:spec` | Natural-language requirement → structured spec under `.claude/specs/` with a checkbox-tracked Tasks section. Detects conflicts with existing specs and updates the old spec body and its checkboxes in place when superseded or refined. |
| `/mol:litrev` | Literature + reference-implementation review (gated on `science.required`). |
| `/mol:arch` | Architecture compliance against `arch.rules_section`. Read-only. |
| `/mol:review` | Multi-axis review aggregator (architect ∥ optimizer ∥ scientist ∥ compute-scientist ∥ documenter ∥ undergrad ∥ pm). Read-only. |
| `/mol:test` | Run tests via `build.test`, analyze gaps. |
| `/mol:perf` | Hot-path review. The optimizer agent detects which anti-pattern catalogs apply (numpy / pytorch / cuda / simd-xsimd / wasm-bridge / subprocess / async-io / web-render) per file under review — no project-wide enum, multi-facet projects activate multiple catalogs. Read-only. |
| `/mol:docs` | Doc audit keyed to `doc.style`. Writes docs. |
| `/mol:note` | Capture decision into `.agent/notes.md`, detect conflicts, promote stable rules into CLAUDE.md (or a `.agent/<topic>.md` page). |
| `/mol:fix` | Minimal-diff bug-fix loop. Writes code. |
| `/mol:debug` | Diagnose-only — never writes code. Use `/mol:fix` to patch. |
| `/mol:refactor` | Restructure preserving invariants. Writes code. |
| `/mol:ship` | Pre-commit / pre-push / pre-merge CI-parity gate (PROCEED / BLOCK). Read-only. |

Harness lifecycle (install / update / audit) lives in the
[`mol-agent`](../mol-agent/README.md) plugin: `/mol-agent:bootstrap`,
`/mol-agent:update`, `/mol-agent:check`.

## Agents

Agents are invoked only through skills (never directly by the user).
Each owns one expertise axis.

| Agent | Axis |
|---|---|
| `architect` | Module boundaries, layer rules, dependency graph |
| `tester` | TDD red-before-green, coverage gaps, tolerance discipline |
| `scientist` | Equations, units, conservation, literature (every claim cites a fetched-this-run reference or derives inline) |
| `compute-scientist` | Numerical stability, complexity, determinism, HPC / DDP readiness |
| `optimizer` | Hot-path performance. Detects per-file which anti-pattern catalog(s) apply (numpy / pytorch / cuda / simd-xsimd / wasm-bridge / subprocess / async-io / web-render). |
| `documenter` | Docstrings + narrative docs, audience locked to a capable but uninitiated undergraduate (no jargon-without-definition, no "as is well known") |
| `undergrad` | User's-perspective review: API, onboarding, extension ergonomics |
| `pm` | Product-lead review: public-surface discipline, breaking-change analysis |
| `ci-guard` | CI-parity: detects CI config, runs tiered local equivalent |
| `web-design` | Visual / UX review on frontend code — design-token consistency, info density, empty/error/loading states, a11y (keyboard, focus, aria, contrast), responsive behavior. Detect-then-run: self-skips files without JSX/Vue/Svelte content. |
| `security-reviewer` | Adversarial-input review — shell / SQL / path / SSRF / prompt injection, deserialization hazards, secret leakage, missing authorization on mutating endpoints. Detect-then-run: self-skips files outside the attack-surface signal set. |
| `janitor` | Continuous tech-debt servicing — applies the project's captured `.agent/` aesthetic rules to every diff (dead code, stale TODOs, magic numbers, naming drift). Pays down debt a little every review instead of letting it accumulate. |
| `reviewer` | Review-output aggregator (also surfaces `janitor`'s rule-capture suggestions for `/mol:note`) |

Review-style agents emit `<emoji> file:line — message` using 🚨 Critical,
🔴 High, 🟡 Medium, 🟢 Low.

## Design contract

The plugin follows the harness-engineering layering plus a strict
two-layer model (skill = verb + workflow; agent = single expertise
axis). Layering, orthogonality, knowledge-locality, capability,
workflow, output, and idempotency rules are spelled out — and
audit-checked — in [`docs/design-principles.md`](docs/design-principles.md).
Run `/mol-agent:check` against any project's harness to verify
compliance. (The marketplace repo itself has no harness; use
`/mol-plugin:check` for its self-audit.)

## Adopt in a project

1. Run `/mol-agent:bootstrap` from the project root. It inspects the
   repo, asks what to add, and installs only what's justified —
   including the `mol_project:` frontmatter if you opt in.
2. Smoke-test with `/mol:arch` and `/mol-agent:check`.

Each project's harness is rewritten in place rather than migrated in
phases — this is continuous iteration.

## License

MIT — see the root [LICENSE](../../LICENSE).
