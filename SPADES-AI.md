# SpaDES AI Context

SpaDES (Spatial Discrete Event Simulation) is an R meta-package ecosystem for building modular,
spatiotemporally explicit ecological models on a discrete event simulation engine. Modules are
reusable R scripts with formal metadata; they share a single simulation environment (`simList`)
and communicate exclusively through named objects on that environment. Any model that can be
written in R — or called from R via Python, C++, or Java — can be built as a SpaDES workflow.

Key packages: `SpaDES.core`, `reproducible`, `Require`, `SpaDES.project`, `SpaDES.experiment`, `LandR`.
Primary GitHub org: https://github.com/PredictiveEcology

---

## PERFICT Principles

| Principle | Meaning | Code implication |
|-----------|---------|-----------------|
| **P**redict frequently | Re-run often as data or understanding changes | Wrap slow calls in `Cache()` so re-runs are fast |
| **E**valuate | Validate against real data, not just check for errors | Include validation modules; use out-of-sample data |
| **R**eusable | Modules work across study areas and workflows | Declare all inputs/outputs in metadata; no hardcoded assumptions |
| **F**reely accessible | Data and code are open and linkable | Use FAIR data sources; link via URLs in `prepInputs()` |
| **I**nteroperable | Modules connect through shared contracts | Respect `expectsInput`/`createsOutput` contracts exactly |
| **C**ontinuous | End-to-end automation; no manual steps | Everything runs from a single entry point script |
| **T**ested | Automated tests catch regressions | Use `SpaDES.core::testInit()` in tests; write `testthat` tests in `tests/` |

---

## Mental Model: Three Layers

```
Functions → Modules → Models/Workflows
```

- **Functions** are R functions: the atomic unit of computation (bricks)
- **Modules** are `.R` files containing `defineModule()` metadata + event functions + helper functions. They declare what objects they need and produce. (structures built from bricks)
- **Models/Workflows** are collections of modules assembled by a control script via `simInit()`. (cities assembled from structures)

Modules do not call each other directly. They read from and write to `sim$` (the shared `simList` environment). The SpaDES scheduler executes events in time order.

---

## Key Objects

| Expression | What it is |
|-----------|-----------|
| `sim$objectName` | Shared simulation environment — read and write all shared objects here |
| `P(sim)$paramName` | Module parameters — **read-only** during events; set at `simInit()` time |
| `mod$localVar` | Module-local variables — private to the module; not visible to other modules |
| `time(sim)` | Current simulation time (numeric, in units of `timeunit`) |
| `start(sim)` | Simulation start time |
| `end(sim)` | Simulation end time |
| `timeunit(sim)` | Time unit string, e.g. `"year"` |

---

## File Map — Read Next

| Topic | File |
|-------|------|
| Writing or editing a module | `core/module-anatomy.md` |
| How the simulation object works / event scheduling | `core/simlist-events.md` |
| Caching, data download, reproducibility | `core/caching-reproducibility.md` |
| Package management | `core/require-packages.md` |
| PERFICT in depth | `core/perfict-principles.md` |
| LandR Biomass forest succession pipeline | `workflows/LandR-biomass.md` |
| Creating a module-level CLAUDE.md | `templates/module-context-template.md` |
| Setting up a new project (`setupProject()` syntax) | `core/setup-project.md` |
| Writing a `global.R` from scratch | `workflows/global-construction.md` |
| Step-by-step development workflow (first run, reading modules, iteration) | `workflows/development-workflow.md` |
| Error reference — diagnosing `setupProject`, `simInit2`, `spades()` failures | `workflows/iterative-run.md` |

> Paths are relative to the [`spades.ai`](https://github.com/PredictiveEcology/spades.ai) repository root. When this file is used as `CLAUDE.md` in a module repo, fetch the referenced files from that repository.

---

## Critical Rules — Never Violate

1. **Never use `library()` or `install.packages()`** — use `Require()` from the `Require` package. The one permitted exception: `library(SpaDES.project)` at the very top of a control script, before `Require` is available for bootstrapping.
2. **Never hardcode paths** — use `inputPath(sim)`, `outputPath(sim)`, `modulePath(sim)`, `cachePath(sim)`
3. **Wrap all slow/expensive calls in `Cache()`** — data downloads, geoprocessing, model fitting
4. **Check `suppliedElsewhere()` before assigning defaults in the `init` event function** — prevents overwriting objects another module will provide
5. **Read `defineModule()` metadata before editing any module logic** — the declared inputs/outputs are the contract
6. **Never modify `P(sim)$`** — parameters are read-only after `simInit()`; use `mod$` for mutable module state
7. **Always `scheduleEvent()` recurring events at the end of each event handler** — forgetting this silently stops the event from firing again
