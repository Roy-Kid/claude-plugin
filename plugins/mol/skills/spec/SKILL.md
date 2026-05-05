---
description: Convert a natural-language requirement into a structured technical spec under .claude/specs/, with a checkbox-tracked Tasks section, then negotiate the binding `<slug>.acceptance.md` contract that defines "done". Self-validates spec quality (sections / atomic tasks / RED-before-GREEN / Files↔Tasks cross-reference) before showing the user. Detects conflicts with existing specs and updates the old spec body + regenerates acceptance criteria when superseded or refined. Writes specs and acceptance files (and amends old ones during conflict resolution). Supports Chinese and English.
argument-hint: "<feature description>"
---

# /mol:spec — Specification Generator

Read CLAUDE.md. Parse `mol_project:` (`$META`). If missing, emit the
adoption hint and stop.

Resolve the specs path:

1. If `$META.specs_path` is set, use it (canonical default
   `.claude/specs/`).
2. Otherwise, default to `.claude/specs/`.

Create the directory if missing.

## Specs are active runtime artifacts

Specs live under `.claude/` because they are **alive**: `/mol:impl`
ticks their checkboxes off as work progresses, and the spec file is
**deleted** when every task is complete. A spec is not documentation
— it is a working contract between this conversation and the next
one.

This skill produces **two** files per spec:

- `<slug>.md` — the design (Summary / Design / Files / Tasks /
  Testing / Out of scope).
- `<slug>.acceptance.md` — the binding "done" contract: structured
  observable criteria that `/mol:impl` and any runtime evaluator
  (e.g. `/mol:web` for UI) verify against. Format defined in
  `plugins/mol/docs/evaluator-protocol.md`.

The pair is produced together. `/mol:impl` refuses to start without
both files in place — this is the planner→generator handoff
formalized.

Where specs do **not** belong:

- `docs/` — public documentation. Never put a spec there.
- `.agent/` — passive context (notes, decisions, debt). Specs are
  not passive; they are active runtime tasks.

The repository should hold one live spec per design decision. When a
spec finishes, the decisions worth keeping have already been written
into code, tests, and (if non-obvious) `.agent/notes.md`.

## Procedure

### 1. Parse the request

What capability is being added? Which layer / package / crate is
affected? State this in one sentence before going further.

### 2. Search domain basis (if applicable)

If physics is involved AND `$META.science.required` is `true`,
delegate to the `scientist` agent for equations, references, and
validation targets.

### 3. Conflict check (MANDATORY before any write)

Read every existing spec under `$META.specs_path`. For the new
request, classify against each existing spec:

- **Duplicate.** The existing spec already covers this scope → tell
  the user, don't add a sibling.

- **Supersede / refine.** The new request changes, expands, or fixes
  the design of an existing spec (the user found a gap, a better
  approach, a missing edge case) → **update the old spec in place**,
  do not create a sibling. Specifically:
  1. Update the affected sections of the old spec body (Summary,
     Design, Files, Testing strategy as relevant).
  2. **Recompute the Tasks checklist**:
     - Items still valid AND already done → keep `[x]`.
     - Items still valid AND not yet done → keep `[ ]`.
     - Items invalidated by the new design → **remove** them
       (don't just leave them stranded).
     - Items where prior work needs redoing → restore to `[ ]` and
       add a `(rework: <why>)` note on the line.
     - Genuinely new tasks → add as `[ ]` at the bottom.
  3. Bump the frontmatter `revised: YYYY-MM-DD`. Do not change
     `created`.
  4. **Regenerate `{slug}.acceptance.md`** from scratch against the
     new Tasks list (Step 7 below). The previous acceptance was
     negotiated against the old design and is stale; criteria ids
     restart at `ac-001` for the new contract.
  5. Tell the user explicitly which old items were unchecked, which
     were removed, and which were added. The user must see the
     diff before approving.

- **Independent.** Different scope from any existing spec → safe to
  create a new file.

Never create a sibling spec for a supersede / refine. If you find
yourself wanting to create `morse-bond-v2.md` next to
`morse-bond.md`, that's the wrong move — update `morse-bond.md`.

### 4. Scan interaction points

Glob the project root for files relevant to `$META.language`. Identify
the closest existing pattern. Flag any new public API, data-model
change, or cross-layer dependency.

### 5. Draft the spec body

Sections every spec must have:

- **Summary** — 1 paragraph, plain prose.
- **Domain basis** — equations, references with DOI/arXiv, units (if
  `$META.science.required`).
- **Design** — entities touched, new symbols, lifecycle / ownership.
- **Files to create or modify** — bulleted file paths.
- **Tasks** — checkbox-tracked subtasks (see below). **This section is
  mandatory.**
- **Testing strategy** — happy path, edge cases, domain validation.
- **Out of scope** — what this spec does not do.

### 6. Tasks (the implementation tracker)

This is the heart of the spec. `/mol:impl` ticks these as it
progresses; when every task is `[x]` and the test/check suites pass,
`/mol:impl` deletes the spec.

Format:

```markdown
## Tasks

- [ ] Write failing tests for <component> (<path/to/test_file>)
- [ ] Implement <symbol> in <path>
- [ ] Add docstring per <doc.style> with units
- [ ] Verify against <validation case>
- [ ] Run full check + test suite
```

Rules for the Tasks section:

- Each item is **concrete, atomic, and checkable**. No "implement
  everything"; no aspirational items.
- Aim for 4–10 tasks. If a spec needs more than ~12, split it into
  two specs.
- Keep verbs first ("Write…", "Implement…", "Verify…") so the list
  reads as a procedure.
- Write the tests-first task before any implementation task — RED
  before GREEN (W2).
- The last task is usually "Run full check + test suite" so the
  verification step is itself tracked.

### 7. Self-validate the spec body (internal quality gate)

Before showing anything to the user, walk the drafted body with
this checklist. Every failure is a blocker for *this draft attempt*
— silently revise and re-check up to 3 times. If the third revision
still fails, surface the failed items to the user with a question:
*"I cannot satisfy <items>; do you want to relax X, or refine the
request?"*

Required sections (MUST exist and be non-empty):

- [ ] **Summary** — one paragraph; states the user-visible outcome.
- [ ] **Design** — entities, lifecycle, ownership; not just a
  restatement of Summary.
- [ ] **Files to create or modify** — every path is concrete (no
  glob like "various test files"). Paths exist on disk OR are
  marked as new.
- [ ] **Tasks** — present and non-empty.
- [ ] **Testing strategy** — happy path, edge cases, and (if
  `$META.science.required`) domain validation are explicitly
  enumerated.
- [ ] **Out of scope** — present, even if "none". An empty section
  is a smell.

Required for Tasks (each item):

- [ ] Verb-first ("Write…", "Implement…", "Verify…").
- [ ] Concrete and atomic (one observable change).
- [ ] References a file path or symbol where the work lands.
- [ ] **RED-before-GREEN ordering** — every "Write failing tests
  for X" task precedes its corresponding "Implement X" task.
- [ ] Total count is 4–12.

Required for cross-references:

- [ ] Every file in **Files to create or modify** appears in at
  least one Tasks item.
- [ ] Domain references (DOI / arXiv) present iff
  `$META.science.required` and the spec declares physics.

### 8. Negotiate acceptance criteria (the binding "done" contract)

Walk the validated spec section by section and propose
`<slug>.acceptance.md` — the observable projection of the spec.
Every Task and every behavior in Testing strategy that matters
for "done" gets a criterion. Format defined in
`plugins/mol/docs/evaluator-protocol.md`.

For each candidate criterion, decide:

#### `id`

`ac-001`, `ac-002`, … in order of appearance in the spec. Stable
identifier: once written, ids do not renumber on minor refinement
(supersede / refine restarts at `ac-001` because the spec body
itself was rewritten).

#### `summary`

≤80 chars, imperative or stative. *"Workspace.load reads v0.4
JSON layouts"* not *"Backwards compatibility"*.

#### `type`

Pick the **narrowest type that suffices**:

| Spec signal                                       | Type            |
|---------------------------------------------------|-----------------|
| pure logic, no side effects                       | `code`          |
| invokes CLI / FastAPI / shell / subprocess        | `runtime`       |
| user-visible UI behavior, click / route / render  | `ui_runtime`    |
| numerical match against reference data            | `scientific`    |
| time / memory / throughput threshold              | `performance`   |
| docstring / generated docs / link integrity       | `docs`          |

If a criterion crosses categories, split it into multiple criteria.
A single Task may spawn 2–3 criteria.

#### `evaluator_hint` (optional)

Soft preference. For `ui_runtime`, default to `mol:web` if the
project has a frontend. Never hard-bind — the user / orchestrator
may pick differently.

#### `pass_when`

A single observable condition. Test it:

- Could a third-party evaluator answer yes/no without rereading
  the spec? If no, rewrite.
- Does it name a fixture, file, threshold, or visible state? If
  no, rewrite.
- Bad: *"Implementation is correct."*
- Good: *"`pytest tests/workspace/test_load.py::test_v04_layout`
  passes against the fixture in `tests/fixtures/ws_v04/`."*

### 9. Show, confirm, persist

Show the user **both** files in the same turn:

- the spec body
- the proposed `acceptance.md` (criteria table: id, type,
  pass_when truncated)

Call out:

- criteria you derived from Testing strategy (easy to miss)
- spec items you deliberately did **not** turn into criteria, with
  one-line reason

Wait for explicit approval. Edits — adding/removing/retyping
criteria, tightening `pass_when`, moving items to `out_of_scope` —
are all acceptable. Apply, re-show, re-confirm.

After approval:

1. Write `{$META.specs_path}{slug}.md` (kebab-case slug:
   `morse-bond`, `nose-hoover`, `amber-prmtop-reader`).

   ```markdown
   ---
   title: <spec title>
   status: approved
   created: YYYY-MM-DD
   # revised: YYYY-MM-DD   # added only on supersede/refine
   ---
   ```

2. Write `{$META.specs_path}{slug}.acceptance.md` per the schema
   in `plugins/mol/docs/evaluator-protocol.md`.

3. Update `{$META.specs_path}INDEX.md`:

   ```markdown
   # Specs

   - [{slug}]({slug}.md) — <one-line summary> [approved]
   ```

   On supersede/refine, update the entry in place. When `/mol:impl`
   finishes a spec, it removes the entry.

If the user asks to defer approval (*"let me think about it"*),
write the spec with `status: draft` and skip writing
`acceptance.md` — when they come back, re-invoke `/mol:spec` on
the same slug to resume from this step.

### 10. Report

Print:

- spec path + acceptance path
- task count (e.g. *"7 tasks; 0 completed"*)
- criteria count, broken down by `type`
- one-line note flagging which criteria need a runtime evaluator
  (any `type` other than `code` / `docs`). Example:
  *"3 ui_runtime criteria — invoke `/mol:web <slug>` after
  `/mol:impl` finishes."*
- for supersede flows: a short diff (what changed, what was
  unchecked, what was removed, what was added)

End with a one-line user-facing summary.

## Lifecycle

- `draft` — written but the user deferred approval; acceptance.md
  not yet written. Re-run `/mol:spec <slug>` to resume.
- `approved` — user signed off both spec body and `<slug>.acceptance.md`.
  This is the ready-to-impl state.
- `in-progress` — `/mol:impl` started; tasks being ticked off.
- `done` — every task is `[x]`; tests green. `/mol:impl` deletes
  the spec file, the INDEX entry, and `<slug>.acceptance.md` on
  its way out.

The deleted state is intentional. Once a spec is `done`, the
information that mattered has been encoded into code, tests, and
(when non-obvious) `.agent/notes.md`. Keeping a paper trail of done
specs would clutter the repo without adding signal — that is what
git history is for.

## Bilingual

If the user argument is in Chinese, produce the spec body in Chinese.
Keep the frontmatter keys, INDEX entry, and the verb-prefix of each
Tasks line in English so `/mol:impl` and downstream tooling can parse
the checklist deterministically.
