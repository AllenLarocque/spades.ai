# SpaDES AI Context Documents — Design Spec

**Date:** 2026-03-20
**Status:** Approved by user

---

## Overview

A modular, extensible set of markdown files that give AI coding assistants (Claude, Copilot, Cursor, etc.) the domain knowledge needed to write, modify, and debug code in the SpaDES R meta-platform. Files live in the `spades.ai` repository but are designed to be embedded directly into individual SpaDES module and workflow repos as `CLAUDE.md` / `AGENTS.md` context.

The design mirrors SpaDES's own philosophy: modular, interoperable components that can be composed and extended without duplication.

---

## Background: SpaDES and PERFICT

SpaDES (Spatial Discrete Event Simulation) is an R meta-package ecosystem for building modular, reproducible ecological models. Its philosophical foundation is the **PERFICT** framework (McIntire et al. 2022, Barros et al. 2023):

- **P**redict frequently
- **E**valuate models
- **R**eusable components
- **F**reely accessible
- **I**nteroperable modules
- **C**ontinuous workflows
- **T**ested automatically

Key packages: `SpaDES.core`, `reproducible`, `Require`, `SpaDES.experiment`, `SpaDES.project`, `LandR`.
Primary GitHub org: https://github.com/PredictiveEcology

---

## Target Audience

General-purpose AI coding assistants working on SpaDES projects. The documents are not for end-users — they are for AIs being asked to:
- Write new SpaDES modules from scratch
- Modify or extend existing modules
- Write control scripts and workflows that assemble and run modules
- Debug SpaDES-specific failures (caching, dependency resolution, event scheduling)

---

## Repository Structure

```
spades.ai/
├── SPADES-AI.md                    # Root file — use as CLAUDE.md / AGENTS.md
├── core/
│   ├── perfict-principles.md       # 7 PERFICT principles + code implications
│   ├── module-anatomy.md           # defineModule(), events, functions, dir layout
│   ├── simlist-events.md           # simList, simInit(), spades(), event scheduling
│   ├── caching-reproducibility.md  # Cache(), prepInputs(), suppliedElsewhere()
│   └── require-packages.md         # Require(), package management, version pinning
├── workflows/
│   ├── LandR-biomass.md            # LandR Biomass pipeline: modules, data flow
│   └── (fireSense.md, CBM.md…)     # Future additions follow same pattern
└── templates/
    └── module-context-template.md  # Fill-in template for individual module repos
```

---

## File Specifications

### `SPADES-AI.md` (Root file)

**Purpose:** Always-included anchor. Short enough (~150 lines) to serve as `CLAUDE.md` in any SpaDES project. Orients an AI to the ecosystem before it reads any code.

**Contents:**

1. **What SpaDES is** — One paragraph: R meta-package for modular, spatiotemporally explicit ecological models built on discrete event simulation. Can accommodate any model type that can be written in R or called from R (Python, C++, Java).

2. **PERFICT summary table** — 7 principles, each with a one-line code implication:

   | Principle | Code implication |
   |-----------|-----------------|
   | Predict frequently | Use `Cache()` so re-runs are fast |
   | Evaluate | Include validation modules; use out-of-sample data |
   | Reusable | Declare all inputs/outputs in metadata; no hardcoded assumptions |
   | Freely accessible | Use FAIR data sources; link via URLs in `prepInputs()` |
   | Interoperable | Respect `expectsInput`/`createsOutput` contracts exactly |
   | Continuous | Everything runs end-to-end; no manual intervention steps |
   | Tested | Use `testInit` assertions; write `testthat` tests in `tests/` |

3. **Three-layer mental model** — Functions → Modules → Models/Workflows (the Lego analogy). Functions are bricks; modules are structures built from bricks with metadata describing how they connect; models are cities assembled from structures.

4. **Key objects every AI must know:**
   - `sim$objectName` — shared simulation environment (read/write objects here)
   - `P(sim)$paramName` — module parameters (read-only during events)
   - `mod$localVar` — module-local variables (private to the module)
   - `time(sim)` — current simulation time
   - `end(sim)` — simulation end time

5. **File map table** — Topic → which file to read next.

6. **Critical AI rules (never violate):**
   - Never use `library()` or `install.packages()` — use `Require()`
   - Never hardcode paths — use `inputPath(sim)`, `outputPath(sim)`, `modulePath(sim)`
   - Wrap all expensive/slow calls in `Cache()`
   - Always check `suppliedElsewhere()` before assigning object defaults in `Init`
   - Read `defineModule()` metadata before editing any module logic

---

### `core/module-anatomy.md`

**Purpose:** The most critical reference. Everything an AI needs to understand and correctly write a SpaDES module.

**Contents:**

1. **Module directory structure:**
   ```
   moduleName/
   ├── moduleName.R          # Main module file (metadata + events + functions)
   ├── moduleName.Rmd        # Human-readable documentation
   ├── R/                    # Additional R scripts sourced by the module
   ├── data/                 # Included data files
   │   └── CHECKSUMS.txt     # Expected checksums for data files
   ├── tests/
   │   └── testthat/         # Unit tests
   ├── citation.bib          # BibTeX citation for the module
   ├── LICENSE.md
   └── NEWS.md
   ```

2. **Annotated `defineModule()` skeleton** — every field explained:
   - `name`, `description`, `keywords`, `authors`
   - `childModules` — sub-modules this module manages
   - `version` — semantic versioning
   - `spatialExtent`, `timeframe`, `timeunit`
   - `citation`, `documentation`, `reqdPkgs`
   - `parameters` — `defineParameter()` calls with name, class, default, min, max, description
   - `inputObjects` — `expectsInput()` calls with objectName, objectClass, desc, sourceURL
   - `outputObjects` — `createsOutput()` calls with objectName, objectClass, desc

3. **`doEvent()` dispatcher pattern** — how events are routed; how to add a new event correctly.

4. **Standard event types:**
   - `init` — first event; sets up objects, schedules subsequent events
   - `plot` — visualisation; should be skippable
   - `save` — persistence; should be skippable
   - Custom events (e.g., `grow`, `disperse`, `disturb`)

5. **Common AI mistakes:**
   - Declaring objects in `createsOutput` that are never actually assigned to `sim$`
   - Using `expectsInput` for objects the module itself creates
   - Forgetting to `scheduleEvent()` recurring events at the end of each event function
   - Modifying `P(sim)$` (parameters are read-only)

---

### `core/simlist-events.md`

**Purpose:** How the simulation object works and how events are scheduled and executed.

**Contents:**

1. **`simList` structure** — an R environment containing all simulation state; every module shares it
2. **Accessor patterns** with examples: `sim$`, `P(sim)$`, `mod$`, `time(sim)`, `start(sim)`, `end(sim)`, `timeunit(sim)`, `moduleName(sim)`
3. **`simInit()`** — arguments: `times`, `params`, `modules`, `objects`, `paths`, `loadOrder`; what it validates; when it's called
4. **`spades()`** — executes the simulation; event queue mechanics
5. **`experiment2()`** — runs factorial simulation experiments
6. **`restartSpaDES()`** — resumes an interrupted simulation from current module code; invaluable during development
7. **Event scheduling pattern:**
   ```r
   scheduleEvent(sim, time(sim) + P(sim)$.plotInterval, "moduleName", "plot")
   ```
8. **Event priority** — how ties in event time are resolved

---

### `core/caching-reproducibility.md`

**Purpose:** SpaDES's caching system is central to PERFICT. AIs must understand when and how to use it.

**Contents:**

1. **`Cache()` mechanics** — wraps any function call; hashes inputs; returns cached result if inputs unchanged; stored in `options("reproducible.cachePath")`
2. **When to use `Cache()`** — any slow operation: data download, geoprocessing, model fitting, parameterisation
3. **What invalidates the cache** — input changes, algorithm changes, explicit `clearCache()`
4. **`prepInputs()`** — the canonical way to download, verify checksums, reproject, crop, and mask spatial inputs. Arguments: `url`, `targetFile`, `alsoExtract`, `destinationPath`, `studyArea`, `rasterToMatch`, `fun`
5. **`suppliedElsewhere()`** — returns `TRUE` if an object will be provided by another module or the user; use this in `Init` before assigning defaults so modules remain interoperable
6. **`reproducible` package philosophy** — same inputs + same algorithm = same output, always; caching enforces this

---

### `core/require-packages.md`

**Purpose:** SpaDES uses a non-standard package management system. AIs must never use base R package functions.

**Contents:**

1. **Why `Require()` not `library()`** — `Require()` installs if missing, handles version constraints, works from GitHub, and integrates with SpaDES's reproducibility guarantees
2. **`reqdPkgs` metadata field** — where to declare package dependencies in a module; SpaDES reads this to install packages before running
3. **Version pinning syntax** — `"packageName (>= 1.2.0)"`, GitHub sources `"PredictiveEcology/SpaDES.core@development"`
4. **`SpaDES.project::setupProject()`** — initialises a full workflow: creates directory structure, installs all module dependencies, sets paths. The recommended entry point for new projects.
5. **`SpaDES.install`** — helper package for managing SpaDES-specific installations

---

### `core/perfict-principles.md`

**Purpose:** Deep reference on the philosophical framework. Useful when an AI needs to justify design decisions or evaluate whether a proposed approach is PERFICT-compliant.

**Contents:**

For each of the 7 principles:
- What it means conceptually
- How SpaDES implements it technically
- What an AI should do to uphold it
- What an AI should avoid that would violate it

Includes the key insight from the papers: PERFICT is not just about code quality — it enables **nimbleness** (easy transfer to new study areas) and **iterative forecasting** (re-run as data updates).

---

### `workflows/LandR-biomass.md`

**Purpose:** Concrete, real-world example of a PERFICT workflow. Essential context for any AI working on LandR projects.

**Contents:**

1. **What LandR Biomass models** — spatiotemporally explicit forest succession: cohort growth, mortality, dispersal, responses to disturbance; derived from LANDIS-II LBSE but reimplemented in R/SpaDES

2. **The 5-module pipeline:**

   | Module | Type | Role |
   |--------|------|------|
   | `Biomass_speciesData` | Data | Downloads/prepares species percent cover inputs |
   | `Biomass_borealDataPrep` | Data/Calibration | Prepares landscape parameters for Western Canadian boreal |
   | `Biomass_speciesParameters` | Calibration | Estimates species growth traits (maxB, maxANPP) from forest inventory |
   | `Biomass_core` | Simulation | Runs pixel-based forest succession dynamics |
   | `Biomass_validationKNN` | Validation | Validates against KNN biomass snapshots (2001, 2011) |

3. **Key `sim$` objects passed between modules** — species tables, cohort biomass/age maps, pixel-level parameters, species traits

4. **LandR-specific patterns** — how `Biomass_core` uses cohort tables; the role of `maxB` and `maxANPP`; how disturbance modules plug in

5. **Common AI tasks on LandR projects:**
   - Changing study area (swap polygon, re-run data modules)
   - Disabling calibration (set `Biomass_speciesParameters` to NULL in module list)
   - Adding a disturbance module
   - Interpreting validation metrics (MAD, SNLL)

6. **Resource links** — GitHub repos, LandR manual URL

---

### `templates/module-context-template.md`

**Purpose:** A fill-in template that any module maintainer uses to create a `CLAUDE.md` for their individual module repo.

**Sections:**

1. **Module identity** — name, type (data/calibration/prediction/validation), one-paragraph purpose
2. **Core files map** — which shared core files to read (always include `module-anatomy.md`; optionally others)
3. **Inputs** — table of `expectsInput()` entries with objectName, class, description, which module provides it
4. **Outputs** — table of `createsOutput()` entries with objectName, class, description, which module consumes it
5. **Events** — list of events and what each does
6. **Non-obvious implementation details** — things an AI would get wrong without being told explicitly
7. **Known dependencies / integration notes** — upstream/downstream module contracts
8. **Active issues / gotchas** — current bugs, quirks, temporary workarounds
9. **Resource links** — GitHub repo, manual, associated workflow doc

---

## Usage Patterns

| Scenario | Files to include |
|----------|-----------------|
| Any SpaDES project (baseline) | `SPADES-AI.md` |
| Writing or modifying a module | + `core/module-anatomy.md` |
| Debugging caching/data issues | + `core/caching-reproducibility.md` |
| LandR Biomass project | + `workflows/LandR-biomass.md` |
| Working in a specific module repo | + that module's filled `module-context-template.md` |
| Full context for complex work | All of the above |

---

## Extension Pattern

To add a new workflow (e.g., `fireSense`):
1. Create `workflows/fireSense.md` following the same structure as `LandR-biomass.md`
2. Add an entry to the file map table in `SPADES-AI.md`
3. No other files need to change

To add a new module context:
1. Copy `templates/module-context-template.md` into the module repo
2. Fill in all sections
3. Add `# Core context` link pointing back to `spades.ai` shared files

---

## Out of Scope

- Tutorial/onboarding content for human users (the training book at predictiveecology.org covers this)
- Automated tooling to assemble context files (future work)
- Non-R languages (Python, C++) called from SpaDES modules (can be added as a core file later)
