# PERFICT Principles — Deep Reference

Source papers:
- McIntire et al. 2022 — *PERFICT: A reuseable and flexible framework for iterative forecasting and reanalysis*
- Barros et al. 2023 — *Empowering ecological modellers with a PERFICT workflow*

PERFICT is not a code style guide — it is a scientific philosophy that determines how ecological
forecasting systems should be built so they remain nimble (easy to transfer to new study areas)
and enable iterative forecasting (re-run as data updates without starting from scratch).

---

## P — Predict Frequently

**Concept:** Models should be run repeatedly — not once for a publication. Frequent re-runs allow
updating forecasts as new data arrives, catching model drift, and building confidence in outputs.

**SpaDES implementation:** Caching (`Cache()`) makes re-runs cheap. If inputs haven't changed,
cached results are returned instantly. Only changed computations re-execute.

**AI should:**
- Wrap every slow operation in `Cache()`
- Structure workflows so a fresh `spades()` call re-runs the full pipeline correctly
- Never assume a model will only be run once

**AI should avoid:**
- Saving intermediate results to ad-hoc files and reading them back manually
- Hard-coding "skip this step if file exists" logic — use `Cache()` instead

---

## E — Evaluate

**Concept:** Predictions must be evaluated against real-world data, not just checked for internal
consistency. Validation is a first-class part of the workflow, not an afterthought.

**SpaDES implementation:** Validation modules (e.g., `Biomass_validationKNN`) plug into the same
pipeline and produce standardised metrics. They use out-of-sample data that was not used for fitting.

**AI should:**
- Include or preserve validation modules when modifying a pipeline
- Not remove `expectsInput` entries for validation objects when "simplifying" a module
- Keep validation code in a separate module from prediction code

**AI should avoid:**
- Commenting out validation steps to speed up development runs (set `.plotInitialTime = NA` in params to disable plotting; remove the validation module from the module list to skip validation entirely)
- Treating validation metrics as optional output

---

## R — Reusable

**Concept:** A module written for one study area should work in another with no code changes —
only data and parameters change.

**SpaDES implementation:**
- Study area is passed as a `sim$` object (typically a polygon), not hardcoded
- All paths use `inputPath(sim)`, `outputPath(sim)`, etc.
- `prepInputs()` downloads, reprojects, and crops data to whatever study area is provided

**AI should:**
- Always use SpaDES path accessors — never `file.path("data", "myFile.tif")`
- Parameterise anything that might vary by study area
- Use `suppliedElsewhere()` in the `init` event function before assigning object defaults

**AI should avoid:**
- Hardcoded CRS strings (derive from `sim$studyArea` or `sim$rasterToMatch`)
- Study-area-specific logic inside module event functions

---

## F — Freely Accessible

**Concept:** Data and code should be openly available and linkable so others can reproduce and
extend the work. FAIR principles: Findable, Accessible, Interoperable, Reusable.

**SpaDES implementation:**
- `prepInputs(url = "https://...", targetFile = "...", destinationPath = inputPath(sim))` downloads data from canonical URLs
- Checksums in `CHECKSUMS.txt` verify data integrity
- Modules are published on GitHub; workflows link to them by URL

**AI should:**
- Use `prepInputs()` with a `url` argument for all external data sources
- Populate `sourceURL` in `expectsInput()` metadata when a canonical URL is known
- Include `CHECKSUMS.txt` entries for data files

**AI should avoid:**
- Loading data from local paths that won't exist on another machine
- Using data without a documented provenance URL

---

## I — Interoperable

**Concept:** Modules connect by contract. Any module that produces object X can be substituted
for another that produces the same X, as long as the class and structure match.

**SpaDES implementation:**
- `expectsInput(objectName, objectClass, description, sourceURL)` — declares what a module needs
- `createsOutput(objectName, objectClass, description)` — declares what a module produces
- SpaDES validates these contracts at `simInit()` time and warns on mismatches

**AI should:**
- Read `defineModule()` metadata to understand the contract before editing any module
- Update `createsOutput` when adding a new `sim$` assignment
- Update `expectsInput` when adding a new `sim$` read
- Keep objectClass accurate — `SpatRaster` ≠ `RasterLayer`

**AI should avoid:**
- Declaring objects in `createsOutput` that the module never actually assigns to `sim$`
- Reading `sim$objects` that are not declared in `expectsInput`
- Changing object names without updating all downstream module metadata

---

## C — Continuous

**Concept:** The full workflow runs end-to-end from a single script with no manual intervention.
No "run step 1, then manually copy the file, then run step 2."

**SpaDES implementation:**
- `simInit()` + `spades()` is the entire entry point
- Data download, preprocessing, simulation, validation, and output saving all happen inside modules
- `SpaDES.project::setupProject()` handles environment setup before `simInit()`

**AI should:**
- Keep all pipeline logic inside module event functions or the control script
- Use `saveSimList()` / `loadSimList()` for checkpointing, not manual file saves:
  ```r
  saveSimList(mySim, filename = file.path(outputPath(mySim), "checkpoint.qs"))
  mySim <- loadSimList(file.path(outputPath(mySim), "checkpoint.qs"))
  ```
- Ensure every workflow can be re-run with a single `source("main.R")`

**AI should avoid:**
- Instructions like "run `prepare_data.R` first, then run `model.R`"
- Writing modules that require a human to move files between steps

---

## T — Tested

**Concept:** Automated tests verify that modules and workflows produce correct output and catch
regressions when code changes.

**SpaDES implementation:**
- `tests/testthat/` directory in each module
- `SpaDES.core::testInit()` sets up a minimal `simList` for unit testing a single module:
  ```r
  # In tests/testthat/test-myModule.R
  test_that("init creates expected output", {
    sim <- testInit("myModule",
                    params = list(myModule = list(paramA = 5)),
                    objects = list(studyArea = testStudyArea))
    sim <- spades(sim, events = "init")
    expect_s3_class(sim$outputObjectName, "data.table")
  })
  ```
- Integration tests run the full `simInit()` + `spades()` on a small test study area

**AI should:**
- Write `testthat` tests for new functions added to a module
- Use `SpaDES.core::testInit()` for module-level tests rather than constructing `simList` manually
- Test both the happy path and edge cases (empty study area, missing optional inputs)

**AI should avoid:**
- Skipping tests when "just fixing a small thing"
- Writing tests that pass by hardcoding expected output instead of computing it
