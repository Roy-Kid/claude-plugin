---
description: Diagnose build, test, or runtime failures with structured root-cause analysis. Read-only — never writes code. Use /mol:fix to actually patch.
argument-hint: "<error message or symptom>"
---

# /mol:debug — Diagnosis Only

Read CLAUDE.md. Parse `mol_project:` (`$META`).

This skill **never** edits files. It produces a diagnosis only. To patch,
use `/mol:fix`.

## Procedure

### 1. Classify the failure

- **Build failure** — compile error, link error, missing dependency,
  configure / CMake error, `cargo build` error, rsbuild error.
- **Test failure** — assertion failure, crash, timeout, snapshot mismatch.
- **Runtime failure** — null / dangling pointer, illegal memory access,
  kernel launch failure, NaN / inf, deadlock, unexpected panic.

### 2. Gather

Run the relevant command from `$META.build` and capture the tail:

```
$META.build.check        # for format / lint failures
$META.build.test         # for the full suite
$META.build.test_single  # with the specific test path
```

For runtime failures, collect the stack trace, device state (if CUDA
project), and any relevant logs the project documents in CLAUDE.md.

### 3. Diagnosis by type

**Build failure** — Check that the changed file's includes / imports
respect the layer rules under `$META.arch.rules_section`. Check for
recently added symbols that haven't been registered in the project's
build manifest (CMakeLists, Cargo.toml, package.json, pyproject.toml).

**Test failure** — Read the failing test and the symbol under test.
Verify the test categories match what the project documents. Check for
fixture / seed issues.

**Runtime failure** — Inspect the failing file. For CUDA code (`.cu`
files / `__global__` / `<<<...>>>`), check kernel launch
configuration, device-pointer lifetimes, stream synchronization. For
numpy / pytorch code, check tensor shapes and device placement. For
subprocess-heavy code (`subprocess.Popen` / job scheduler), check
process lifecycle, resource leaks, polling races. For WASM bridges
(`wasm-bindgen` / cdylib-wasm), check that host/guest pointers are
not captured across frames. For async I/O (`async def` + `await`),
check for synchronous calls inside the event loop and missing
`await` on coroutines.

**NaN / inf** — Check division by zero in any distance-based kernel,
unit conversion mismatches, uninitialized state.

### 4. Report

- **Root cause.** One paragraph, precise.
- **Fix recommendation.** What to change — not the change itself.
- **Preventive measure.** What test would catch this in the future.

No code edits. Hand off to `/mol:fix` for implementation.
