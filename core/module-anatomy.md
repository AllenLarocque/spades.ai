# Module Anatomy

A SpaDES module is a single `.R` file (plus optional support files) that declares its metadata
via `defineModule()` and implements its behaviour via event functions routed through `doEvent()`.

---

## Directory Structure

```
moduleName/
├── moduleName.R          # Main file: metadata + doEvent() + event functions + helpers
├── moduleName.Rmd        # Human-readable documentation (rendered to HTML)
├── R/                    # Additional R scripts; sourced automatically by SpaDES.core
│   └── helpers.R
├── data/
│   └── CHECKSUMS.txt     # Expected checksums for data files shipped with the module
├── tests/
│   └── testthat/
│       └── test-moduleName.R
├── citation.bib
├── LICENSE.md
└── NEWS.md
```

---

## `defineModule()` Skeleton

Every module file begins with a `defineModule(sim, list(...))` call. All fields are required
unless marked optional.

```r
defineModule(sim, list(
  name = "moduleName",
  description = "One paragraph: what this module does and why.",
  keywords = c("keyword1", "keyword2"),
  authors = c(
    person("First", "Last", email = "email@example.com", role = c("aut", "cre"))
  ),
  childModules = character(0),   # Names of sub-modules this module manages, if any
  version = list(moduleName = "1.0.0"),
  spatialExtent = terra::ext(rep(NA_real_, 4)),  # terra::ext(), NOT raster::extent() (deprecated)
  timeframe = as.POSIXlt(c(NA, NA)),
  timeunit = "year",             # "second", "hour", "day", "month", "year"
  citation = list("citation.bib"),
  documentation = list("README.txt", "moduleName.Rmd"),
  reqdPkgs = list(              # All packages the module needs — SpaDES installs these
    "data.table", "terra",
    "PredictiveEcology/reproducible@main"  # GitHub packages use "org/repo@ref" syntax
  ),
  parameters = rbind(
    #                         name          class    default  min   max
    defineParameter("paramName",       "numeric",       1,   0,  10,
                    desc = "What this parameter controls."),
    defineParameter(".plotInitialTime","numeric",      NA,  NA,  NA,
                    desc = "Simulation time for first plot. NA = no plots."),
    defineParameter(".plotInterval",   "numeric",       1,  NA,  NA,
                    desc = "Interval between plots."),
    defineParameter(".saveInitialTime","numeric",      NA,  NA,  NA,
                    desc = "Simulation time for first save. NA = no saves."),
    defineParameter(".saveInterval",   "numeric",       1,  NA,  NA,
                    desc = "Interval between saves.")
  ),
  inputObjects = rbind(
    expectsInput("objectName", "SpatRaster",
                 desc = "What this object is and why the module needs it.",
                 sourceURL = "https://example.com/source"),
    expectsInput("studyArea", "SpatVector",
                 desc = "Study area polygon used to crop/mask inputs.")
  ),
  outputObjects = rbind(
    createsOutput("outputObjectName", "data.table",
                  desc = "What this object contains and who will consume it.")
  )
))
```

### Field Notes

- **`reqdPkgs`**: List entries are strings. Use `"pkg (>= 1.2.0)"` for version constraints.
  GitHub: `"org/repo@branch"`. SpaDES reads this before running and installs missing packages.
- **`parameters`**: `.plotInitialTime`, `.plotInterval`, `.saveInitialTime`, `.saveInterval`
  are conventional names — include them if the module plots or saves.
- **`inputObjects`**: List every `sim$` object the module reads. If you read it, declare it.
- **`outputObjects`**: List every `sim$` object the module assigns. If you write it, declare it.

---

## `doEvent()` Dispatcher

`doEvent()` is the event router. SpaDES calls it with the current event type; it dispatches
to the correct event function.

```r
doEvent.moduleName <- function(sim, eventTime, eventType, debug = FALSE) {
  switch(eventType,
    init = {
      sim <- Init(sim)
      sim <- scheduleEvent(sim, P(sim)$.plotInitialTime, "moduleName", "plot")
      sim <- scheduleEvent(sim, P(sim)$.saveInitialTime, "moduleName", "save")
      sim <- scheduleEvent(sim, start(sim) + 1,          "moduleName", "grow")
    },
    plot = {
      sim <- Plot(sim)
      sim <- scheduleEvent(sim, time(sim) + P(sim)$.plotInterval, "moduleName", "plot")
    },
    save = {
      sim <- Save(sim)
      sim <- scheduleEvent(sim, time(sim) + P(sim)$.saveInterval, "moduleName", "save")
    },
    grow = {
      sim <- Grow(sim)
      sim <- scheduleEvent(sim, time(sim) + 1, "moduleName", "grow")
    },
    warning(paste("Undefined event type: '", eventType, "' in module '",
                  currentModule(sim), "'", sep = ""))
  )
  return(invisible(sim))
}
```

**Rules for `doEvent()`:**
- Always return `invisible(sim)`
- Always schedule the next occurrence of recurring events at the end of each event handler
- Use `time(sim) + P(sim)$.plotInterval` (not a hardcoded number) for intervals
- The `warning()` default case catches typos in event names early

---

## Standard Event Functions

### `Init`

```r
Init <- function(sim) {
  # 1. Check which objects are already supplied by another module
  if (!suppliedElsewhere("outputObjectName", sim)) {
    sim$outputObjectName <- data.table(...)  # create default
  }

  # 2. Download / prepare inputs
  sim$inputData <- Cache(
    prepInputs,
    url = extractURL("inputData", sim),  # reads sourceURL from expectsInput metadata
    destinationPath = inputPath(sim),
    studyArea = sim$studyArea,
    rasterToMatch = sim$rasterToMatch
  )

  return(invisible(sim))
}
```

### Custom event (e.g., `Grow`)

```r
Grow <- function(sim) {
  # Read inputs from sim$
  cohorts <- sim$cohortData

  # Do computation
  cohorts[, B := B + growthIncrement(B, maxB)]

  # Write outputs back to sim$
  sim$cohortData <- cohorts

  return(invisible(sim))
}
```

### `Plot` and `Save`

```r
Plot <- function(sim) {
  if (!is.na(P(sim)$.plotInitialTime)) {
    quickPlot::Plot(sim$outputRaster, title = "My Output")  # quickPlot::Plot, NOT recursive self-call
  }
  return(invisible(sim))
}

Save <- function(sim) {
  saveFiles(sim)
  return(invisible(sim))
}
```

---

## Common AI Mistakes

| Mistake | Consequence | Fix |
|---------|------------|-----|
| Declare object in `createsOutput` but never assign `sim$object` | Other modules expecting it get NULL | Assign it in `Init` or the relevant event |
| Use `expectsInput` for objects the module itself creates | Misleads dependency resolution | Move to `createsOutput` |
| Forget `scheduleEvent()` at end of recurring event | Event fires once, then silently stops | Add scheduling at the end of every event handler |
| Modify `P(sim)$param` | Error — parameters are read-only | Store mutable state in `mod$` |
| Use `library()` inside a module | Bypasses reproducibility guarantees | Add package to `reqdPkgs`; use `::` notation |
| Hardcode a path | Breaks on any other machine | Use `inputPath(sim)`, `outputPath(sim)`, etc. |
| Read `sim$x` without declaring `expectsInput("x", ...)` | Silent contract violation; fragile | Add `expectsInput` entry |
