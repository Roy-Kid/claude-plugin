# Evaluator Protocol

Specifies the artifact and verdict shapes that connect the planner
(`/mol:spec`), the generator (`/mol:impl`), and the runtime
evaluators (`/mol:web`, future `/mol:bench` / `/mol:numeric` / …).
The protocol exists so the **driving flow** stays domain-neutral
while runtime evaluators can be added per-domain without re-shaping
the rest of the harness.

This is a **protocol document**, not an enforcement mechanism.
There is no central registry; `/mol:review` does not auto-discover
evaluators. The protocol is a *convention* that orchestrators
(manual user, a future `/mol:loop` skill, or a third-party system
like the user's Symphony orchestrator) can rely on.

## Three artifacts

```
.claude/specs/<slug>.md              ← the spec       (planner output)
.claude/specs/<slug>.acceptance.md   ← the contract   (planner output, sibling)
.claude/specs/<slug>.artifacts/      ← evaluator artifacts (screenshots, logs)
```

- **spec.md** — design, files, tasks, testing strategy. Produced
  by `/mol:spec`. Its `status:` field gates `/mol:impl`.
- **acceptance.md** — the binding "done" contract, also produced
  by `/mol:spec` (validation + acceptance negotiation are folded
  into the same skill). Structured frontmatter so external
  orchestrators can parse it without LLM intervention.
- **artifacts/** — scratch directory for runtime-evaluator output
  (screenshots, console logs, network dumps). Per-criterion
  subdirectories. Cleaned up manually when the spec is closed.

## acceptance.md format

```markdown
---
spec: <slug>                       # must match spec filename without .md
created: YYYY-MM-DD
criteria:
  - id: ac-001
    summary: "<short title, ≤80 chars>"
    type: code | runtime | ui_runtime | scientific | performance | docs
    evaluator_hint: <optional, e.g. "mol:web">
    pass_when: "<single observable condition in plain prose>"
out_of_scope:
  - "<bullet>"
---

# Acceptance — <slug>

<one paragraph: what "done" means for this spec, in human prose>

## AC-001 — <title>

<expanded reasoning, fixtures, edge cases, links to test files>

(repeat per criterion)
```

### Field semantics

- **`id`** — kebab-case, prefixed `ac-`. Stable identifier within
  a spec lifetime; ids do not renumber on minor refinement. A
  supersede / refine flow that rewrites the spec body restarts
  numbering at `ac-001` because the spec itself is a new design.
- **`type`** — partitions criteria by which evaluator can verify
  them:
  - `code` — static analysis suffices. `/mol:review`'s static
    reviewers (architect, optimizer, documenter, undergrad, pm,
    janitor) handle these.
  - `runtime` — requires executing code (test suite, CLI
    invocation, backend smoke).
  - `ui_runtime` — requires driving a running frontend (Playwright
    via `/mol:web`).
  - `scientific` — requires numerical comparison against reference
    data or analytical solution.
  - `performance` — requires a benchmark with a quantified
    threshold.
  - `docs` — requires inspecting generated documentation
    (cross-references, examples that compile, link integrity).
- **`evaluator_hint`** — soft preference, not a hard binding. The
  user (or an orchestrator) MAY pick a different evaluator if
  available. `mol` does not enforce.
- **`pass_when`** — must be observable: a query you can answer
  yes/no by inspecting the impl, the running system, or an
  artifact. *"Code is well-designed"* is **not** observable;
  *"Reading `Workspace` from a JSON dir produced by v0.4 succeeds
  without errors"* **is**.

### Acceptance lifecycle

`<slug>.acceptance.md` is written by `/mol:spec` at the same time
as `<slug>.md`. The two files always exist together. `/mol:impl`
gates on **both** the spec status (`approved`) **and** the
existence of the acceptance file.

If a supersede / refine flow rewrites the spec body, `/mol:spec`
also regenerates `acceptance.md` from scratch — criteria
negotiated against the old design are not portable.

When `/mol:impl` finishes a spec, it deletes `<slug>.md`,
`<slug>.acceptance.md`, and the INDEX entry together. The
`artifacts/` directory is left for the user to clean (since impl
does not own it).

## Evaluator skill contract

A skill that provides a runtime evaluator MUST:

### Accept

A single argument shaped as `<spec-slug> [criterion-id]`:

- without `criterion-id`: evaluate every criterion the skill can
  handle (matched by `type`).
- with `criterion-id`: evaluate only the named criterion.

The skill reads:

- `.claude/specs/<slug>.md`
- `.claude/specs/<slug>.acceptance.md`
- the implementation under the project root

### Self-skip when prerequisites are absent

If the skill's prerequisites are missing (e.g. no
browser-automation MCP for `/mol:web`, no benchmark harness
configured for a future `/mol:bench`), the skill MUST exit cleanly
with a message naming what is missing — not crash and not pretend
to verify. Detection happens up front, before any acceptance file
is read.

### Return

A markdown block with this shape:

```markdown
## /<plugin>:<skill> — <slug>

| Criterion | Verdict | Evidence |
|-----------|---------|----------|
| ac-001    | ✅ pass | <one line + path to artifact> |
| ac-003    | ❌ fail | <one line + path to artifact> |
| ac-005    | ⏭ skip | <reason — wrong type for this skill> |

### Artifacts

- `<path>` — <what it is>
```

Verdicts MUST be exactly one of `✅ pass`, `❌ fail`, `⏭ skip`. The
evaluator MUST NOT mutate `spec.md` or `acceptance.md`; only
`/mol:spec` writes those files.

### Naming convention

The skill SHOULD be `mol:<axis>` so orchestrators can find it by
convention: `/mol:web` for `ui_runtime`, future `/mol:bench` for
`performance`, future `/mol:numeric` for `scientific`. Each
self-skips when its target type is not present in `acceptance.md`.

## Known evaluator skills

| Skill      | Handles `type`     | Prerequisite                                              |
|------------|--------------------|-----------------------------------------------------------|
| `mol:web`  | `ui_runtime`       | A browser-automation MCP (Playwright MCP, claude-in-chrome, …) installed by the user |

Add to this table when a new evaluator skill lands inside `mol`.

## What this protocol is *not*

- **Not an auto-dispatcher.** `/mol:review` does not invoke
  evaluators. It does the static-review job (always available)
  and *suggests* which runtime evaluator to invoke next based on
  `type` fields in `acceptance.md`. A human or external
  orchestrator decides whether to run them.
- **Not a model contract.** Evaluator skills are free to use any
  model, MCP server, or external tool. The contract is over the
  *artifact shape* on input and the *markdown shape* on output.
- **Not a CI integration.** This protocol describes interactive
  evaluation during development. CI hookup is the orchestrator's
  problem.
- **Not a plugin contract.** Runtime evaluators live as `/mol:*`
  skills inside the `mol` plugin (not as separate plugins).
  Plugins are reserved for capability boundaries that justify
  independent versioning and dependency scope (e.g. `mol-agent`
  owns harness lifecycle; `mol-plugin` owns marketplace
  maintenance). Evaluators are just procedures that consume a
  shared artifact and emit a shared verdict shape — they do not
  warrant their own plugin scope.
