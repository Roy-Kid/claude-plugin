---
description: Push the current branch to origin (the contributor's fork) after gating with /mol:ship push (format + lint + full test suite). Auto-runs /mol:commit first if the working tree is dirty. Follows the standard GitHub fork convention — origin = your fork, upstream = the canonical repo — and never pushes branches to upstream.
argument-hint: "[<branch>]"
---

# /mol:push — Gated Push to Fork

Writes to a remote: `git push origin`. Standard GitHub fork-and-PR layout:

- **`origin`** = contributor's personal fork. Feature branches push here.
- **`upstream`** (when present) = canonical org-owned repo. Branches **never** push here from this skill — reach upstream only via `/mol:pr`.

Single-remote layouts (solo / non-fork) have no `upstream`; warn and ask explicit confirmation before pushing to `origin`.

## Procedure

### 1. Resolve branch and remote

- Branch: `$ARGUMENTS` if provided, else current (`git rev-parse --abbrev-ref HEAD`).
- Detect remotes: `git remote`. Push target = `origin`. Missing → stop.

`upstream` exists and `origin == upstream` (same URL) → warn (no fork/canonical separation); ask explicit confirmation.

Only `origin` exists (no `upstream`) → warn: "no upstream remote configured — pushing directly to origin." Ask confirmation. (User can `git remote add upstream <url>` for the safety check.)

### 2. Refuse default branch on a forked repo

Resolve upstream default branch:

```
git symbolic-ref --short refs/remotes/upstream/HEAD 2>/dev/null \
  | sed 's@^upstream/@@'
```

Fallback: `git ls-remote --heads upstream main master | head -1`.

Current branch == default + `upstream` exists → stop. Pushing fork's `master`/`main` is rarely intent; ask user to create a feature branch.

No `upstream` → skip this check.

### 3. Commit pending work first

`git status --porcelain` non-empty → invoke `/mol:commit` (which runs `/mol:ship commit` gate). BLOCK → stop.

Clean tree → skip.

### 4. Run the push gate

Invoke `/mol:ship push`. The `push` tier is `commit` ⊇ format + lint + pre-commit ⊇ full test suite. Budget 5–10 min.

**BLOCK** → stop, relay top blocker + suggested `/mol:fix` / `/mol:impl` action. Do not push.
**PROCEED** → continue.

### 5. Push

```
git push -u origin <branch>
```

`-u` on first push so subsequent `git pull` / `git push` without args target `origin`. Never use `--force` / `--force-with-lease` from this skill — non-fast-forward needed → user does it manually.

### 6. Report

```
/mol:push: pushed <branch> → origin/<branch>
  <short-sha-range>

next: /mol:pr     (open PR against upstream/<default_branch>)
```

Already exists on origin + fast-forward → mention. Remote rejected → surface git error verbatim and stop.

## Guardrails

- **Never** push to `upstream`. Branches reach upstream only via `/mol:pr`.
- **Never** force-push (`-f` / `--force` / `--force-with-lease`).
- **Never** push the default branch from a fork without explicit user confirmation.
- **Never** skip the `/mol:ship push` gate.
- **Never** push tags from this skill — tags go through `/mol:tag`.

## Idempotency

Up-to-date branch → no-op (`Everything up-to-date`). Reported as success.
