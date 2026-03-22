# Writing a global.R From Scratch

At this stage modules are not yet downloaded — write defensively and annotate unknowns with `# TODO:`. The first `source("global.R")` will download modules and reveal correct parameter names and contracts from each module's `defineModule()` metadata. See `core/setup-project.md` for full `setupProject()` syntax, argument reference, and the bootstrap block pattern.

---

## Which path applies?

| Scenario | Path |
|---|---|
| User provides the module list | **Path A — Application** |
| User describes a goal; you must choose modules | **Path B — Development** |

---

## Path A: Application (modules known)

Use this path when the user has already decided which modules to run.

### Step 1 — Write the bootstrap block

Copy the canonical bootstrap block from `core/setup-project.md` (Bootstrap Block section). The required elements are:

- `Require::setupOff()` at the very top — disables Require's automatic package management for the rest of the session
- `getOrUpdatePkg()` — installs or upgrades `Require`, `SpaDES.project`, and any other bootstrap-phase packages only if they are out of date
- `Require::setLinuxBinaryRepo()` — on Linux hosts, prefers pre-compiled binaries over source builds

```r
# ── Bootstrap ──────────────────────────────────────────────────────────────────
Require::setupOff()

repos <- c("https://predictiveecology.r-universe.dev", getOption("repos"))

source("https://raw.githubusercontent.com/PredictiveEcology/pemisc/refs/heads/development/R/getOrUpdatePkg.R")

getOrUpdatePkg(
  c("Require", "SpaDES.project", "reticulate"),
  c("1.0.1.9003", "0.1.1.9037",  "1.43.0")
)

if (!require("SpaDES.project")) {
  Require::Require(
    c("SpaDES.project", "SpaDES.core", "reproducible"),
    repos        = repos,
    dependencies = TRUE
  )
}

Require::setLinuxBinaryRepo()
```

### Step 2 — Define project-level scalars

Define any scalar used in more than one place before calling `setupProject()`. This allows them to be referenced inside `setupProject()` arguments, including inside curly-brace expressions.

```r
# ── Project-level parameters ───────────────────────────────────────────────────
startYear    <- 2011L
periodLength <- 10L   # years per planning period
horizon      <- 10L   # number of planning periods
```

### Step 3 — Call `setupProject()`

Build the full `out` list. A realistic example with two GitHub modules:

```r
out <- SpaDES.project::setupProject(

  paths = list(
    projectPath = getwd(),
    modulePath  = "modules",
    inputPath   = "inputs",
    outputPath  = "outputs",
    cachePath   = "cache"
  ),

  times = list(
    start = startYear,
    end   = startYear + periodLength * horizon
  ),

  modules = c(
    "PredictiveEcology/Biomass_borealDataPrep@main",
    "PredictiveEcology/Biomass_core@main"
  ),

  params = list(
    .globals = list(
      periodLength = periodLength   # shared by two or more modules
    ),
    Biomass_core = list(
      .plotInitialTime = NA   # module-specific; not shared
      # TODO: confirm additional parameter names from defineModule()
    )
  ),

  studyArea = {
    # TODO: replace with real polygon source
    # Example using bcdata:
    # bcdata::bcdc_query_geodata("...") |> dplyr::filter(...) |> dplyr::collect()
    terra::vect("inputs/studyArea.shp")
  },

  studyAreaLarge = {
    sf::st_buffer(studyArea, dist = 20000)   # 20 km buffer for data download extent
  },

  packages = c("terra", "data.table", "sf"),

  options = list(
    spades.allowInitDuringSimInit = TRUE,
    reproducible.useCache         = TRUE
  )
)
```

### Step 4 — Put shared scalars in `.globals`

Any parameter read by two or more modules must go in `params$.globals`, not in a module-specific block. Module-specific blocks are for parameters consumed by exactly one module.

```r
params = list(
  .globals = list(
    periodLength = periodLength   # shared; accessible via P(sim)$.globals$periodLength
  ),
  MyModule = list(
    localParam = 42               # consumed only by MyModule
  )
)
```

### Step 5 — Use curly-brace expressions for spatial objects

`studyArea`, `studyAreaLarge`, and any derived spatial object should be written as curly-brace expressions. They are evaluated in order within `setupProject()`, so later blocks can reference earlier ones:

```r
studyArea = {
  # TODO: replace placeholder with real polygon retrieval
  terra::vect("inputs/studyArea.shp")
},
studyAreaLarge = {
  sf::st_buffer(studyArea, dist = 20000)
},
rasterToMatch = {
  sa  <- terra::vect(studyArea)
  rtm <- terra::rast(sa, res = c(250, 250), crs = terra::crs(sa))
  rtm[] <- 1
  terra::mask(rtm, sa)
}
```

### Step 6 — End with the three-step execution block

```r
# Step 1: setupProject() — already called above; `out` is now a named list
# Step 2: Initialize the sim (downloads remaining data, runs init events)
sim <- SpaDES.core::simInit2(out)
# Step 3: Run the event queue
sim <- SpaDES.core::spades(sim)
```

`simInit2()` initializes the sim and runs init events. `spades()` executes the full event queue. Keeping them separate lets you inspect or modify `sim` between the two steps, which is useful for debugging.

---

## Path B: Development (goal known, modules unknown)

Use this path when the user has described what they want to simulate but has not specified which modules to use.

### Module selection guide

| Domain | Module(s) | GitHub reference |
|---|---|---|
| Forest inventory / data prep | `Biomass_borealDataPrep` | `PredictiveEcology/Biomass_borealDataPrep@main` |
| Annual succession (growth, mortality) | `Biomass_core` | `PredictiveEcology/Biomass_core@main` |
| Fire disturbance | `scfm` suite | `file.path("PredictiveEcology/scfm@development/modules", c("scfmDataPrep", "scfmLandcoverInit", "scfmDriver", "scfmEscape", "scfmSpread", "scfmRegime"))` |
| Harvest (WS3) | `biomass_ws3Harvest` | local / project-specific |
| Carbon accounting | `LandRCBM` suite | `PredictiveEcology/LandRCBM@main` |
| Yield tables for WS3 | `biomass_yieldTablesWS3` | local / project-specific |

### Step-by-step

1. Identify the domain(s) from the user's goal — match each domain to a row in the table above.
2. Select the appropriate modules and GitHub references.
3. For new or custom modules being developed locally (not yet on GitHub), add their parent directory to `modulePath` as a vector rather than listing a GitHub reference in `modules`.
4. Write `global.R` following Path A steps. Annotate uncertain parameter names with `# TODO:` — do not guess.
5. Run `source("global.R")`. The first run downloads modules and prints their `defineModule()` metadata, which reveals the correct parameter names and expected types. Update the `# TODO:` annotations once the names are confirmed.

### Example — local and GitHub modules together

```r
out <- SpaDES.project::setupProject(
  paths = list(
    projectPath = getwd(),
    modulePath  = c("modules", "local_modules"),  # local_modules/ holds in-development modules
    inputPath   = "inputs",
    outputPath  = "outputs",
    cachePath   = "cache"
  ),

  modules = c(
    "PredictiveEcology/Biomass_borealDataPrep@main",
    "PredictiveEcology/Biomass_core@main",
    "myNewHarvestModule"   # lives in local_modules/myNewHarvestModule/
  ),

  params = list(
    .globals = list(
      periodLength = periodLength
    ),
    myNewHarvestModule = list(
      # TODO: fill in after first run reveals defineModule() param names
    )
  ),

  studyArea = {
    # TODO: replace with real polygon
    terra::vect("inputs/studyArea.shp")
  },
  studyAreaLarge = {
    sf::st_buffer(studyArea, dist = 20000)
  },

  packages = c("terra", "data.table", "sf"),

  options = list(
    spades.allowInitDuringSimInit = TRUE,
    reproducible.useCache         = TRUE
  )
)

# Step 1: setupProject() — already called above; `out` is now a named list
# Step 2: Initialize the sim (downloads remaining data, runs init events)
sim <- SpaDES.core::simInit2(out)
# Step 3: Run the event queue
sim <- SpaDES.core::spades(sim)
```

---

## Common Patterns (both paths)

| Rule | Why |
|---|---|
| Never use `library()` — use `packages` in `setupProject()` | `library()` breaks module isolation; packages declared in `setupProject()` are managed correctly across all modules |
| Never use `setwd()` | `setupProject()` manages all paths; `setwd()` breaks reproducibility by making paths session-dependent |
| Typically define `studyAreaLarge` as a 20 km buffer around `studyArea` | `Biomass_borealDataPrep` downloads climate and inventory data for the surrounding region; the buffer size is functional, not cosmetic |
| Annotate unknowns with `# TODO:` | The first run reveals correct parameter names from `defineModule()` metadata; leave placeholders rather than guessing wrong names |
| Put shared scalars in `.globals` | Any parameter read by two or more modules must be in `params$.globals`, not in a module-specific block |
| Use direct calls `sim <- SpaDES.core::simInit2(out)`, not `do.call(...)` | `out` is a named list accepted directly; direct calls are more readable and match the recommended three-step pattern |
