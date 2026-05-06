---
description: Open a free-form design / improvement / scientific-insight discussion with the agent. The skill drives the conversation toward convergence; on convergence it hands the agreed direction to /mol:spec to produce a binding spec, otherwise it discards the thread without leaving artifacts behind. Read-only — never writes code or specs directly. Distinct from /mol:spec (which assumes the requirement is already clear) and /mol:note (which captures decided rules, not exploratory threads). Supports Chinese and English. 通过这个 skill，和 agent 讨论代码设计、改进意见、科学洞见之类的问题；意见收敛之后形成 spec，否则 discard。
argument-hint: "<topic or question>"
---

# /mol:discuss — Design Discussion

Read CLAUDE.md. Parse `mol_project:` (`$META`).

A free-form conversational skill for working through design
trade-offs, improvement ideas, or scientific insights *before*
they harden into a spec. The skill is explicit about its two
exit doors — **converge** (hand off to `/mol:spec`) or
**discard** (leave no trace) — so a discussion never silently
turns into half-built code or a forgotten orphan file. Distinct
from `/mol:spec` (which assumes the requirement is already
clear) and `/mol:note` (which captures *decided* rules, not
exploratory threads).

## Procedure

### 1. Frame the topic

Restate the topic in one sentence as you understand it, in the
user's language. List the relevant code surface — files,
functions, prior commits, related specs under `.claude/specs/`,
related notes under `.agent/` — that you read before responding.
This anchors the discussion in concrete artifacts and makes the
"is this about X or Y?" misalignment visible early.

If the topic is genuinely vague, ask one targeted clarifying
question and stop. Do not start exploring on a wrong premise.

### 2. Drive toward convergence

Each turn, end with an explicit pulse on where the discussion
stands, in this fixed shape:

```
Convergence pulse
- Agreed: <bullets of what is now settled>
- Open: <bullets of what is still in dispute or undefined>
- My read: converging | diverging | stuck
```

Convergence signals (any one is enough): the user says
"yes / 同意 / let's do it"; the open list is empty; two
consecutive turns produce no new "Open" items.

Divergence signals: the open list grows turn-over-turn; the
user keeps reframing the topic; both of you keep proposing
mutually incompatible alternatives without reduction.

Hard cap: **8 turns**. At turn 8, force a verdict in Step 3 or
Step 4 — no skill should silently consume an unbounded
conversation.

### 3. Converge → hand off to `/mol:spec`

When the pulse reads "converging" *and* the open list is
empty (or the user explicitly accepts the remaining opens as
out-of-scope), package the conclusion:

- one-paragraph requirement, in the user's language, that a
  fresh `/mol:spec` invocation could consume verbatim
- a short "Context from discussion" block listing the
  alternatives considered and the reason this one won — this
  becomes the spec's *Why* later
- the relevant file paths surfaced in Step 1

Tell the user: *"converged. To produce the binding spec,
run `/mol:spec <one-paragraph requirement>`. Paste the
Context block as additional context if you want the
trade-off rationale captured."* Do not invoke `/mol:spec`
yourself — the user controls when a spec gets created.

### 4. Discard cleanly

When the pulse reads "diverging" or "stuck" for two
consecutive turns, when the user changes their mind about
the topic, when the question dissolves into a non-question,
or when the 8-turn cap fires without convergence:

- say so explicitly and name the reason in one sentence
  (e.g. *"discarded — the question turned out to depend on a
  decision in `auth/` that hasn't been made yet"*)
- write nothing: no notes, no draft spec, no `.agent/` entry,
  no `.claude/specs/` entry
- if a *stable rule* fell out of the discussion (rare but
  possible), suggest the user run `/mol:note` separately —
  do not promote it yourself

Discarding is a first-class outcome. Not-converging is fine;
shipping a half-decided spec is not.

## Output format

- One framing sentence + the surfaced code-surface bullets
  (Step 1).
- Per-turn `Convergence pulse` block.
- On convergence: the one-paragraph requirement + Context
  block + handoff instruction.
- On discard: the one-sentence reason; nothing else.

End with a one-line summary (F2):

```
/mol:discuss <topic>: converged → /mol:spec <one-line requirement>
```

or

```
/mol:discuss <topic>: discarded (<reason>)
```

## Guardrails

- **Read-only on code.** This skill never edits files. Even
  on convergence the spec is written by `/mol:spec`, not by
  this skill.
- **Do not promote** outputs into `.claude/specs/` or
  `.agent/notes.md` directly. Convergence hands off; discard
  leaves no trace. Both paths refuse to write here.
- **Do not silently merge with `/mol:note`.** If a stable
  rule emerges, surface it as a `/mol:note` suggestion —
  the user runs it.
- **Do not auto-loop or auto-invoke `/mol:spec`.** The user
  decides when a discussion becomes a spec; this skill only
  prepares the handoff payload.
- **Hard 8-turn cap.** A discussion that hasn't converged in
  8 turns has a deeper problem than another turn can fix.
