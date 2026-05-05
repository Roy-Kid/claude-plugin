---
description: Cut a new release of one or more plugins in this marketplace. Bumps version in plugin.json + marketplace.json, generates a changelog from git log grouped by conventional-commit type, runs /mol-plugin:check, and prepares a tagged commit. Writes plugin.json, marketplace.json, and CHANGELOG.md inside the chosen plugins.
argument-hint: "<plugin> <bump> | <plugin>=<bump> [<plugin>=<bump> ...]   (bump = patch|minor|major)"
---

# /mol-plugin:release — Plugin Release

Cut a new release of one or more plugins. Use when changes have
landed on the default branch and you want to ship them.

This skill writes only:

- `plugins/<plugin>/.claude-plugin/plugin.json` (`version` field)
- `.claude-plugin/marketplace.json` (matching `version` field)
- `plugins/<plugin>/CHANGELOG.md` (prepended, never rewritten)
- one git commit and one git tag per plugin (when the user opts in)

## Procedure

### 1. Parse arguments

Forms accepted:

- `<plugin> <bump>` — release one plugin (`mol patch`)
- `<p1>=<bump> <p2>=<bump>` — release multiple in one go
  (`mol=minor mol-agent=patch`)

Validate:

- every named plugin exists under `plugins/`
- every bump is `patch | minor | major`

### 2. Confirm clean tree

`git status --porcelain` must be empty. If not, stop and ask the
user to commit or stash. Releases on dirty trees produce noisy
diffs and ambiguous tags.

Confirm we're on the default branch (or warn if not).

### 3. Run validation

Invoke `/mol-plugin:check` for the named plugins (or all, if the
release touches shared marketplace.json fields). If the verdict is
`FIX REQUIRED`, stop. The user can override with explicit
confirmation, but default is to halt.

### 4. Compute new versions

For each plugin:

- read current `version` from `plugin.json`
- bump per requested level (semver: `0.1.3` + patch → `0.1.4`,
  `0.1.3` + minor → `0.2.0`, `0.1.3` + major → `1.0.0`)
- record old → new

### 5. Build changelog entries

For each plugin, find the last release tag (`<plugin>-v<version>`).
Take the git log range `<last-tag>..HEAD`, filtered to commits that
touched `plugins/<plugin>/**` (or, when there's no last tag, the
whole history of that path).

Group by conventional-commit type:

- `feat:` → **Features**
- `fix:` → **Fixes**
- `refactor:` → **Refactors**
- `docs:` → **Docs**
- `perf:` → **Performance**
- `test:` → **Tests**
- `chore:` / `ci:` → **Internal** (or omit if empty)
- anything else → **Other**

Skip merge commits. One bullet per commit, format
`- <subject> (<short-sha>)`.

If there are zero non-trivial commits in scope, ask the user
whether to release anyway (a documentation-only or version-bump
release may still be intentional).

### 6. Reach approval

Show the user, per plugin:

- old version → new version
- the proposed CHANGELOG entry (full preview)
- the proposed commit message (`release: <plugin> v<new>`)
- the proposed tag (`<plugin>-v<new>`)

Wait for go-ahead. Allow per-plugin opt-out.

### 7. Apply

For each approved plugin:

- write `plugin.json` with the new version
- update the matching entry in `marketplace.json`
- prepend the new entry to `plugins/<plugin>/CHANGELOG.md`
  (creating it if absent, with an `# Changelog` heading)

Stage and commit:

```
git add plugins/<plugin>/.claude-plugin/plugin.json
git add plugins/<plugin>/CHANGELOG.md
git add .claude-plugin/marketplace.json
git commit -m "release: <plugin> v<new>"
git tag <plugin>-v<new>
```

If multiple plugins are released in one go, do one commit per
plugin (cleaner tag history) unless the user asks for a single
combined commit.

Do **not** push. The user pushes themselves when they are ready.

### 8. Report

Per-plugin one-line summary:
`released mol v0.1.0 → v0.2.0 (tag mol-v0.2.0, 7 commits)`

Final summary (F2): one line — what was released, that nothing
was pushed, and the suggested next step (push + create release on
GitHub).

## Guardrails

- **Do not** push to a remote. Tagging and committing locally is
  the boundary.
- **Do not** force-overwrite an existing tag. If `<plugin>-v<new>`
  already exists, stop.
- **Do not** rewrite an existing CHANGELOG entry. Always prepend.
- **Do not** release if `/mol-plugin:check` reports `FIX
  REQUIRED` without explicit user override.
- **Do not** invent commits or rephrase commit messages. Use them
  verbatim in the changelog (subject only).

## Idempotency

Releasing the *same* version twice is an error: tag collision
stops the run. To redo a botched release, the user must delete the
tag manually and re-run; this skill does not delete tags.

A run that reaches the approval gate and is rejected leaves the
tree exactly as it found it. No file writes happen before approval.
