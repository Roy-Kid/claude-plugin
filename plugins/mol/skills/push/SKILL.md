---
description: Push the current branch to origin (the contributor's fork) after gating with /mol:ship push (format + lint + full test suite). Auto-runs /mol:commit first if the working tree is dirty. Follows the standard GitHub fork convention — origin = your fork, upstream = the canonical repo — and never pushes branches to upstream.
argument-hint: "[<branch>]"
---

# /mol:push — Gated Push to Fork

This skill writes to a remote: it runs `git push origin`. The
contract assumes the standard GitHub fork-and-PR layout:

- **`origin`** is the contributor's personal fork. Feature
  branches push here.
- **`upstream`** (when present) is the canonical org-owned repo.
  Branches **never** push here from this skill — they reach
  upstream only via `/mol:pr`.

Repos with a single remote (solo-maintainer or non-fork clones)
have no `upstream`; in that case `/mol:push` warns and asks for
explicit confirmation before pushing to `origin`, because the
fork/upstream separation that this skill is designed to protect
isn't present.

## Procedure

### 1. Resolve branch and remote

- Branch: `$ARGUMENTS` if provided, else current branch
  (`git rev-parse --abbrev-ref HEAD`).
- Detect remotes: `git remote`. The push target is `origin`. If
  `origin` is missing, stop with a clear error.

If a remote literally named `upstream` exists and `origin == upstream`
(same URL), warn the user: this layout doesn't actually separate
fork from canonical. Ask for explicit confirmation.

If only `origin` exists (no `upstream` remote), warn: "no upstream
remote configured — pushing directly to origin." Ask for
confirmation. The user can add an upstream with
`git remote add upstream <url>` if they want the safety check.

### 2. Refuse default branch on a forked repo

Resolve the upstream default branch:

```
git symbolic-ref --short refs/remotes/upstream/HEAD 2>/dev/null \
  | sed 's@^upstream/@@'
```

Fallback if the symbolic-ref is unset:
`git ls-remote --heads upstream main master | head -1` — pick the
first that exists.

If the current branch equals that default branch and an `upstream`
remote exists, stop. Pushing `master` (or `main`) of a fork
straight is rarely the intent; ask the user to create a feature
branch.

If there is no `upstream` remote, skip this check.

### 3. Commit pending work first

If `git status --porcelain` is non-empty (staged or unstaged
changes), invoke `/mol:commit` and let it run its `/mol:ship
commit` gate. If `/mol:commit` reports BLOCK, stop here.

If the tree is already clean, skip this step.

### 4. Run the push gate

Invoke `/mol:ship push`. The `push` tier is `commit` ⊇ format +
lint + pre-commit ⊇ full test suite. Budget 5–10 min.

If the verdict is **BLOCK**, stop and relay the top blocker plus
the suggested `/mol:fix` / `/mol:impl` action. Do not push.

If the verdict is **PROCEED**, continue.

### 5. Push

```
git push -u origin <branch>
```

Use `-u` on the first push so subsequent `git pull` / `git push`
without arguments target `origin`. Never use `--force` or
`--force-with-lease` from this skill — if a non-fast-forward is
needed, ask the user to do it manually so they can review what is
being overwritten.

### 6. Report

```
/mol:push: pushed <branch> → origin/<branch>
  <short-sha-range>

next: /mol:pr     (open PR against upstream/<default_branch>)
```

If the branch already exists on origin and the push was a
fast-forward, mention that. If the remote rejected, surface the
git error verbatim and stop.

## Guardrails

- **Never** push to `upstream`. Branches reach upstream only via
  `/mol:pr`.
- **Never** force-push (`-f` / `--force` / `--force-with-lease`).
  Ask the user to do this manually if needed.
- **Never** push the default branch from a fork without explicit
  user confirmation.
- **Never** skip the `/mol:ship push` gate.
- **Never** push tags from this skill — tags go through `/mol:tag`.

## Idempotency

Running `/mol:push` on an already-up-to-date branch is a no-op
(`Everything up-to-date`). The skill reports this as success, not
failure.
