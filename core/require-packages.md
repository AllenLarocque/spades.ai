# Package Management with `Require`

SpaDES uses a non-standard package management system. **Never use `library()`,
`require()`, or `install.packages()` in SpaDES module code.**

---

## Why `Require()` not `library()`

| `library()` / `install.packages()` | `Require()` |
|------------------------------------|-------------|
| Errors if package not installed | Installs if missing |
| No version constraint support | Handles `(>= x.y.z)` constraints |
| CRAN only | CRAN + GitHub + local |
| No reproducibility integration | Works with SpaDES caching guarantees |
| Silent version conflicts | Warns on conflicts; can pin exact versions |

```r
# Wrong — never do this in a module
library(terra)
install.packages("data.table")

# Right
Require::Require(c("terra", "data.table"))  # Require::Require() — package and function share the same name
```

---

## `reqdPkgs` — Declaring Module Dependencies

All packages a module needs go in the `reqdPkgs` field of `defineModule()`. SpaDES reads
this before running and installs any missing packages.

```r
reqdPkgs = list(
  "terra",                                         # CRAN; any version
  "data.table (>= 1.14.0)",                        # CRAN; minimum version
  "PredictiveEcology/reproducible@main",           # GitHub; main branch
  "PredictiveEcology/SpaDES.core@development",     # GitHub; development branch
  "PredictiveEcology/LandR@HEAD"                   # GitHub; latest commit
),
```

**Syntax:**
- CRAN package: `"pkgName"` or `"pkgName (>= x.y.z)"`
- GitHub package: `"org/repo"` or `"org/repo@branch"` or `"org/repo@tag"`

**Do not put packages in `reqdPkgs` and also call `Require()` inside the module.** Pick one.
`reqdPkgs` is the preferred approach — it makes dependencies visible in module metadata.

---

## `SpaDES.project::setupProject()`

The recommended entry point for new projects. Sets up directory structure, installs all
module dependencies, and configures paths.

```r
library(SpaDES.project)  # bootstrap exception: library() is permitted ONLY here, before Require is available

out <- setupProject(
  name     = "myProject",
  paths    = list(projectPath = "~/myProject"),
  modules  = c(
    "PredictiveEcology/Biomass_core@main",
    "PredictiveEcology/Biomass_speciesData@main"
  ),
  params   = list(
    Biomass_core = list(.plotInitialTime = NA)
  ),
  # objects and times can also be passed here
)

# out is a named list of arguments — pass to simInit() via do.call, NOT a simList itself
mySim <- do.call(simInit, out)
mySim <- spades(mySim)
```

`setupProject()` calls `Require()` internally to install all packages declared in the
`reqdPkgs` of each listed module before `simInit()` runs.

---

## `SpaDES.install`

Helper package for managing SpaDES-specific installations, particularly on HPC environments
or when installing the full SpaDES ecosystem.

```r
# Install the full SpaDES ecosystem
install.packages("SpaDES.install")
SpaDES.install::installSpaDES()
```

Use this for initial environment setup only — not inside module code.

---

## Using Packages Inside Module Code

After declaring a package in `reqdPkgs`, use `::` notation inside module functions:

```r
# Good — explicit namespace
myData <- data.table::data.table(x = 1:10, y = letters[1:10])
myRaster <- terra::rast(nrows = 100, ncols = 100)

# Acceptable for heavily-used packages in a module that declares them in reqdPkgs
myData <- data.table(x = 1:10)  # if data.table is in reqdPkgs

# Bad — never do this inside module code
library(terra)
```
