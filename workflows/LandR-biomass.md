# LandR Biomass Workflow

LandR Biomass is a spatiotemporally explicit forest succession model implemented in SpaDES.
It models cohort-level tree growth, mortality, and dispersal at the pixel level, derived
from the LANDIS-II Landscape Biomass Succession Extension (LBSE) but reimplemented entirely
in R/SpaDES for full reproducibility and flexibility.

GitHub org: https://github.com/PredictiveEcology (all LandR modules)
LandR manual: https://landr-manual.predictiveecology.org

---

## The 5-Module Pipeline

| Module | Type | Role |
|--------|------|------|
| `Biomass_speciesData` | Data | Downloads and prepares species percent cover rasters |
| `Biomass_borealDataPrep` | Data + Calibration | Prepares landscape parameters for Western Canadian boreal |
| `Biomass_speciesParameters` | Calibration | Estimates species growth traits from forest inventory data |
| `Biomass_core` | Simulation | Runs pixel-based forest succession dynamics |
| `Biomass_validationKNN` | Validation | Validates against KNN biomass snapshots (2001, 2011) |

### Module order and dependencies

```
Biomass_speciesData
    ↓ sim$speciesLayers (species % cover rasters)
Biomass_borealDataPrep
    ↓ sim$cohortData, sim$pixelGroupMap, sim$speciesEcoregion, sim$species
Biomass_speciesParameters
    ↓ sim$species (updated with maxB, maxANPP, growthcurve)
Biomass_core  ←──── runs the succession simulation
    ↓ sim$cohortData (annual biomass/age per cohort per pixel)
Biomass_validationKNN
    ← reads sim$cohortData; produces validation metrics
```

---

## Key `sim$` Objects

| Object | Class | Produced by | Consumed by |
|--------|-------|-------------|-------------|
| `sim$studyArea` | `SpatVector` | User / control script | All data modules |
| `sim$rasterToMatch` | `SpatRaster` | User or `Biomass_borealDataPrep` | All data modules |
| `sim$speciesLayers` | `SpatRaster` (stack) | `Biomass_speciesData` | `Biomass_borealDataPrep` |
| `sim$species` | `data.table` | `Biomass_borealDataPrep` | `Biomass_core`, `Biomass_speciesParameters` |
| `sim$speciesEcoregion` | `data.table` | `Biomass_borealDataPrep` | `Biomass_core` |
| `sim$cohortData` | `data.table` | `Biomass_borealDataPrep` | `Biomass_core`, `Biomass_validationKNN` |
| `sim$pixelGroupMap` | `SpatRaster` | `Biomass_borealDataPrep` | `Biomass_core`, `Biomass_validationKNN` |
| `sim$biomassMap` | `SpatRaster` | `Biomass_core` | `Biomass_validationKNN`, user |

### `sim$cohortData` structure

The central data structure. One row per cohort per pixel group.

| Column | Type | Description |
|--------|------|-------------|
| `pixelGroup` | integer | Links to `pixelGroupMap` |
| `speciesCode` | factor | Species identifier (links to `sim$species`) |
| `age` | integer | Cohort age in years |
| `B` | integer | Biomass in g/m² |
| `mortality` | integer | Mortality this timestep |
| `aNPPAct` | integer | Actual ANPP this timestep |

### `sim$species` — key columns

| Column | Description |
|--------|-------------|
| `speciesCode` | Factor key |
| `maxB` | Maximum biomass (g/m²) — caps cohort growth; estimated by `Biomass_speciesParameters` |
| `maxANPP` | Maximum net primary productivity (g/m²/yr) — sets potential growth rate; estimated by `Biomass_speciesParameters` |
| `longevity` | Maximum species age |
| `mortalityshape` | Mortality curve shape parameter |
| `growthcurve` | Growth curve exponent |

`Biomass_core` uses `maxB` and `maxANPP` each year: actual ANPP is calculated as a fraction of `maxANPP`
scaled by current biomass vs. `maxB` and age vs. `longevity`. When cohort biomass approaches `maxB`,
growth is suppressed and mortality increases. These values are the primary levers for species-specific
succession dynamics — wrong values produce silently plausible-looking but ecologically incorrect outputs.

---

## LandR-Specific Patterns

### Pixel groups

`Biomass_core` uses a pixel-group optimisation: pixels with identical species/age/biomass
compositions are grouped. Succession runs once per pixel group, not per pixel. `pixelGroupMap`
maps pixels to their group. When cohort composition changes (e.g., after disturbance), pixels
are re-assigned to new groups.

**AI implication:** Do not iterate over pixels in LandR modules — iterate over pixel groups
in `cohortData`. Pixel-group logic is managed by `Biomass_core`; disturbance modules should
update `cohortData` and set `sim$rstCurrentBurn` (or equivalent), not pixel maps directly.

### Adding a disturbance module

Disturbance modules (e.g., `fireSense_Burn`) plug in by:
1. Writing a `sim$rstCurrentBurn` raster (1 = burned, NA = not burned) each year
2. `Biomass_core` reads `rstCurrentBurn` during its `Biomass` event and kills affected cohorts

No changes to `Biomass_core` are needed to add a disturbance module.

---

## Common AI Tasks

### Change the study area

1. Replace `sim$studyArea` with a new `SpatVector` polygon
2. Delete or clear the cache (`clearCache(cachePath(sim))`) — all spatial inputs need reprocessing
3. Re-run from `simInit()` — data modules will redownload and reprocess for the new extent

### Disable calibration

Remove `Biomass_speciesParameters` from the module list in `simInit()`. You must then supply
`sim$species` with pre-populated `maxB` and `maxANPP` columns, or use defaults from
`Biomass_borealDataPrep`.

### Skip validation

Remove `Biomass_validationKNN` from the module list. No other modules depend on it.

### Interpret validation metrics

`Biomass_validationKNN` outputs:
- **MAD**: Mean Absolute Deviation of predicted vs. observed biomass — lower is better
- **SNLL**: Spatially-explicit Negative Log Likelihood — lower is better; accounts for spatial autocorrelation

---

## Typical Control Script

```r
library(SpaDES.project)  # bootstrap exception: library() is permitted ONLY here, before Require is available

out <- setupProject(
  name    = "LandR_run",
  paths   = list(projectPath = "~/LandR_project"),
  modules = c(
    "PredictiveEcology/Biomass_speciesData@main",
    "PredictiveEcology/Biomass_borealDataPrep@main",
    "PredictiveEcology/Biomass_speciesParameters@main",
    "PredictiveEcology/Biomass_core@main",
    "PredictiveEcology/Biomass_validationKNN@main"
  ),
  params = list(
    Biomass_core = list(
      .plotInitialTime = NA,
      .saveInitialTime = NA
    )
  ),
  objects = list(
    studyArea     = myStudyArea,      # SpatVector polygon
    rasterToMatch = myTemplateRaster  # SpatRaster defining resolution/CRS
  ),
  times = list(start = 2001, end = 2031)
)

mySim <- do.call(simInit, out)
mySim <- spades(mySim)
```
