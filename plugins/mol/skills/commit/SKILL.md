---
description: Stage and commit changes after running the /mol:ship commit gate (format + lint + pre-commit hooks). Refuses to commit when the gate reports BLOCK. Generates a conventional-commit message from the staged diff if the user does not supply one. Never pushes; commits are local-only.
argument-hint: "[<message>]"
---

# /mol:commit — Gated Local Commit

This skill is a **write** skill: it stages files and creates a git
commit. It is not a remote operation — pushing is `/mol:push`'s job.

The contract is "no commit without a passing pre-commit gate." The
gate itself lives in `/mol:ship commit` so commit and CI-parity stay
factored apart.

## Procedure

### 1. Sanity check

If the working tree is clean (`git status --porcelain` empty), stop
and tell the user there is nothing to commit.

### 2. Decide what to stage

- If anything is **already staged**, treat that as the commit's
  contents. Do not auto-add unstaged files; the user has already
  curated the staging area.
- If nothing is staged but there are unstaged changes, list them
  and ask the user whether to `git add` them all or pick a subset.
  Never run `git add -A` or `git add .` silently — that risks
  picking up `.env`, credentials, or large binaries.

Stage by exact paths only. Never pass `-A` / `.` / globs.

### 3. Run the pre-commit gate

Invoke `/mol:ship commit`. If the verdict is **BLOCK**, stop and
relay the top blocker plus the `/mol:fix` (or `/mol:impl` /
`/mol:refactor`) action that `/mol:ship` recommended. Do not
commit.

If the verdict is **PROCEED**, continue.

### 4. Resolve the commit message

If `$ARGUMENTS` is non-empty, treat it as the commit subject.

Otherwise, generate a conventional-commit message from the staged
diff:

- Type: `feat` (new capability), `fix` (bug fix), `refactor` (no
  behavior change), `docs`, `test`, `chore`, `perf`, `ci` —
  inferred from which paths changed.
- Subject: ≤ 72 chars, imperative mood, no trailing period.
- Body (optional): wrap at 72 cols, explain *why* not *what* when
  the diff is non-obvious.

Show the proposed message to the user and wait for approval before
committing. The user may edit it inline.

### 5. Commit

```
git commit -m "<subject>" [-m "<body>"]
```

Never pass `--no-verify`, `--no-gpg-sign`, or `--amend`. If a
pre-commit hook fails *despite* the `/mol:ship commit` gate
reporting PROCEED, that is a real signal — surface the hook output
to the user, fix, re-stage, and run `/mol:commit` again. Do **not**
amend; a failed commit means the commit did not happen, so amending
would mutate the *previous* commit.

### 6. Report

```
/mol:commit: committed <short-sha> on <branch>
  <subject>

next: /mol:push   (push to origin = your fork)
```

## Guardrails

- **Do not** `git push`. That is `/mol:push`.
- **Do not** stage with `-A` / `.` / globs.
- **Do not** skip hooks (`--no-verify`).
- **Do not** amend (`--amend`). Always create a new commit.
- **Do not** commit `.env`, credential files, or files matching
  `*.key` / `*.pem` / `id_rsa*` even if the user staged them —
  warn and ask for explicit confirmation.

## Idempotency

Running `/mol:commit` on a clean tree is a no-op (reports "nothing
to commit"). Running it twice in a row commits twice if there are
new staged changes between the runs; otherwise the second run is a
no-op.
