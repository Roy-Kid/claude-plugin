---
description: Validate project architecture — module boundaries, dependency graph, layer rules, pattern compliance. Read-only.
argument-hint: "[path or module]"
---

# /mol:arch — Architecture Validation

Delegate to the `architect` agent with the user's argument (or "all files"
if none given).

The agent will:

1. Read CLAUDE.md and parse `mol_project:` frontmatter (`$META`).
2. Read the architecture rules from the section named by
   `$META.arch.rules_section`.
3. Discover current files via glob, filtered by `$META.language`.
4. Run the checks appropriate to `$META.arch.style`:
   - `layered` — module X depends only on modules allowed by the layer graph.
   - `crate-graph` — workspace `Cargo.toml` dependency edges match the
     documented graph.
   - `backend-pillars` — backend root files are shared infrastructure;
     backend-specific types do not leak into facade headers.
   - `package-tree` — each package's public surface is what CLAUDE.md
     claims; no cross-package reach-through.
   - `monorepo` — workspace references use the canonical alias form;
     source-alias rules respected.
5. Report findings in `<emoji> file:line — message` format (F1), with
   emoji from 🚨 / 🔴 / 🟡 / 🟢 (Critical / High / Medium / Low).
