---
description: Inspect a repository's agent harness and report what's installed, what's missing, what's misaligned. Combines a quick presence/health check with the full harness-engineering design audit (layering + two-layer model + orthogonality + knowledge + capability + workflow + output + idempotency). Read-only.
argument-hint: "[<project-root>]"
---

# /mol-agent:check — Harness Inspection

Inspect a repo's agent harness — `docs/`, `.claude/notes/`, `.claude/`, `CLAUDE.md` — and produce one report covering:

1. **Health** — installed and internally consistent?
2. **Design** — follows `${CLAUDE_PLUGIN_ROOT}/rules/design-principles.md`?

Apply to any project using the mol harness. **Not** for the `claude-plugin/` marketplace itself — use `/mol-plugin:check` for that self-audit.

**Read-only.** Reports findings; never edits. Use `/mol-agent:bootstrap` (first time) or `/mol-agent:update` (idempotent re-run) to apply fixes.

## Procedure

### Phase 1 — Health (presence & consistency)

Cheap checks first; fail-fast if the harness isn't there:

- **Presence** — `CLAUDE.md` and at least one of `.claude/notes/`, `.claude/specs/` exist? If none → *"no harness installed — run `/mol-agent:bootstrap`"* and stop.
- **`mol_project:` frontmatter** (when project opted into `mol` contract) — YAML at top of CLAUDE.md parseable? Required fields populated (per `${CLAUDE_PLUGIN_ROOT}/rules/claude-md-metadata.md`)? Flag missing/malformed as 🔴.
- **Spec INDEX consistency** — every file in `.claude/specs/` listed in `INDEX.md`? Every entry has a file? Mismatches are 🟡.
- **Status / checkbox consistency** per spec:
  - `status: done` + file still present → 🟡 (`/mol:impl` should have deleted)
  - `status: done` + unchecked task boxes → 🔴 contradiction
- **Stable markers** — every `<!-- mol-agent:bootstrap:managed begin -->` has matching `end -->`. Orphans are 🔴.

### Phase 2 — Design (full checklist)

Read `${CLAUDE_PLUGIN_ROOT}/rules/design-principles.md`. Walk each section against target tree.

#### Layering (L)

Inspect four zones, **active vs passive** split inside `.claude/`:

- `docs/` — exists? Only public-user content (tutorials, API ref, user guides)? Flag any agent contract / handoff / working memory / rubric / **spec** under `docs/`. (L1, L2)
- `.claude/notes/` — passive internal context. Look for `notes.md`, `architecture.md`, plus optional `decisions/`, `contracts/`, `handoffs/`, `rubrics/`, `debt/`, `open-questions.md`. Free of public-user prose? Free of active runtime artifacts (specs)? (L1)
- `.claude/specs/` — active runtime artifacts. Per spec:
  - **Tasks** section with checkboxes? (L4) Flag if missing.
  - (Status/checkbox consistency checked in Phase 1.)
- `.claude/agents/`, `.claude/skills/`, `.claude/hooks/`, `.claude/settings.json` — Claude Code's own runtime config. Flag long-lived domain knowledge embedded in skill/agent bodies that belongs in `.claude/notes/` or CLAUDE.md. (L2). Also flag project notes mixed into `.claude/agents/` (common newcomer mistake — `agents/` is for *agent definitions*, project notes go in `.claude/notes/`).
- Anything else directly under `.claude/` (`.md` or directory not listed above) → flag as zone-mixing. Loose files at `.claude/` root are always a smell. (L1, L2)
- `CLAUDE.md` — line-count it. ≤ ~150 lines soft budget. Flag if it embeds large rule sets, full architecture diagrams, long examples, anything else that should be linked. (L3)

For projects opted into `mol` contract, confirm CLAUDE.md starts with `mol_project:` frontmatter and:

- `specs_path` → under `.claude/specs/` (canonically just `.claude/specs/`). Flag if `docs/`, `.claude/notes/`, or bare `.claude/`.
- `notes_path` → under `.claude/notes/` (canonically `.claude/notes/notes.md`). Flag if under `.claude/specs/`, `.claude/` root (e.g. `.claude/NOTES.md`), `docs/`, or anywhere outside `.claude/notes/`.

#### Layer presence

- `skills/*/SKILL.md` and `agents/*.md` both exist (when project ships custom)?
- Every user-invocable verb in `skills/`, not `agents/`?
- CLAUDE.md acting as router?

#### Orthogonality (O)

- O1 — list each agent's single axis. Flag overlap.
- O2 — grep agent bodies for `delegate to .* agent` / `invoke .*-agent`. Agents must not call agents.
- O3 — each skill names its delegates explicitly (no implicit routing).
- O4 — skills contain no expert knowledge (worked examples, reference data, tolerances). If found, flag as agent extraction.

#### Knowledge (K)

- K1 — every agent's first non-frontmatter line mentions reading CLAUDE.md (and project notes file).
- K2 — `/mol:note`-equivalent skill exists, enforces conflict detection + stale sweep + promotion.
- K3 — agent prompts don't duplicate CLAUDE.md content.
- K4 — skill prompts don't restate domain rules; reference CLAUDE.md.

#### Capability (C)

- C1 — read-only agents declare only Read/Grep/Glob/Bash (plus optional WebSearch/WebFetch for scientist). Flag Write/Edit on non-write-capable roles.
- C2 — every writing skill's `description` frontmatter states what it writes.
- C3 — diagnose-only skills enforce read-only in procedure body, not just description.

#### Workflow (W)

- W1 — Plan (spec/litrev) / Build (impl/fix/refactor) / Verify (arch/review/test/perf) / Document (docs) / Memory (note) — each phase has ≥1 skill.
- W2 — implementation skill calls `tester` agent before any implementation.
- W3 — review-style skills fan out in parallel (one message, multiple tool calls).
- W4 — architecture validation scheduled at: (a) scope assessment in `impl`, (b) post-impl, (c) post-refactor.
- W5 — debug and fix are separate skills; fix doesn't subsume debug.

#### Output (F)

- F1 — review-style agents output `<emoji> file:line — message` with 🚨 / 🔴 / 🟡 / 🟢 (Critical / High / Medium / Low).
- F2 — skills end with one-line user-facing summary.

#### Idempotency (I)

- I1 — bootstrap-style skills detect existing files; offer merge/replace/keep instead of overwriting. Look for stable markers on managed sections.
- I2 — notes skill does conflict + duplicate detection before writing.
- I3 — generators write managed sections via stable markers.

#### Anti-patterns

Spot-check `rules/design-principles.md` § 7:

- template sprawl (large skill/agent set on tiny repo)
- knowledge in `.claude/` (rules in skill/agent bodies that should be in CLAUDE.md or `.claude/notes/`)
- contracts in `docs/`
- CLAUDE.md as manual
- fake precision (architecture rules without evidence)
- agent calling agent
- non-idempotent generators

#### Per-project domain coverage

Identify each project-specific axis of risk; confirm each has an agent. For the plugin itself, the thirteen generic axes (architecture / tests / science / numerics / performance / docs / user-experience / product / CI-parity / visual-design / security / hygiene / review) are expected coverage. Visual-design and security are detect-then-run — only contribute when diff actually contains frontend code or attack surface.

## Output

Severity-sorted findings (🚨 Critical, 🔴 High, 🟡 Medium, 🟢 Low):

```
<emoji> <path> — message
  Rule: <rule id, e.g. H (health), L2, O2, K3, I1>
  Fix: <recommendation, often "run /mol-agent:update">
```

Summary table:

| Rule family       | 🚨 | 🔴 | 🟡 | 🟢 |
|-------------------|----|----|----|----|
| Health (H)        |    |    |    |    |
| Layering (L)      |    |    |    |    |
| Orthogonality (O) |    |    |    |    |
| Knowledge (K)     |    |    |    |    |
| Capability (C)    |    |    |    |    |
| Workflow (W)      |    |    |    |    |
| Output (F)        |    |    |    |    |
| Idempotency (I)   |    |    |    |    |

Verdict: APPROVE / REQUEST CHANGES / BLOCK.

End with one-line user-facing summary (F2).
