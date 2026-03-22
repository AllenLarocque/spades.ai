# global.R and setupProject() — SpaDES Project Entry Point

`global.R` is the single control script for a SpaDES project. It is sourced from the terminal:

```r
source("global.R")
```

All simulation configuration lives here: paths, module references, parameters, and shared objects. `SpaDES.project::setupProject()` is the standard way to build this configuration — it returns a named list that is passed directly to `simInit2()` + `spades()` (or the convenience wrapper `simInitAndSpades()`).

---

## Bootstrap Block

The bootstrap block runs before any modules are downloaded. Its job is to ensure `Require` and `SpaDES.project` are present at the correct versions. This is the canonical pattern used in production projects:

```r
# ── Bootstrap ──────────────────────────────────────────────────────────────────

# Disable Require's automatic package management during the sim run.
# This prevents Require from re-checking or re-installing packages that
# are already handled by the bootstrap and setupProject() phases.
Require::setupOff()

# Build the repo list, putting r-universe first so pre-built packages are
# preferred over CRAN source builds.
repos <- c("https://predictiveecology.r-universe.dev", getOption("repos"))

# Source a helper that installs/upgrades a package only if the installed
# version is older than the required minimum.
source("https://raw.githubusercontent.com/PredictiveEcology/pemisc/refs/heads/development/R/getOrUpdatePkg.R")

# Ensure bootstrap packages are at the required minimum versions.
getOrUpdatePkg(
  c("Require", "SpaDES.project", "reticulate"),
  c("1.0.1.9003", "0.1.1.9037",  "1.43.0")
)

# Fallback: if the raw GitHub source() line above fails (e.g., network
# unavailable), comment it out and run this instead:
#   install.packages("Require")
# Then re-run and the getOrUpdatePkg() call will succeed.

# Ensure SpaDES.project is present before the main setup block.
if (!require("SpaDES.project")) {
  Require::Require(
    c("SpaDES.project", "SpaDES.core", "reproducible"),
    repos       = repos,
    dependencies = TRUE
  )
}

# On Linux, use pre-compiled binary packages from r-universe instead of
# compiling from source. This is faster, avoids system library issues,
# and does not require root access.
Require::setLinuxBinaryRepo()
```

### `Require::setupOff()`

`Require::setupOff()` disables Require's automatic package management for the rest of the session. Call it at the very top of `global.R`, before any other `Require` calls. Without it, Require may attempt to re-check or re-install packages during the sim run, which is redundant and can slow startup.

### `Require::setLinuxBinaryRepo()`

On Linux hosts (CI, servers, developer workstations), R packages normally compile from source, which requires system build tools and takes significantly longer. `Require::setLinuxBinaryRepo()` configures R to prefer pre-compiled binaries from the PredictiveEcology r-universe. Call it after `Require` is confirmed to be installed.

### Network failure fallback

If the `source("https://raw.githubusercontent.com/...")` line fails due to a network issue, comment it out and install `Require` manually:

```r
# install.packages("Require")
```

Then re-run `global.R`. Once `Require` is installed, `getOrUpdatePkg()` can be sourced on subsequent runs.

---

## Minimal Pattern

```r
# ── Bootstrap ─────────────────────────────────────────────────────────────────
# See the "Bootstrap Block" section above for the full production pattern.
if (!require("SpaDES.project")) stop("Run the Bootstrap Block above first.")

# ── Top-level variables ────────────────────────────────────────────────────────
# Define scalars here. They can be referenced inside setupProject() arguments,
# including inside curly-brace expressions (evaluated in order, in scope).
startYear  <- 2011
endYear    <- 2111
myParam    <- 42

out <- SpaDES.project::setupProject(

  # ── Paths ──────────────────────────────────────────────────────────────────
  paths = list(
    projectPath = getwd(),
    modulePath  = "modules",
    inputPath   = "inputs",
    outputPath  = "outputs",
    cachePath   = "cache"
  ),

  # ── Simulation time ────────────────────────────────────────────────────────
  times = list(start = startYear, end = endYear),

  # ── Modules (GitHub references) ────────────────────────────────────────────
  # Format: "org/repo@branch"
  # Sub-modules: file.path("org/repo@branch/modules", c("mod1", "mod2"))
  modules = c(
    "PredictiveEcology/Biomass_borealDataPrep@main",
    "PredictiveEcology/Biomass_core@main"
  ),

  # ── Parameters ─────────────────────────────────────────────────────────────
  # .globals are accessible to all modules via P(sim)$.globals$<name>
  params = list(
    .globals = list(
      sharedParam = myParam
    ),
    Biomass_core = list(
      .plotInitialTime = NA
    )
  ),

  # ── Shared objects (passed into sim$) ─────────────────────────────────────
  # Curly-brace expressions are evaluated in order and in scope —
  # later ones can reference earlier ones.
  studyArea = {
    terra::vect("inputs/studyArea.shp")
  },
  rasterToMatch = {
    terra::rast("inputs/rtm.tif")
  },

  # ── Packages ───────────────────────────────────────────────────────────────
  packages = c("terra", "data.table"),

  # ── Options ────────────────────────────────────────────────────────────────
  options = list(
    spades.allowInitDuringSimInit = TRUE,
    reproducible.useCache         = TRUE
  ),

  # ── Functions ──────────────────────────────────────────────────────────────
  # Sources helper R files before sim runs
  functions = "R/helpers.R"
)

# ── Run (three-step pattern) ──────────────────────────────────────────────────
# Step 1: setupProject() — downloads modules, returns named list `out`
# Step 2: simInit2()     — initializes sim, runs init events
# Step 3: spades()       — executes event queue
sim <- SpaDES.core::simInit2(out)   # out is a named list accepted directly
sim <- SpaDES.core::spades(sim)
```

---

## Key Concepts

### Top-level variables as arguments

Any named scalar defined before `setupProject()` can be referenced inside its arguments, including inside `{}` expressions. This is the standard way to define project-level parameters that multiple modules share:

```r
periodLength <- 10
horizon      <- 10

out <- SpaDES.project::setupProject(
  times = list(start = 2011, end = 2011 + periodLength * horizon),
  params = list(
    .globals = list(periodLength = periodLength)
  )
)
```

### Curly-brace expressions (inline code blocks)

Named arguments whose value is a `{}` block are evaluated in order within `setupProject()`. Later blocks can reference objects created in earlier blocks:

```r
out <- SpaDES.project::setupProject(
  studyArea = {
    terra::vect("inputs/studyArea.shp")
  },
  rasterToMatch = {
    # can reference studyArea because it was defined above
    terra::project(terra::rast(nrows=100, ncols=100), studyArea)
  }
)
```

The resulting objects are injected into `sim$` at init.

### Module references

Modules are referenced as `"org/repo@branch"` strings. `setupProject()` handles downloading, installation, and path setup automatically.

For repos containing multiple sub-modules in a `modules/` directory:

```r
modules = c(
  "PredictiveEcology/Biomass_core@main",
  file.path("PredictiveEcology/scfm@development/modules",
            c("scfmDataPrep", "scfmLandcoverInit", "scfmDriver"))
)
```

### Local modules (not on GitHub)

Modules that live only on disk — not published to GitHub — should be placed in a local directory and that directory added to `modulePath` as a vector. This is done after `setupProject()` returns:

```r
out <- SpaDES.project::setupProject(
  paths = list(modulePath = "modules"),
  modules = c("PredictiveEcology/Biomass_core@main")
)

# Add local module directory alongside the GitHub-resolved path
out$paths$modulePath <- c(out$paths$modulePath, "local_modules")
```

`setupProject()` accepts `modulePath` as a vector directly if all paths are known upfront, but the post-processing pattern is more flexible when the local path depends on runtime context.

### `require` vs `packages` — which one to use

`setupProject()` has two separate package arguments with distinct timing:

| Argument   | When it runs              | Use for                                                                 |
|------------|---------------------------|-------------------------------------------------------------------------|
| `require`  | Bootstrap phase — before modules are downloaded or resolved | `Require`, `SpaDES.project`, `reticulate`, and other packages that must be present before module code runs |
| `packages` | After modules are resolved | Module dependencies, including GitHub packages like `PredictiveEcology/LandR@development` |

**Rule:** Never put a GitHub-hosted package in `require`. The `require` phase runs before `setupProject()` has resolved module sources, so it cannot handle GitHub references. Doing so breaks the install order and produces confusing errors. GitHub packages belong in `packages`.

```r
out <- SpaDES.project::setupProject(

  # Correct: bootstrap-phase packages that must exist before module resolution
  require = c("Require", "SpaDES.project", "reticulate"),

  # Correct: module dependencies, including GitHub packages
  packages = c(
    "qs", "terra", "data.table",
    "PredictiveEcology/LandR@development",
    "PredictiveEcology/SpaDES.core@development (>= 3.0.3.9003)"
  )
)
```

### `.globals` parameters

Parameters in `.globals` are accessible to all modules. Use for shared scalars that multiple modules need (e.g., a period length that drives both a yield-table module and a harvest module):

```r
params = list(
  .globals = list(ws3PeriodLength = 10),
  MyModule  = list(localParam = 5)
)
```

Inside a module, access globals via `P(sim)$.globals$ws3PeriodLength` (or `params(sim)$.globals$ws3PeriodLength`).

### Three-step execution pattern (preferred)

The preferred way to run the simulation is to split initialization and execution into two explicit steps:

```r
# Step 1: setupProject() — downloads modules, returns named list `out`
# Step 2: simInit2()     — initializes sim, runs init events
# Step 3: spades()       — executes event queue
out <- SpaDES.project::setupProject(...)

sim <- SpaDES.core::simInit2(out)   # out is a named list accepted directly
sim <- SpaDES.core::spades(sim)
```

This gives you a chance to inspect or modify the initialized `simList` (`sim`) before the sim runs — useful for debugging, attaching profiling hooks, or confirming module graph structure.

`simInitAndSpades()` is an acceptable shorthand only in finalized production scripts where no intermediate inspection is needed:

```r
sim <- SpaDES.core::simInitAndSpades(out)   # out is a named list accepted directly
```

### Post-processing `out`

After `setupProject()` returns, you can modify `out` before running:

```r
out <- SpaDES.project::setupProject(...)

# Fix module dependency errors by explicitly setting load order.
# If SpaDES cannot resolve the dependency graph automatically, setting
# loadOrder to the flat module list forces a deterministic sequence.
out$loadOrder <- unlist(out$modules)

# Add a locally-cloned module not on GitHub
out$paths$modulePath <- c(out$paths$modulePath, "local_modules")

sim <- SpaDES.core::simInit2(out)   # out is a named list accepted directly
sim <- SpaDES.core::spades(sim)
```

`out$loadOrder <- unlist(out$modules)` is the standard fix when `simInit2()` raises a module dependency error. It flattens the module list into a character vector and passes it as the explicit load order, bypassing automatic graph resolution.

---

## What NOT to do in global.R

- **No `library()` calls** — use `packages` argument in `setupProject()` instead
- **No hardcoded absolute paths** — use `paths$projectPath` + relative refs
- **No module logic** — global.R sets up the sim, modules do the work
- **No `setwd()`** — `setupProject()` handles paths; `setwd()` breaks reproducibility
- **No GitHub packages in `require`** — they belong in `packages`; see the `require` vs `packages` table above

---

## Reference Examples

- Simple LandR + WS3 coupling: `WS3_LandR/global.R` (this project)
- LandR + SCFM fire: [DominiqueCaron/LandRCBM — global_scfm.R](https://github.com/DominiqueCaron/LandRCBM/blob/main/global_scfm.R)
- WS3 harvest demo: [cccandies-demo-202503B — global.R](https://github.com/AllenLarocque/cccandies-demo-202503B/blob/master/global.R)
