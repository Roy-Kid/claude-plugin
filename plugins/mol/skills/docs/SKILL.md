---
description: Documentation audit and writing — docstrings in the project's documented style plus narrative tutorials. Writes docs and code comments.
argument-hint: "[path, spec slug, or feature name]"
---

# /mol:docs — Documentation

Read CLAUDE.md. Parse `mol_project:` (`$META`). Pick a mode from the
caller's intent — do not mix the two.

## Mode A — Docstring / API audit

Delegate to the `documenter` agent in audit mode. It loads the style rules
for `$META.doc.style`:

- `google` — Google-style Python docstrings: short summary, blank line,
  Args/Returns/Raises sections, types in signatures not in prose.
- `rustdoc` — Rustdoc with `# Examples`, `# Errors`, `# Panics` sections;
  `///` for items, `//!` for modules; math via `\\[...\\]`.
- `jsdoc-tiered` — three tiers per symbol kind:
  - **Full** — class and top-level function declarations.
  - **Brief** — one-line for internal helpers.
  - **Inline** — next to non-obvious expressions only.
- `doxygen` — `@brief`, `@param` with units, `@return`, `@file` headers.

The audit checks every public symbol for:
1. Presence of the required tier.
2. Unit annotations on every physical quantity (if
   `$META.science.required`).
3. Matches current implementation (no stale references).
4. No dangling cross-references.

Output: `<emoji> file:line — message` with `Fix:` lines (🚨 / 🔴 / 🟡 /
🟢 for Critical / High / Medium / Low).

## Mode B — Narrative tutorial

Delegate to the `documenter` agent in tutorial mode. Before delegating,
collect:

1. The relevant spec in `$META.specs_path`, if one exists.
2. The implementation files that define current behavior.
3. Any existing user-facing docs on the same topic.
4. A real example program that the tutorial will reference.

If a file, target, plot, or API does not exist, do not mention it as if it
does. Either correct the tutorial or create the missing artifact first.

Tutorial rules:
- Lead with the problem and intuition; introduce notation after the reader
  knows why it matters.
- Prose over lists. Bullets only for genuine enumeration.
- Every symbol defined before use.
- Reference a concrete example the reader can build and run.
- Math uses standard Markdown LaTeX: `$...$` inline, `$$...$$` display. No
  Doxygen `\f` markers in Markdown.

**Science content (if `$META.science.required` or the topic is
scientific):** the agent must produce two parts of equal weight —
*Part 1* a self-contained textbook derivation that a newcomer with
only undergraduate background can verify *without reading the code*,
and *Part 2* the project / code mapping. Part 1 is a verification
artifact; it must be **completely equivalent** to the implementation
(equations, signs, units, conventions). Discrepancies between Part 1
and the code are findings, not stylistic notes — fix the side that's
wrong. See `agents/documenter.md` § "Science content: mandatory
two-part structure" and § "Equivalence rule" for the full contract.

End with a one-line summary: files written, example referenced, TODOs.
For science tutorials, also report the equivalence-rule check
performed (which equations were compared to which code locations).
