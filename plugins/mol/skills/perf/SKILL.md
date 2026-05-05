---
description: Performance review — delegates to the optimizer agent, which detects which anti-pattern catalogs apply (numpy / pytorch / cuda / simd-xsimd / wasm-bridge / subprocess / async-io / web-render) per file by inspecting imports, headers, and file extensions, then scans hot paths for catalog-specific findings. Read-only.
argument-hint: "[path or module]"
---

# /mol:perf — Performance Review

Delegate to the `optimizer` agent with the user's argument (or all
source files if none given).

There is **no project-level `perf.focus` enum**. The `optimizer` agent
detects which anti-pattern catalogs apply per file under review (a
multi-facet project can have several), per the table at the top of
`optimizer.md`:

| Catalog       | File / signal that turns it on |
|---------------|--------------------------------|
| `numpy`       | `import numpy` / `.npy` / numpy in deps |
| `pytorch`     | `import torch` / torch in deps |
| `simd-xsimd`  | `<xsimd/...>` in C++ source |
| `cuda`        | `.cu` / `.cuh` / `__global__` / `<<<...>>>` |
| `wasm-bridge` | `wasm-bindgen` / wasm-pack / cdylib-wasm target |
| `subprocess`  | `subprocess.Popen` / job scheduler / `sqlite3` |
| `async-io`    | `async def` + `await` over network / DB / disk |
| `web-render`  | React / Vue / Svelte source |

Multiple catalogs may apply per file. If none match, the agent runs a
generic scan (allocation in loops, unbounded growth, missing
benchmarks).

The agent will:

1. Determine which catalogs apply to each file under review.
2. Identify the hot path(s) — a step loop, a forward pass, a render
   frame, a request handler, a submit/poll cycle.
3. Grep for the catalog-specific anti-patterns plus any
   project-specific patterns documented in CLAUDE.md.
4. Check that benchmarks exist for the hot path. If not, flag as a
   gap.
5. Report findings in `<emoji> file:line — message` format (🚨 / 🔴 /
   🟡 / 🟢) tagged with the catalog that raised them, plus an
   `Impact:` estimate and a `Fix:` recommendation.
