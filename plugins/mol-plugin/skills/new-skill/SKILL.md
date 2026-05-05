---
description: Scaffold a new skill inside one of the molcrafts plugins (mol, mol-agent, mol-plugin). Creates the skill directory, a SKILL.md with the project's frontmatter shape (description, argument-hint), and a stub procedure. Writes only inside plugins/<plugin>/skills/<name>/.
argument-hint: "<plugin>:<skill-name> [\"one-line description\"]"
---

# /mol-plugin:new-skill — Skill Scaffold

Scaffold a new skill in this marketplace. Use when adding a verb
that doesn't yet exist — e.g. `mol:bench`, `mol-agent:foo`,
`mol-plugin:audit-templates`.

This skill writes only inside
`plugins/<plugin>/skills/<skill-name>/`. It never touches existing
skills, READMEs, or plugin metadata.

## Procedure

### 1. Parse arguments

Argument shape: `<plugin>:<skill-name> [description]`.

Validate:

- `<plugin>` is one of `mol`, `mol-agent`, `mol-plugin` (or another
  directory that already exists under `plugins/`)
- `<skill-name>` is kebab-case, no spaces, not already taken
- description, if given, fits one sentence

If anything fails validation, report and stop.

### 2. Pick a sibling as a model

Read one existing SKILL.md under the same plugin to match the local
voice and frontmatter shape. Default models:

- `mol` → `plugins/mol/skills/note/SKILL.md` (mid-complexity,
  writing skill)
- `mol-agent` → `plugins/mol-agent/skills/check/SKILL.md` (read-only)
  or `plugins/mol-agent/skills/update/SKILL.md` (writing)
- `mol-plugin` → `plugins/mol-plugin/skills/check/SKILL.md`

Note the structure: frontmatter (description + argument-hint), an
H1 with `/<plugin>:<skill>` heading, a one-paragraph purpose,
numbered Procedure, optional Guardrails, optional Idempotency,
Output format.

### 3. Generate the stub

Create `plugins/<plugin>/skills/<skill-name>/SKILL.md` with:

```markdown
---
description: <user-supplied description, or "TODO: one-sentence purpose">
argument-hint: "<placeholder, e.g. [arg1] [arg2]>"
---

# /<plugin>:<skill-name> — <Title>

<one paragraph: what this skill does and when to use it>

## Procedure

### 1. <step>

<what to do>

### 2. <step>

<what to do>

## Output format

<what the user sees when this skill runs>
```

Do not invent procedure steps. Leave the body skeletal — it is the
author's job to fill it in.

### 4. Reach approval

Show the user the path you would create and the stub body. Ask for
go-ahead. Do not write before approval.

### 5. Apply

Write the file. Report `created plugins/<plugin>/skills/<skill-name>/SKILL.md`.

### 6. Suggest next steps

Tell the user:

- run `/mol-plugin:check` once the body is filled in
- update `plugins/<plugin>/README.md`'s skills table (manually —
  this skill does not edit READMEs)

End with a one-line summary (F2).

## Guardrails

- **Do not** create a skill in a plugin directory that doesn't
  already exist; that would imply scaffolding a whole new plugin,
  which is a bigger decision.
- **Do not** copy another skill's procedure body. Frontmatter shape
  and headings are fine to mirror; specific steps must be authored.
- **Do not** edit the plugin's README, plugin.json, or any other
  file. This skill's surface is exactly one new SKILL.md.

## Output format

- Validation result: pass or specific failure.
- Plan: path + skeleton preview.
- Application: `created <path>`.
- Next-step prompt: one line.
- Final summary (F2): one line.
