# SpaDES Iterative Development — Error Reference

When `source("global.R")` or a step in the three-step execution pattern fails, use this guide to diagnose and fix the error. Fix one error at a time; re-run from the failing step.

Reference: See `workflows/development-workflow.md` for the full step-by-step workflow.

---

## Iteration Protocol

1. **Fix one error at a time** — resolve the first error before looking at the next
2. **Read the module file before changing anything** — error messages name the object or event; read the relevant `defineModule()` block or event handler before guessing a fix
3. **If the fix is in a module `.R` file only** — re-run from `simInit2(out)` directly; no need to re-run `setupProject()`
4. **If the fix requires changing `out`** (adding a module, changing a parameter, changing paths in `global.R`) — re-run `setupProject()` and continue from there

---

## `setupProject()` Errors

| Error pattern | Likely cause | Fix |
|---|---|---|
| `curl` error / `cannot open URL` before `setupProject()` runs | Network unavailable, or raw GitHub URL for `getOrUpdatePkg` has moved | Check network; fall back to `install.packages("Require")` and comment out the `source(...)` bootstrap line |
| `Error in Require(...)` / version conflict | Package version mismatch | Check `getOrUpdatePkg()` version pins; update or relax the minimum version |
| GitHub 404 / rate limit | Bad module reference or unauthenticated | Check `"org/repo@branch"` string is correct; set `GITHUB_PAT` env var |
| `Error: path does not exist` | `modulePath` or `inputPath` directory missing | Create the directory: `dir.create("modules", recursive = TRUE)` |

---

## `simInit2()` Errors

When `spades.allowInitDuringSimInit = TRUE`, `init` events run during `simInit2()`. Errors that look like runtime errors may surface here — check the traceback for the event name.

| Error pattern | Likely cause | Fix |
|---|---|---|
| `object 'X' not found in sim` | Missing `inputObject` | In `defineModule()`, check `sourceModuleSelect` for `X` — if set, that module is missing from `modules` list; if `NA`, supply `X` manually in `global.R` |
| `"unused.*parameter"` / `"not defined in.*module"` | Wrong parameter name in `params` | Read the module's `defineModule()` → `parameters` block for the correct name and spelling |
| `"cannot import Python 'ws3'"` or `reticulate` import error | Python env missing ws3, or wrong Python env | Run `pip install ws3`; check `reticulate::py_config()` points to the correct Python env |
| `studyArea` / `rasterToMatch` CRS error | CRS mismatch between study area and downloaded data | Re-project in the `studyArea = {}` block before passing to `setupProject()` |

---

## `spades()` Errors

| Error pattern | Likely cause | Fix |
|---|---|---|
| Traceback names a specific module + event | Bug in that module's event handler | Read the named event handler function; fix the logic or file an issue upstream |
| `NA` or empty `sim$X` immediately after `simInit2` | The producing module's `init` event returned bad output | Read that module's `init` handler; verify `inputObjects` were valid at init time |
| `NA` or empty `sim$X` mid-run (during `spades()`) | The producing module's periodic/annual event returned bad output | Read the named event handler from the traceback; check upstream `sim$` objects at that time step |
| Infinite loop / sim never advances | Event scheduling bug | In the affected module, check `scheduleEvent()` calls — the event may be scheduling at the same time as `time(sim)` instead of `time(sim) + interval` |

---

## Post-fix Checklist

Before re-running:
- [ ] Only one change made
- [ ] The change is in the right file (module `.R` vs `global.R`)
- [ ] Re-running from the correct step (not always from the top)
- [ ] Cache is not stale (clear only if a data input changed)
