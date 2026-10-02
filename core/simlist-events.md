# simList and Event Scheduling

---

## What is `simList`?

`simList` is an R environment (S4 object) that holds all simulation state. Every module shares
the same `simList` instance. Modules communicate exclusively by reading from and writing to it.
There is no direct module-to-module function call.

---

## Accessor Patterns

```r
# Shared objects (readable and writable by any module)
sim$myObject                  # get
sim$myObject <- newValue      # set

# Parameters (read-only after simInit)
P(sim)$paramName              # get parameter for the currently-executing module
params(sim)$moduleName$param  # get parameter for a specific module by name

# Module-local state (private; not accessible to other modules)
mod$localVar                  # get
mod$localVar <- value         # set

# Time accessors
time(sim)                     # current event time (numeric)
start(sim)                    # simulation start time
end(sim)                      # simulation end time
timeunit(sim)                 # time unit string, e.g. "year"

# Module name
currentModule(sim)            # name of the currently-executing module

# Paths
inputPath(sim)                # where to read input data
outputPath(sim)               # where to write output data
modulePath(sim)               # where module directories live
cachePath(sim)                # where Cache() stores results
```

---

## `simInit()`

`simInit()` initialises the simulation. It validates module metadata, checks that declared
inputs and outputs are consistent, and returns a configured `simList`.

```r
mySim <- simInit(
  times   = list(start = 2000, end = 2030),   # numeric; interpreted in timeunit
  params  = list(
    moduleName = list(paramA = 5, .plotInitialTime = NA)
  ),
  modules = list("Biomass_core", "Biomass_speciesData"),
  objects = list(
    studyArea      = myStudyAreaPolygon,
    rasterToMatch  = myTemplateRaster
  ),
  paths     = list(
    modulePath  = file.path("modules"),
    inputPath   = file.path("inputs"),
    outputPath  = file.path("outputs"),
    cachePath   = file.path("cache")
  ),
  loadOrder = c("Biomass_speciesData", "Biomass_core")  # optional; override default init order
)
```

**What `simInit()` does:**
1. Sources all module `.R` files
2. Calls `defineModule()` for each module to collect metadata
3. Validates `expectsInput`/`createsOutput` contracts and warns on gaps
4. Installs missing packages from `reqdPkgs`
5. Schedules each module's `init` event
6. Returns a configured `simList` ready for `spades()`

---

## `spades()`

`spades()` runs the simulation by processing the event queue.

```r
mySim <- spades(mySim)             # run to completion
mySim <- spades(mySim, debug = 1)  # print event queue as it executes
```

**Event queue mechanics:**
- Events are ordered by `(eventTime, priority)` — ties broken by priority (lower number = earlier)
- Each event calls `doEvent.moduleName(sim, eventTime, eventType)`
- `doEvent` may schedule new events; those enter the queue immediately
- Simulation ends when the queue is empty or `time(sim)` would exceed `end(sim)`

---

## `scheduleEvent()`

```r
# Schedule a one-time event
sim <- scheduleEvent(sim, time(sim) + 1, "moduleName", "eventType")

# Schedule with explicit priority (default is normal = 5)
sim <- scheduleEvent(sim, time(sim) + 1, "moduleName", "plot", eventPriority = 6)
```

**Priority constants (from `SpaDES.core`):** `.first` (1), `.normal` (5), `.last` (100).
`start(sim)` and `end(sim)` are time accessors — they are NOT priority constants. Do not confuse them.
Plot and save events conventionally use a priority of 6 (slightly after simulation events at `.normal` = 5).

---

## Experiments — running a workflow many times

An "experiment" runs a simulation repeatedly with varying parameters, inputs, paths, scenarios, or
replicates: stochastic replication, scenario analysis, sensitivity sweeps, ensemble datasets.

> ⚠️ **`SpaDES.experiment` is deprecated** (since 2026-05-22) and no longer maintained. Its
> functions forward to **`SpaDES.project`**, which is now their maintained home. Depend on
> `SpaDES.project`, not `SpaDES.experiment`.

The family splits in two, and **choosing the wrong half is the common mistake**.

### In-memory — `experiment2()` / `experiment()`

Take `simList` object(s) directly, run them, and return a `simLists` (an environment holding every
run's `simList` at once). Post-process with `as.data.table.simLists()`.

Use when the run set is modest, **fits comfortably in RAM**, and you want the results back in your
session. They are **not** built for resume-after-crash, cross-machine runs, or HPC.

```r
# requires SpaDES.project
results <- SpaDES.project::experiment2(
  mySim,
  params = list(moduleName = list(paramA = c(1, 5, 10))),
  replicates = 3
)
```

`experiment()` wraps `experiment2()` and builds the factorial set for you via `factorialDesign()`.

> ⚠️ Every run's `simList` is held simultaneously. If one `simList` is large (hundreds of MB is
> ordinary for a data-heavy workflow), a modest factorial will exhaust RAM. Size this before
> reaching for it.

### Queue-driven — `experimentTmux()` / `experimentFuture()` / `experimentSBATCH()`

Built around a project `global.R` and a shared job queue. Each row of a `data.frame` is one job;
**a worker assigns each column's value to a variable of that name in `.GlobalEnv`, then `source()`s
`global.R`**. So `global.R` is written once and parameterized by the grid.

Use when runs are long, numerous, memory-bound, spread across machines, or need to resume.

```r
df <- data.frame(scenarioName = c("baseline", "highFire"))

SpaDES.project::experimentFuture(
  df,
  global_path  = "global.R",
  queue_path   = "queue.rds",   # file-locked queue; keep it to resume, delete it to start over
  n_workers    = 2,
  runNameLabel = quote(scenarioName)
)
```

- **`experimentTmux()`** — one tmux pane per worker, optionally over ssh. Best while debugging;
  you can watch workers live and stop them with `tmuxKillPanes()`.
- **`experimentFuture()`** — background processes (`callr::r_bg()`, or `future::cluster` when
  workers span machines). Best for stable scripts.
- **`experimentSBATCH()`** — one Slurm job per worker. Inspect generated scripts with
  `dry_run = TRUE`.

The queue's `status` column (`PENDING` / `INTERRUPTED` / `DONE`) is what gives **resume after a
crash**: workers skip `DONE` rows. Set `n_workers = 1` when a single run already saturates RAM —
the queue still buys unattended sequencing and resume.

Because each job's `outputPath` must be distinct, pair this with the **`scenario`** helpers
(`scenarioFieldsSet()`, `as_scenario()`, `as_path()`, `as_tarname()`), which map a run's field
values to an output path and parse it back again.

---

## `restartSpaDES()`

Resumes an interrupted simulation from the current state of module code on disk. Invaluable
during iterative development: edit a module function, call `restartSpaDES()`, and the
simulation continues from where it left off using the new code.

```r
# At R prompt after an error or manual stop:
restartSpaDES()  # no sim argument — SpaDES tracks the sim state internally
```

---

## Event Scheduling Pattern (Complete Example)

```r
doEvent.myModule <- function(sim, eventTime, eventType, debug = FALSE) {
  switch(eventType,
    init = {
      sim <- Init(sim)
      # Schedule first occurrence of each recurring event
      sim <- scheduleEvent(sim, start(sim) + 1,          "myModule", "grow")
      if (!is.na(P(sim)$.plotInitialTime))
        sim <- scheduleEvent(sim, P(sim)$.plotInitialTime, "myModule", "plot",
                             eventPriority = 6)
    },
    grow = {
      sim <- Grow(sim)
      # Schedule NEXT occurrence — without this line, grow fires only once
      sim <- scheduleEvent(sim, time(sim) + 1, "myModule", "grow")
    },
    plot = {
      sim <- Plot(sim)
      sim <- scheduleEvent(sim, time(sim) + P(sim)$.plotInterval, "myModule", "plot",
                           eventPriority = 6)
    },
    warning(paste("Undefined event type: '", eventType, "' in module '",
                  currentModule(sim), "'", sep = ""))
  )
  return(invisible(sim))
}
```
