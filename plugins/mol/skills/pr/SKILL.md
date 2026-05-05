---
description: Open a pull request from origin (the contributor's fork) to upstream's default branch via the gh CLI. Calls /mol:push first (which gates on /mol:ship and pushes to origin). Follows the standard GitHub fork convention — origin = your fork, upstream = the canonical repo. Drafts the PR title and body from the commit range against upstream's default branch.
argument-hint: "[<title>]"
---

# /mol:pr — Pull Request from Fork to Upstream

This skill creates a GitHub pull request. The base is always
`upstream` / `<default_branch>`; the head is `origin`
(contributor's fork). Direct upstream pushes are not in scope —
the only path from a fork branch into upstream that this plugin
supports is "PR + review."

## Procedure

### 1. Resolve config

- **Origin** must exist (`git remote get-url origin`). If not, stop.
- **Upstream** remote: `git remote get-url upstream`.
  - If absent and `origin` points to a *fork* (i.e. the GitHub
    repo's `parent` field is set), offer to add upstream
    automatically: `git remote add upstream <parent-url>`.
  - If absent and `origin` is *not* a fork, this skill cannot
    proceed — there is no separate upstream to PR against. Stop
    and tell the user.
- **Branch**: current branch (`git rev-parse --abbrev-ref HEAD`).

Resolve the upstream default branch:

```
git fetch upstream
git symbolic-ref --short refs/remotes/upstream/HEAD 2>/dev/null \
  | sed 's@^upstream/@@'
```

Fallback: `git ls-remote --heads upstream main master | head -1`.

If the current branch equals that default branch, stop. PRs from
`master`-on-fork to `master`-on-upstream are almost never the
intent; ask the user to create a feature branch first.

### 2. Verify gh is installed and authenticated

`gh auth status`. If unauthenticated, stop and tell the user to run
`gh auth login` themselves (interactive, not automatable).

### 3. Push first

Invoke `/mol:push`. It runs `/mol:commit` if the tree is dirty
(which itself runs the `/mol:ship commit` gate), then `/mol:ship
push`, then `git push origin`. If `/mol:push` reports a blocker,
stop here.

### 4. Draft title and body

The compare range for drafting is
`upstream/<default_branch>..HEAD`.

If `$ARGUMENTS` is non-empty, treat it as the PR title.

Otherwise:

- **Title** — if there is one commit in the range, use its subject
  (≤ 70 chars; truncate with `…` if longer). If multiple commits,
  pick the dominant theme by reading all subjects, prefix the
  conventional-commit type, and write a one-line summary.
- **Body** — the standard format:

  ```
  ## Summary
  - <bullet 1>
  - <bullet 2>

  ## Test plan
  - [ ] <what was tested locally>
  - [ ] <regression risk to verify>
  ```

  Bullets should explain **why**, not restate the diff. Read the
  commits + the actual diff to write them; do not just copy commit
  subjects.

Show title + body to the user and wait for approval. The user may
edit either inline.

### 5. Detect existing PR

```
gh pr view --json url,state 2>/dev/null
```

If a PR already exists for this branch and is `OPEN`, **do not
create a new one**. Report the existing URL and stop. If `CLOSED`
or `MERGED`, ask the user whether to open a new PR or revive.

### 6. Create the PR

```
gh pr create \
  --base "<default_branch>" \
  --repo "<upstream-owner>/<upstream-repo>" \
  --title "<title>" \
  --body "$(cat <<'EOF'
<body>
EOF
)"
```

`<upstream-owner>/<upstream-repo>` comes from parsing the URL of
the `upstream` remote. The head is auto-resolved by `gh` from the
branch's tracking remote, which `/mol:push` set with `-u` on
`origin`.

If `gh pr create` errors (e.g. fork has no commits diverging from
upstream, branch not pushed), surface the error verbatim.

### 7. Report

```
/mol:pr: opened PR <number> against upstream/<default_branch>
  <url>
```

## Guardrails

- **Never** create a PR with the default branch as the head.
- **Never** open a PR without going through `/mol:push` (so the
  full test suite gate runs).
- **Never** `--draft` automatically — let the user decide on the
  approval gate if they want a draft (they can edit the gh command
  before approving).
- **Never** assign reviewers / add labels automatically. Those are
  team-policy decisions, not skill defaults.

## Idempotency

Re-running `/mol:pr` on a branch that already has an open PR is a
no-op (reports the existing URL). After the existing PR is merged
or closed, a new run can open a new PR.
