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

## `experiment2()`

Runs factorial simulation experiments — all combinations of parameter values.
Provided by `SpaDES.experiment` (not `SpaDES.core`) — ensure it is loaded or listed in `reqdPkgs`.

```r
# requires SpaDES.experiment
results <- experiment2(
  mySim,
  params = list(
    moduleName = list(paramA = c(1, 5, 10))
  ),
  replicates = 3
)
```

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
      sim <- scheduleEvent(sim, P(sim)$.plotInitialTime,  "myModule", "plot",
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
