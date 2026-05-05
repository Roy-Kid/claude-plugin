---
description: Literature and reference-implementation review before coding a domain-critical feature. Gated on mol_project.science.required.
argument-hint: "<method or algorithm name>"
---

# /mol:litrev — Literature Review

Read CLAUDE.md. Parse `mol_project:` (`$META`). If
`$META.science.required` is `false`, print a short message stating this
project opts out of literature review and stop — the caller should not gate
on this skill.

Delegate to the `scientist` agent for domain expertise. Extend with:

## Procedure

1. **Literature search.** Use WebSearch for foundational papers, reviews,
   and published corrections. Prefer primary sources.
2. **Document governing equations.** Every term defined, in project units.
   Derive these from CLAUDE.md (ATV uses eV/Å/fs/amu; molpy follows
   `core/units.py`; molnex uses PyTorch-native units; etc.). If units are
   not documented, flag it as an open question.
3. **Reference implementations.** Search matching code in LAMMPS, VASP, ASE,
   RDKit, PyTorch, Jax-MD, packmol, or whichever upstream is canonical for
   the feature. Pull their API patterns and validation cases.
4. **Validation targets.** Analytical solutions, published benchmarks,
   conservation laws, limiting cases.

## Output

- **Credibility assessment.** `YES` / `NO` / `CONDITIONAL` (with the
  condition).
- **Key references.** Numbered list with DOI or arXiv where available.
- **Implementation guidance.** Algorithm variant, numerical details,
  pitfalls, expected edge cases.
- **Validation plan.** Ordered tests with expected values and tolerances.
- **Open questions.** What the user must decide before `/mol:spec`.
