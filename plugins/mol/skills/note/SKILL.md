---
description: Capture an architectural decision into the project's notes file. Detects conflicts with existing notes or CLAUDE.md, cleans up stale notes, and promotes stable rules into CLAUDE.md. Writes notes and (when promoting) CLAUDE.md.
argument-hint: "<what to remember>"
---

# /mol:note — Architecture Note

Read CLAUDE.md and parse `mol_project:` (`$META`).

Resolve the notes path:

1. If `$META.notes_path` is set, use it (canonical default
   `.claude/notes/notes.md`).
2. Otherwise, default to `.claude/notes/notes.md`.

Create the notes file (and any parent directory) if missing. Notes
are **passive internal context** — they live under `.claude/notes/`, never
under `docs/` (public) and never under `.claude/specs/` (active
runtime artifacts; see `/mol:spec`). The active/passive distinction
matters: notes outlive any single feature, specs do not.

## Procedure

### 1. Conflict check (MANDATORY — run before any write)

Read both the notes file and `CLAUDE.md`. For the new note:

- **Duplicate.** Existing note or CLAUDE.md section already says the
  same thing → tell the user, don't add.
- **Contradiction.** Existing note or CLAUDE.md rule says the
  *opposite*:
  ```
  CONFLICT: new note "<X>" contradicts existing "<Y>" in {file}:{line}.
  ```
  Ask the user which is correct. Then:
  - Update or delete the wrong entry.
  - If the wrong entry is in CLAUDE.md, fix CLAUDE.md.
  - If the wrong entry is in the notes file, delete or rewrite it.
- **Supersede.** New note refines or replaces an older note → update
  the old note in-place; do not keep both.

### 2. Scan for stale notes

Every invocation also scans existing notes entries:

- **Outdated.** Grep to verify the note still references live code.
  Delete notes that point at dead code.
- **Already promoted.** If the rule is now fully covered by CLAUDE.md,
  delete the note.
- **Contradicts current code.** Flag for the user.

Report any cleanup performed.

### 3. Decide placement

- Stable, proven, already verified in code → write directly to
  CLAUDE.md in the appropriate section. Keep CLAUDE.md as a thin
  router (L3): if the rule is large, write a `.claude/notes/<topic>.md`
  page and link from CLAUDE.md instead of inlining.
- New or still-evolving → write to the notes file.
- Already covered → tell the user, do nothing.

### 4. Write the entry

In the notes file, use:

```markdown
---

## [YYYY-MM-DD] Short title

Description of the decision or correction.

**Rule**: the concrete rule to follow.
```

### 5. Promotion check

After writing, scan all existing notes entries. Promote any that are:

- Verified in current code (grep to confirm).
- Unchanged in recent conversation history.

Promoting = copying the rule to the appropriate CLAUDE.md section (or
to a `.claude/notes/<topic>.md` page if the rule is large enough that
inlining would bloat CLAUDE.md past its router budget), then deleting
from the notes file.

End with a one-line summary: what was captured, what was cleaned, what
was promoted, and the path of the notes file used.
