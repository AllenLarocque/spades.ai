# [Module Name] — AI Context

> **How to use this file:** Copy to the module's repository as `CLAUDE.md`. Fill in all
> sections. The core files listed below should be read first to establish SpaDES fundamentals;
> this file adds module-specific context on top.

---

## Module Identity

**Name:** `moduleName`
**Type:** [data | calibration | prediction | validation]
**One-line purpose:** What this module does in one sentence.

**Purpose (paragraph):**
Expand on what this module does, why it exists in the pipeline, and what ecological or
computational process it represents. Include the scientific context if relevant.

---

## Core Context — Read First

Always read these shared files before working in this module:

- `core/module-anatomy.md` — how SpaDES modules are structured (required)
- `core/caching-reproducibility.md` — Cache(), prepInputs(), suppliedElsewhere()
- `core/simlist-events.md` — simList accessors, event scheduling
- `workflows/LandR-biomass.md` — [include if this module is part of LandR]

---

## Inputs

Objects this module reads from `sim$`. All are declared in `expectsInput()` in `defineModule()`.

| Object | Class | Description | Provided by |
|--------|-------|-------------|-------------|
| `studyArea` | `SpatVector` | Study area polygon | User / control script |
| `rasterToMatch` | `SpatRaster` | Template raster defining CRS and resolution | User or upstream module |
| `inputObjectName` | `data.table` | Description of what this object contains | `upstreamModuleName` |

---

## Outputs

Objects this module assigns to `sim$`. All are declared in `createsOutput()` in `defineModule()`.

| Object | Class | Description | Consumed by |
|--------|-------|-------------|-------------|
| `outputObjectName` | `SpatRaster` | Description of what this object contains | `downstreamModuleName` |

---

## Events

| Event | When | What it does |
|-------|------|-------------|
| `init` | Start | Downloads inputs, prepares objects, schedules recurring events |
| `[eventName]` | Every [interval] years | Brief description of what happens |

---

## Cost & Scaling

How this module's wall time and peak memory scale with the drivers that actually move them —
typically the size of the primary data object (e.g. `ntaxa`, `ncell`, `nsamples`), the number of
subsets or strata it loops over, and any replicate/permutation parameter.

**Give the exponent, and label every figure `MEASURED` or `ESTIMATED`.** A figure extrapolated
from a single scale point is a guess wearing a number's clothing: one point is consistent with
every exponent. Two points fix it. This distinction is the whole value of the section — a module
that is comfortable at one scale and impossible at another looks identical in its metadata.

| Driver | Time | Peak memory | Anchor |
|--------|------|-------------|--------|
| `ntaxa` | O(n²) | O(n²) — distance matrix materialized in RAM | `MEASURED` 12,923 → 2.6 h, 1.3 GB |
| `nsamples` | O(n) | O(1) | `MEASURED` 299, included above |
| `nRuns` | O(n) | O(1) | `MEASURED` 1000, included above |

**Hard limits.** Any scale at which this module *fails* rather than merely slows — an allocation
exceeding RAM, a file-size ceiling, an integer overflow, a dependency's own cap. State the
crossover point and what the failure looks like, so a caller can guard it instead of discovering
it mid-run.

**Cheap knobs.** Parameters that trade accuracy or completeness for time, and roughly what each
one buys.

> If the scaling is genuinely unmeasured, write `UNMEASURED` and say what it would take to find
> out. That is honest and actionable. A plausible-looking guess is neither.

---

## Non-Obvious Implementation Details

Things an AI would get wrong without being explicitly told. Be specific.

- **[Detail 1]:** Explanation of a gotcha, non-obvious design decision, or thing that looks
  wrong but is intentional.
- **[Detail 2]:** e.g., "The `cohortData` table is keyed on `(pixelGroup, speciesCode, age)` —
  any merge must preserve this key or downstream modules will silently produce wrong results."
- **[Detail 3]:** e.g., "Parameters `.plotInitialTime` and `.plotInterval` are intentionally
  set to `NA` in production runs — do not 'fix' them."

---

## Known Dependencies / Integration Notes

How this module fits into the broader pipeline.

- **Upstream:** [Module A] must run before `init` because it creates `sim$requiredObject`.
- **Downstream:** [Module B] reads `sim$outputObject`; its class must remain `SpatRaster`
  with the same CRS as `rasterToMatch`.
- **Optional dependency:** If `sim$optionalObject` is not present, this module creates a
  default. Supply it via `simInit(objects = list(...))` to override.

---

## Active Issues / Gotchas

Current bugs, temporary workarounds, or known quirks. Update as issues are resolved.

- **[Issue 1]:** Description. Workaround: [what to do].
- **[Issue 2]:** Description. Tracked at: [GitHub issue URL].

---

## Resource Links

- **Module repo:** https://github.com/PredictiveEcology/moduleName
- **Workflow doc:** `workflows/LandR-biomass.md` [or other relevant workflow]
- **Issue tracker:** https://github.com/PredictiveEcology/moduleName/issues
- **Manual:** https://landr-manual.predictiveecology.org [if applicable]
