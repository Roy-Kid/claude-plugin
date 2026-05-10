---
description: Upgrade an existing agent harness to the current mol-agent plugin spec — refreshes managed CLAUDE.md sections, repairs `mol_project` frontmatter, migrates legacy layouts, and rebuilds drifted spec INDEX. Use after pulling a new mol-agent version; surfaces content-level judgement calls as a TODO list and never edits user-authored prose.
argument-hint: "[<project-root>]"
---

# /mol-agent:update — Upgrade Harness to Current Version

Brings an already-installed harness **forward to the version of the mol-agent plugin currently on disk.** The plugin's own SKILL.md / templates / migration table are the *target state*.

```
bootstrap   creates the harness at the current version (no harness exists)
check       reads the harness; reports drift from the current version
update      applies the upgrade — brings the harness to the current version
```

If no harness exists, this skill says so and points at `/mol-agent:bootstrap`. Does not initialize from scratch.

## What "the current version's spec" covers

Target state = everything the plugin **as it sits on disk** dictates:

1. **Frontmatter contract** — `mol_project:` shape per `${CLAUDE_PLUGIN_ROOT}/plugins/mol/rules/claude-md-metadata.md`. Currently-required fields and current names.
2. **Managed-section templates** — `<!-- mol-agent:bootstrap:managed begin/end -->` block bodies inside `${CLAUDE_PLUGIN_ROOT}/plugins/mol-agent/skills/bootstrap/SKILL.md`.
3. **Layout conventions** — migration table § below.
4. **Structural invariants** — every spec under `specs_path/` listed in `INDEX.md` and vice versa; every stable marker has matching close.

Anything outside this list is **content**, not version-spec — never auto-modified. Examples (always manual TODO):

- knowledge embedded in skill body that should live in CLAUDE.md
- `.claude/agents/` agent whose axis overlaps another's
- contract document misplaced under `docs/`
- spec body internally inconsistent
- `status: done` spec still on disk (deletion contract belongs to `/mol:impl` § 7)

These need user reasoning. Update surfaces them faithfully (from `/mol-agent:check`) but never acts.

## Migration table (current-version layout)

| From (legacy)               | To (current)         | Why |
|-----------------------------|----------------------|-----|
| `.agent/specs/`             | `.claude/specs/`     | active vs passive split (pre-v0.3.0; specs are runtime artifacts) |
| `docs/decisions/`           | `.claude/notes/decisions/`  | internal context, not public docs |
| `docs/contracts/`           | `.claude/notes/contracts/`  | internal context |
| `docs/agent-rubrics*.md`    | `.claude/notes/rubrics/`    | internal context |
| `.claude/NOTES.md`          | `.claude/notes/notes.md`    | passive memory belongs in `.claude/notes/`, not `.claude/` root |
| `.agent/notes.md`           | `.claude/notes/notes.md`    | v0.3.0: passive context folds into `.claude/notes/` per Claude Code spec; name avoids collision with `.claude/agents/` |
| `.agent/architecture.md`    | `.claude/notes/architecture.md` | v0.3.0 |
| `.agent/decisions/`         | `.claude/notes/decisions/`  | v0.3.0 |
| `.agent/rubrics/`           | `.claude/notes/rubrics/`    | v0.3.0 |
| `.agent/contracts/`         | `.claude/notes/contracts/`  | v0.3.0 |
| `.agent/debt/`              | `.claude/notes/debt/`       | v0.3.0 |
| `.agent/handoffs/`          | `.claude/notes/handoffs/`   | v0.3.0 |
| `.agent/open-questions.md`  | `.claude/notes/open-questions.md` | v0.3.0 |
| `.agent/README.md`          | `.claude/notes/README.md`   | v0.3.0 |
| `mol_project.notes_path: .agent/...` or `.claude/NOTES.md` | `mol_project.notes_path: .claude/notes/...` | v0.3.0: frontmatter follows the path move |
| `mol_project.perf:` block in CLAUDE.md frontmatter | (delete) | `perf.focus` was a single-value enum that didn't scale; `optimizer` agent now detects catalogs per file |

Add new rows as new conventions are codified — that is how the version-spec evolves. Layout violation **not** in this table = content-level drift → manual TODO; do not invent moves.

## Procedure

### 1. Detect

Confirm harness present:

- `CLAUDE.md` exists at target root
- at least one of `.claude/notes/`, `.claude/specs/` exists

If not: *"no harness found — run `/mol-agent:bootstrap` first"* and stop.

Refuse on dirty working tree by default (moves hard to review inside dirty diff). Suggest commit-or-stash and stop; allow `--allow-dirty` (or explicit user override) to proceed.

### 2. Compute the version diff

Build "current version's expected shape" from plugin source:

- frontmatter contract from `claude-md-metadata.md`
- managed-section template body from `bootstrap/SKILL.md`
- migration table § above
- structural invariants list above

Walk harness on disk; produce **structural diff**:

- frontmatter: missing block / missing required field / stale value / required field renamed
- managed sections: marker missing / drifted body / orphan marker
- layout: any `From (legacy)` path that exists in repo
- structural invariants: spec files not in INDEX, INDEX entries with no file, unmatched markers

For DRY, also invoke `/mol-agent:check` internally; pull **content-level** findings (Layering / Orthogonality / Knowledge / Capability / Workflow / Idempotency / anti-patterns) into manual TODO bucket. Structural diff = upgrade plan; check's content findings = manual TODO list.

If structural diff empty AND check returned zero content findings → *"harness already at current version"* and stop.

### 3. Rescan repo (only when frontmatter values are queued)

If upgrade plan rewrites any `mol_project:` field's value (e.g. `build.test`), re-run bootstrap inspection — primary language, build tool, test runner, formatter, linter, numeric library use, CI config — to derive ground-truth value. Compare against existing frontmatter; record old → new.

Skip if no frontmatter values need rewriting.

### 4. Reach approval

Show one combined plan:

```
Upgrade plan (n items, target = current plugin version):
  - install mol_project frontmatter (block missing per current contract)
  - rename mol_project.foo → mol_project.bar (renamed in current contract)
  - refresh CLAUDE.md managed section (template body changed)
  - move .agent/specs/ → .claude/specs/ (current convention)
  - rebuild .claude/specs/INDEX.md (3 files unindexed)

Manual TODO (m content-level items, surfaced from /mol-agent:check):
  🔴 .claude/notes/notes.md — agent contract content here belongs in `.claude/agents/` (rule O3)
  🟡 abc-feature.md — status: done but file still present (rule H; deletion is /mol:impl's job)
  ...
```

Empty upgrade plan + non-empty manual list → *"harness already at current version; m manual TODOs from content-level review below"* + manual list.

Both empty → *"harness already at current version"*.

Otherwise wait for explicit go-ahead. Allow per-item rejection.

### 5. Apply

Per approved item:

- **frontmatter (install)** — write fresh `mol_project:` YAML at top of CLAUDE.md per current contract. Preserve any pre-existing first-line content by moving below new frontmatter.
- **frontmatter (field rename / repair)** — rewrite only affected fields. Preserve user-customization fields the contract doesn't enforce.
- **managed-section refresh** — replace contents between stable markers with current template body.
- **orphan-marker repair** — only when context unambiguous; otherwise demote to manual TODO.
- **INDEX rebuild** — regenerate from actual spec files.
- **migration** — `git mv` (or `mv`) per table; patch INDEX entries that referenced old path.

After each apply, one-line action log: `upgraded mol_project frontmatter to current contract`, `refreshed CLAUDE.md managed section`, `migrated .agent/specs/ → .claude/specs/`.

### 6. Re-run /mol-agent:check (convergence proof)

Run check on updated harness. Verify:

- applied structural drift disappeared
- no new structural findings introduced

Manual TODOs expected to remain.

Surface any 🚨 / 🔴 still standing on structural axis as a bug in *this* skill (upgrade plan should have covered them).

### 7. Report

- upgrade items applied (count + brief list)
- manual TODOs surfaced (count + severity breakdown)
- post-update structural check: clean / not clean
- suggested next step

If manual TODOs exist: *"address the manual TODOs and run `/mol-agent:check` to re-verify content-level cleanliness."*

End with one-line summary (F2).

## Guardrails

- **Don't** rewrite anything outside managed sections, `mol_project:` block, or paths approved for migration.
- **Don't** invent migrations not in the table. The table *is* the version-spec for layout.
- **Don't** delete user-authored notes / decisions / specs during migration. Move, never drop.
- **Don't** pad managed sections with content the project doesn't have. Template is a ceiling, not a floor.
- **Don't** auto-delete `status: done` spec — deletion contract belongs to `/mol:impl` § 7. Flag as manual TODO.
- **Don't** silently change anything not dictated by current version's spec.
- **Don't** run on dirty tree without explicit override.

## Idempotency

Repeated runs must converge:

- once at current version, output is one line, no writes
- managed sections byte-identical after consecutive runs (no whitespace churn)
- migrations one-shot
- post-fix re-check (step 6) is convergence proof; structural findings still standing after step 5 = bug in this skill

## Output format

- Detect: one line — *"harness found"* / *"no harness"* / *"dirty tree, aborting (use --allow-dirty)"*.
- Version diff: silent count of upgrade items + manual TODOs.
- Planning: two-section plan (Upgrade plan / Manual TODO).
- Application: per-item action lines.
- Re-check: structural findings still standing, or *"clean"*.
- Final summary (F2): one line — *N upgraded, M manual TODOs, re-check status*.
