---
description: Convert a natural-language requirement into a structured spec under `.claude/specs/` plus a binding `<slug>.acceptance.md` contract that defines "done". Use to start any non-trivial feature; detects conflicts with existing specs and supports Chinese and English.
argument-hint: "<feature description>"
---

# /mol:spec — Specification Generator

Read CLAUDE.md → parse `mol_project:` (`$META`); else emit adoption hint and stop.

Resolve specs path: `$META.specs_path` if set (canonical default `.claude/specs/`), else `.claude/specs/`. Create dir if missing.

## What this skill produces

Two files per spec, written together:

- `<slug>.md` — the design (Summary / Design / Files / Tasks / Testing / Out of scope).
- `<slug>.acceptance.md` — binding "done" contract per `plugins/mol/rules/evaluator-protocol.md`; `/mol:impl` and runtime evaluators (`/mol:web` etc.) verify against it.

`/mol:impl` refuses to start without both. `/mol:impl` ticks Tasks and **deletes** both files when complete.

Specs do **not** belong in `docs/` (public docs) or `.claude/notes/` (passive notes).

## Procedure

### 1. Parse the request

What capability? Which layer/package/crate? Derive kebab-case slug. State in one sentence.

### 2. Search domain basis (if applicable)

Physics involved AND `$META.science.required: true` → delegate to `scientist` agent for equations / references / validation targets. Capture verbatim — feeds `spec-writer` in Step 5.

### 3. Conflict check (MANDATORY before any write)

Read every existing spec under `$META.specs_path`. Classify the new request:

- **Duplicate** — existing spec covers this scope → tell user, don't add sibling.
- **Supersede / refine** — new request changes/expands/fixes an existing spec → **update old spec in place**, no sibling. Read old body in full; pass to `spec-writer`.
- **Independent** — different scope → safe to create.

State classification before continuing. Never sibling for supersede/refine. (`morse-bond-v2.md` next to `morse-bond.md` is the wrong move — feed both into `spec-writer` with `conflict_decision: supersede:morse-bond`.)

### 4. Scan interaction points

Glob project root for files relevant to `$META.language`. Identify closest existing pattern. Flag any new public API, data-model change, cross-layer dependency. Capture as structured input for `spec-writer`.

### 4.5. Consult `librarian` (planning-time placement & reuse)

Before `spec-writer`, ask `librarian` (via `.claude/notes/architecture.md`): *"is this already there?"* and *"where does it canonically belong?"*

Invoke `librarian` with:

- `request` — user's requirement (preserve language)
- `scope_layer` — from Step 1

Returns one of two shapes:

- **Shape A** — fresh blueprint, full report: `Reuse candidates`, `Recommended placement`, `Closest pattern`, `Confidence`.
- **Shape B** — `stale: true` + one-line reason.

#### Stale-handling branch (skill orchestrates; O2 preserved)

If `stale: true`, **skill** (not agent) owns recovery routing. `librarian` MUST NOT invoke `architect`; `architect` MUST NOT invoke `librarian`. Chain:

1. skill invokes `architect` in `mode: inventory` to draft fresh catalog;
2. skill invokes `/mol:map` (gates on user-confirm) to write refreshed `.claude/notes/architecture.md`;
3. skill re-consults `librarian`; expect Shape A.

If user defers `/mol:map` write gate → draft `spec-writer` input without librarian report (note "librarian consult skipped — blueprint refresh deferred" at Step 6).

Capture final librarian report verbatim — becomes `librarian_report` input to `spec-writer`.

### 5. Delegate drafting to `spec-writer`

Invoke `spec-writer` with structured prompt:

- `request` — user's requirement (preserve language)
- `scope_layer` — Step 1
- `scientist_output` — Step 2 (or empty)
- `conflict_decision` — `independent` or `supersede:<slug>` (with old body inlined)
- `interaction_points` — Step 4
- `slug` — Step 1

Agent drafts spec body (Summary / Domain basis / Design / Files / Tasks / Testing strategy / Out of scope), self-validates against quality checklist (sections / atomic tasks / RED-before-GREEN / Files↔Tasks cross-reference), proposes acceptance criteria. Returns two markdown blocks + status line; **does not write to disk**.

Branch on `Status:`:

- `Status: ok` — proceed to Step 6.
- `Status: blocked` — surface failed items: *"I cannot satisfy <items>; do you want to relax X, or refine the request?"* Stop until user responds; on response, re-invoke from Step 5.
- `Status: split-needed` — **do not prompt user.** The large-spec split rule (`plugins/mol/rules/large-spec-split.md`) decided. Read proposed-cut chain, then **re-invoke `spec-writer` once per sub-slug in chain order** with:
    - `slug` = sub-slug (`<base>-NN-<phase>`)
    - `request` = sub-slug's scope line from cut
    - `scope_layer` = single layer that sub-spec touches
    - `interaction_points` = inherited + explicit "depends on: <earlier sub-slugs>" line
    - `conflict_decision` = `independent` (sub-specs are siblings, not supersedes; original parent slug not written)

  Each sub-invocation must return `Status: ok` (or `Status: blocked` — surface and stop). Collect all sub-spec bodies + acceptance blocks; jump to Step 6 with **full chain**, not parent.

For supersede flows, agent additionally returns `Diff vs. previous spec` block; surface in Step 6.

### 6. Show, confirm, persist

Single-spec result: show **both** files (spec body + acceptance) in same turn, exactly as `spec-writer` returned them.

Chained result: show **chain summary first** — numbered `<sub-slug>` + one-line scope, in chain order — then each pair in sequence. Call out the chain was produced by the large-spec split rule (no user prompt was issued); user can amend or delete sub-specs after persistence.

Call out:

- **librarian's reuse candidates and recommended placement** from Step 4.5 (surface *first* — user must see what was already in the codebase before approving)
- criteria derived from Testing strategy (easy to miss)
- spec items deliberately not turned into criteria, with one-line reason
- (supersede) the diff against previous spec

Wait for explicit approval. Edits acceptable (rename criterion id, sharpen pass_when, move to `out_of_scope`):

- surface tweaks → apply inline
- material design changes (new task, removed file, retyped criterion) → re-invoke `spec-writer` with amended request

After approval:

1. Write `{$META.specs_path}{slug}.md`. For supersede, overwrites old file; bump `revised: YYYY-MM-DD` (don't change `created`). For chain, write **one `<sub-slug>.md` per sub-spec** in chain order; parent slug not written.
2. Write `{$META.specs_path}{slug}.acceptance.md` per `plugins/mol/rules/evaluator-protocol.md`. Chain → one acceptance per sub-spec.
3. Update `{$META.specs_path}INDEX.md`:

   ```markdown
   # Specs

   - [{slug}]({slug}.md) — <one-line summary> [approved]
   ```

   Supersede/refine → update entry in place. `/mol:impl` removes entry on finish. Chain → **one entry per sub-spec** in chain order (no parent entry).

If user defers approval (*"let me think about it"*), write spec with `status: draft` and skip `acceptance.md`. Re-invoking `/mol:spec` on same slug resumes from Step 5.

### 7. Report

- spec path + acceptance path (per sub-spec for chain)
- task count (e.g. *"7 tasks; 0 completed"*) per spec
- criteria count by `type`, per spec
- one-line note flagging criteria needing runtime evaluator (any `type` other than `code` / `runtime`). Example: *"3 ui_runtime criteria — invoke `/mol:web <slug>` after `/mol:impl`; `/mol:impl` parks at `status: code-complete` until those flip to `verified`."*
- supersede flows: short diff (changed / unchecked / removed / added)
- chain flows: next-step pointer *"start with `/mol:impl <base>-01-<phase>`; auto-creates `feat/<base>` and stage-commits each sub-spec on completion."*

End with one-line user-facing summary.

## Lifecycle

- `draft` — written but user deferred approval; acceptance.md not yet written. Re-run `/mol:spec <slug>` to resume.
- `approved` — user signed off both files. Ready to impl. Every criterion starts at `status: pending`.
- `in-progress` — `/mol:impl` started; Tasks ticked, criteria flipped to `verified` / `failed` as tests run.
- `code-complete` — Step 7 finished code work; every `code` / `runtime` criterion is `verified`, but ≥1 runtime-evaluator-typed criterion (`ui_runtime` / `scientific` / `performance` / `docs`) still `pending`. Files stay so evaluators can read them. User runs evaluator; each updates criterion `status` per `plugins/mol/rules/evaluator-protocol.md`. Re-run `/mol:impl <slug>` to advance to `done`.
- `done` — every criterion `verified` and tests green. `/mol:impl` deletes spec, INDEX entry, acceptance.md.

## Why drafting is delegated

Drafting is one long generative pass; running in `spec-writer`'s subagent context keeps parent free for the conversation. User-interaction parts (triage, approval, persistence, INDEX) stay here.

See `plugins/mol/rules/agent-design.md` for full producer / reviewer / drafter classification.

## Bilingual

If user argument is Chinese, `spec-writer` produces spec body in Chinese; frontmatter keys, INDEX entry, and verb-prefix of each Tasks line stay English so `/mol:impl` and downstream tooling parse deterministically.
