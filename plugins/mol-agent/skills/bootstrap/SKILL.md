---
description: Repository-specific agent-harness initializer. Inspects the repo, discovers local conventions, and installs only the minimum guardrails needed for future agents to work safely. Idempotent — safe to re-run. Writes CLAUDE.md and the agent-harness scaffolding (.claude/notes/ and/or .claude/) only; never writes project source.
argument-hint: "[<purpose>]"
---

# /mol-agent:bootstrap — Agent Harness Bootstrap

Initialize or improve a repo's agent-facing setup. Separate the four content kinds (public docs / internal agent context / runtime / router); install only minimum guardrails this repo needs.

Does not require existing `mol_project:` frontmatter — this is the tool that creates CLAUDE.md and optionally populates that frontmatter.

---

## Core principle — four-zone separation, active vs passive

```
docs/                    public-facing documentation
.claude/                 (canonical Claude Code project folder)
  notes/                 passive internal context — notes, architecture.md,
                         decisions, contracts, handoffs, rubrics, debt,
                         open questions. Outlives features.
  specs/                 active runtime artifacts — alive, ticked off as
                         /mol:impl works, deleted on completion.
  agents/, skills/,      Claude Code's own runtime configuration.
  hooks/, settings.json
CLAUDE.md                thin entry router — points at where things live;
                         does not embed all the rules.
```

`.claude/` is Claude Code's canonical project folder. Inside, **active vs passive**: `.claude/notes/` passive (kept), `.claude/specs/` active (ephemeral).

`.claude/notes/` (project knowledge agent reads) ≠ `.claude/agents/` (Claude Code agent definitions).

Never: pollute `docs/` with internal agent artifacts; put specs in `docs/` or `.claude/notes/`; put long-lived knowledge in `.claude/specs/`. Keep CLAUDE.md short.

If the repo already has a working equivalent for any zone, **respect that**.

---

## Procedure

### 1. Inspect

In parallel where possible:

- cwd, git status, primary language(s) (`git ls-files | awk -F. '{print $NF}' | sort | uniq -c | sort -rn | head`)
- whether `docs/`, `.claude/notes/`, `.claude/`, `CLAUDE.md` exist + their content
- existing build/test/format commands (`pyproject.toml`, `Cargo.toml`, `package.json`, `CMakeLists.txt`, `go.mod`, `Makefile`, `.pre-commit-config.yaml`, `.github/workflows/*`)
- doc conventions (where tutorials / API docs go; project-specific style)
- existing CLAUDE.md, agent definitions, skill directories
- repo shape: library / app / docs / workflow / UI / scientific / mixed

State result in one paragraph. Don't write yet.

### 2. Classify what already exists

For every file you'd consider creating:

- **keep** — exists, fits, leave alone
- **merge** — exists, partially fits; add managed section with stable markers
- **replace** — broken/superseded; ask before overwriting
- **create** — does not exist

Flag any **misplaced** file (agent contracts in `docs/`, public tutorials in `.claude/`). Recommend a move; never auto-move.

### 3. Decide minimal addition

Smallest set that gives this repo a useful harness. Reasonable defaults:

- thin `CLAUDE.md` (router) — always
- `.claude/notes/README.md` — explains directory
- `.claude/notes/notes.md` — passive memory, `/mol:note` writes here
- `.claude/specs/` (empty dir) — `/mol:spec` writes here
- `.claude/notes/architecture.md` stub (one line: `> 跑 /mol:map 填充本蓝图` / "run /mol:map to populate this blueprint") — `librarian` consumes during `/mol:spec` Step 4.5; populated by `/mol:map`, not bootstrap

Add more only when justified by the repo:

- `.claude/notes/contracts/` — real agent handoff contracts
- `.claude/notes/rubrics/` — real review checklists
- `.claude/notes/decisions/` — substantial architectural history
- `.claude/notes/debt/` — tracked technical debt
- `.claude/notes/handoffs/` — work regularly paused mid-flight
- `.claude/skills/`, `.claude/agents/` — only if the project benefits from custom skills/agents

Prefer adding small things later on demand.

If the user wants the full `mol`-plugin contract (`mol_project:` frontmatter + canonical mol skills/agents), offer as opt-in.

### 4. Reach approval

Show short plan: what was inspected, classifications (keep/merge/replace/create/misplaced), proposals with one-line justifications, what's left alone.

Wait for explicit approval before writing.

Idempotent re-runs: plan can be one-liner ("everything in place, no action") and skip approval.

### 5. Apply

Write only the approved set. Use stable markers (`<!-- mol-agent:bootstrap:managed begin -->` … `end -->`) for managed sections so re-runs update in place.

For CLAUDE.md, prefer a short router (template at bottom). Don't turn it into a giant prompt.

If a target path exists with content you'd replace, **ask per-file**. Never silently overwrite.

### 6. Self-check

Verify result satisfies layering rules:

- `docs/` (if exists) — only public-user content (no specs, contracts, rubrics)
- `.claude/notes/` (if created/modified) — only passive internal context (no public-user prose, no specs)
- `.claude/` — only behavior (skills/agents/hooks/settings) and active artifacts (`specs/`)
- `CLAUDE.md` — thin router (≤ ~150 lines), points rather than embeds
- nothing overwritten without approval

Run `/mol-agent:check`; surface any 🚨 / 🔴.

### 7. Report

- inspected
- added or changed (with paths)
- left untouched (and why)
- open questions recorded in `.claude/notes/open-questions.md`
- suggested next improvement

End with one-line summary (F2).

---

## Guardrails

- **Don't** blindly overwrite. Use stable markers; ask per-file otherwise.
- **Don't** create boilerplate. Each file must be justified by something *observed* in the repo.
- **Don't** put agent contracts / handoffs / rubrics / temp plans / private reasoning in `docs/`. Those belong in `.claude/notes/`.
- **Don't** turn CLAUDE.md into a long manual. Router only.
- **Don't** duplicate info that exists elsewhere in the repo.
- **Don't** invent architecture rules without evidence. If unclear, record as open question.
- **Don't** create fake precision. Don't claim pytest if you didn't check.
- **Don't** modify project source. This skill never writes project source.
- **Don't** assume `mol_project:` is wanted. Offer it; let user opt in.

---

## Idempotency

Re-runs must be safe:

- existing well-shaped files → keep
- managed sections → updated in place via stable markers
- new artifacts → added only after approval
- user-authored notes / decisions / docs → never deleted or rewritten

Re-run on a healthy harness → single line: *"harness in place, nothing to change"*.

---

## CLAUDE.md template (router)

```markdown
# CLAUDE.md

<!-- mol-agent:bootstrap:managed begin -->
<!-- This block is regenerated by /mol-agent:bootstrap. Custom content goes
     OUTSIDE these markers. -->

## What this repo is

<one paragraph: what the project does, who it serves, language/stack>

## Where things live

- Source code: `<path>`
- Tests: `<path>`
- Public documentation: `docs/`
- Passive project knowledge (notes, decisions, debt, blueprint): `.claude/notes/`
- Active runtime specs (alive, deleted on completion): `.claude/specs/`
- Claude Code runtime config (agents, skills, hooks, settings):
  `.claude/agents/`, `.claude/skills/`, `.claude/hooks/`, `.claude/settings.json`

## Default workflow

For non-trivial work, prefer:
1. plan (`/mol:spec` or write to `.claude/notes/`)
2. implement (`/mol:impl` or `/mol:fix`)
3. review (`/mol:review`)
4. capture decisions (`/mol:note`)

## What must never change casually

<list invariants, public APIs, on-disk formats, contracts requiring a deliberate decision to break>

<!-- mol-agent:bootstrap:managed end -->

<!-- Free-form additions below this line are preserved across re-runs.
     If a section grows past a screen, promote to .claude/notes/<topic>.md. -->
```

If user opts into `mol` plugin contract: prepend `mol_project:` YAML per `rules/claude-md-metadata.md`; route body references to chosen `notes_path` and `specs_path`. Default `stage: experimental` (right answer pre-1.0; per `plugins/mol/rules/stage-policy.md` allowed values are `experimental`, `beta`, `stable`, `maintenance`). If Step 1 found `1.x.y` on disk (`pyproject.toml` / `Cargo.toml` / `package.json`), surface and ask whether `stable` instead — still default `experimental` if no pick.

---

## .claude/notes/ starter (if creating fresh)

```
.claude/notes/
  README.md             # what this directory is for, how to navigate
  notes.md              # evolving decisions, captured by /mol:note
  architecture.md       # project blueprint — modules, public surface, style,
                          layer roles. Stub points at /mol:map; populated
                          by /mol:map. Consumed by `librarian` during
                          /mol:spec Step 4.5.
  open-questions.md     # things uncertain during bootstrap; user fills over time
```

Add `contracts/`, `rubrics/`, `decisions/`, `debt/`, `handoffs/` **only when** repo has real content. Empty directories are not value.

## .claude/ starter (if creating fresh)

```
.claude/
  specs/                # active runtime artifacts; /mol:spec writes here,
                          /mol:impl ticks + deletes
    INDEX.md            # one-line entry per live spec; updated by /mol:spec,
                          pruned by /mol:impl
```

Skills/agents/hooks/settings under `.claude/` added later only when justified. Don't pre-create empty `skills/` or `agents/`.

---

## Output format

- Inspection: one short paragraph.
- Planning: short bulleted plan with classifications.
- Application: per-file action lines (`created`, `merged`, `kept`, `skipped — needs your call`).
- Self-check: 🚨 / 🔴 findings, or *"layering check passed"*.
- Final summary (F2): one line — what was created, skipped, next step.

Only skill that may exceed one-line summary in body — work itself is structural.
