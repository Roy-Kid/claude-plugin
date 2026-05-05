---
description: Cut a new release of one or more plugins in this marketplace. Bumps version in plugin.json + marketplace.json, runs /mol-plugin:check, and prepares a tagged commit. Writes plugin.json and marketplace.json only — never a CHANGELOG.
argument-hint: "<plugin> <bump> | <plugin>=<bump> [<plugin>=<bump> ...]   (bump = patch|minor|major)"
---

# /mol-plugin:release — Plugin Release

Cut a new release of one or more plugins. Use when changes have
landed on the default branch and you want to ship them.

This skill writes only:

- `plugins/<plugin>/.claude-plugin/plugin.json` (`version` field)
- `.claude-plugin/marketplace.json` (matching `version` field)
- one git commit and one git tag per plugin (when the user opts in)

This skill does **not** write or maintain CHANGELOG files.
Release notes live in the GitHub release body and the git history;
this skill only bumps versions and tags.

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

### 5. Reach approval

Show the user, per plugin:

- old version → new version
- the proposed commit message (`release: <plugin> v<new>`)
- the proposed tag (`<plugin>-v<new>`)

Wait for go-ahead. Allow per-plugin opt-out.

### 6. Apply

For each approved plugin:

- write `plugin.json` with the new version
- update the matching entry in `marketplace.json`

Stage and commit:

```
git add plugins/<plugin>/.claude-plugin/plugin.json
git add .claude-plugin/marketplace.json
git commit -m "release: <plugin> v<new>"
git tag <plugin>-v<new>
```

If multiple plugins are released in one go, do one commit per
plugin (cleaner tag history) unless the user asks for a single
combined commit.

Do **not** push. The user pushes themselves when they are ready.

### 7. Report

Per-plugin one-line summary:
`released mol v0.1.1 → v0.1.2 (tag mol-v0.1.2)`

Final summary: one line — what was released, that nothing was
pushed, and the suggested next step (push + create release on
GitHub, with release notes drafted there from the git log).

## Guardrails

- **Do not** push to a remote. Tagging and committing locally is
  the boundary.
- **Do not** force-overwrite an existing tag. If `<plugin>-v<new>`
  already exists, stop.
- **Do not** create or modify any CHANGELOG file. The marketplace
  policy is "no CHANGELOG; release notes belong on GitHub releases
  and in `git log`."
- **Do not** release if `/mol-plugin:check` reports `FIX
  REQUIRED` without explicit user override.

## Idempotency

Releasing the *same* version twice is an error: tag collision
stops the run. To redo a botched release, the user must delete the
tag manually and re-run; this skill does not delete tags.

A run that reaches the approval gate and is rejected leaves the
tree exactly as it found it. No file writes happen before approval.
