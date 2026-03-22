# SpaDES Development Workflow

The core insight is that **before the first `source("global.R")`, modules are not yet downloaded and the AI has limited context.** The first run is a bootstrapping step — it downloads modules, installs packages, and creates the project structure. After that, the AI reads the downloaded module files to build context before initializing the simulation.

---

## Step 0: Establish Editing Scope

**At the start of every session**, before writing or modifying any code, ask the user:

> "Which modules are you actively developing in this project? I will only edit those modules and `global.R`. All other modules are read-only — I may read them for context but will not modify them."

**Rule:** Once the user specifies the editable modules, treat all other module files as read-only for the remainder of the session. This includes modules downloaded from GitHub and any modules owned by other team members. The modular architecture of SpaDES means modules can be developed and maintained independently — editing a module you don't own can break it for others or introduce conflicts with upstream changes.

**Always editable without asking:** `global.R` and any helper files in the project-level `R/` directory.

**If a fix seems to require editing a locked module:** Stop and tell the user before making any changes. The right approach is almost always to adjust `global.R` (parameters, inputs, module list) rather than patching a module you don't own.

---

## Step 1: Write `global.R`

Follow `workflows/global-construction.md` (and `core/setup-project.md` for `setupProject()` syntax reference). At this point modules are not yet downloaded. Write defensively; annotate unknowns with `# TODO:`.

---

## Step 2: Run `setupProject()`

```r
out <- SpaDES.project::setupProject(...)
```

- What happens: `Require` downloads and installs all referenced modules and their package dependencies. May take several minutes on first run.
- **If it succeeds:** `out` is a named list. Module files are now in `modules/` (or the path in `modulePath`).
- **What to anticipate:** package installation errors, GitHub auth errors, version conflicts, network errors on the bootstrap `source()` line
- **On error:** see `workflows/iterative-run.md` → `setupProject()` errors table

---

## Step 3: Read Downloaded Modules

**Before** running `simInit2`, read the main `.R` file of each downloaded module.

Location pattern: `modules/<ModuleName>/<ModuleName>.R`

What to look for in `defineModule()`:

### `inputObjects` table

```r
expectsInput(
  objectName   = "speciesLayers",
  objectClass  = "SpatRaster",
  desc         = "Species relative biomass layers",
  sourceURL    = NA,
  sourceModuleSelect = "Biomass_borealDataPrep"
)
```

Two fields to examine:

- `objectClass` — the R class `sim$X` must be when this module reads it. If you supply the object manually (e.g., as a curly-brace expression in `setupProject()`), ensure the object's class matches exactly.
- `sourceModuleSelect` — which module produces this object. If set, a "missing object" error means that module is absent from the `modules` list, not that you need to supply the object manually. If `NA`, you must supply it.

### `outputObjects` table

```r
createsOutput(
  objectName  = "cohortData",
  objectClass = "data.table",
  desc        = "Cohort-level biomass and age by pixel"
)
```

Trace the event schedule (`doEvent.*` functions) to understand at which timestep each output is produced. An output listed in `outputObjects` may not exist in `sim$` until the event that creates it has fired.

### Event schedule

Look at `scheduleEvent()` calls inside `doEvent.*` functions. This tells you:

- Which events fire and when
- Whether an event reschedules itself (recurring) or fires once
- Inter-module timing dependencies

For example, a recurring event pattern looks like:

```r
doEvent.MyModule.grow <- function(sim, eventTime, eventType) {
  sim <- scheduleEvent(sim, time(sim) + P(sim)$growthInterval, "MyModule", "grow")
  # ... work ...
  sim
}
```

A once-only `init` event does not call `scheduleEvent()` for itself again — it only schedules downstream events.

This context is critical for diagnosing errors in Steps 4 and 5.

---

## Step 4: Run `simInit2()`

```r
sim <- SpaDES.core::simInit2(out)
```

- What happens: Validates module metadata, checks required `inputObjects`, runs `init` events, sets up the event queue
- **Important:** When `spades.allowInitDuringSimInit = TRUE` (the default in this project), `init` events run during `simInit2`. Errors that look like `spades()` errors may surface here — check the traceback for the event name.
- **If it succeeds:** inspect with `inputs(sim)`, `outputs(sim)`, `events(sim)`
- **On error:** see `workflows/iterative-run.md` → `simInit2()` errors table

---

## Step 5: Run `spades()`

```r
sim <- SpaDES.core::spades(sim)
```

- What happens: Executes the event queue in time order
- **If it succeeds:** outputs are in `sim$` and in `paths$outputPath`
- **On error:** the traceback names the module and event; read that event handler function
- **On error:** see `workflows/iterative-run.md` → `spades()` errors table

---

## General Principles

- **Never skip `simInit2`** — use the three-step pattern during all development. `simInitAndSpades()` is acceptable only in finalized production scripts where the run is known to succeed.
- **`simInit2` runs `init` events** — see Step 4 for the `spades.allowInitDuringSimInit = TRUE` caveat; errors that look like `spades()` errors may surface during `simInit2`.
- **Restart from the failing step** — after a fix, do not re-run `setupProject()` unless `out` itself changed:
  - Fix is in `global.R` (modules list, params, paths) → re-run `setupProject()` and continue
  - Fix is in a module `.R` file only → re-run from `simInit2(out)` directly
- **Cache is your friend** — `reproducible.useCache = TRUE` skips expensive data downloads on re-runs. Do not clear cache unless a data input genuinely changed.
- **`loadOrder` matters** — if `simInit2` throws a dependency error naming two modules, add `out$loadOrder <- unlist(out$modules)` before retrying.

---

## Quick Reference

```
source("global.R")  →  setupProject()  →  Read modules/  →  simInit2()  →  spades()
                              ↓                                    ↓               ↓
                        modules/ created               inspect sim$        outputs written
                        packages installed             events(sim)         to outputPath
```
