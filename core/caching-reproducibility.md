# Caching and Reproducibility

SpaDES's reproducibility guarantee: **same inputs + same algorithm = same output, always.**
The `Cache()` function from the `reproducible` package enforces this by memoising function
calls on disk.

---

## `Cache()`

```r
result <- Cache(
  myFunction,
  arg1, arg2,
  cachePath = cachePath(sim),   # where to store results; defaults to getOption("reproducible.cachePath")
  userTags  = c("myTag")        # optional labels for cache management
)
```

**How it works:**
1. Hashes all inputs (function body + all arguments)
2. Checks the cache database at `cachePath` for a matching hash
3. If found: returns the cached result immediately (no computation)
4. If not found: runs the function, stores result + hash, returns result

**When to use `Cache()`:** Any call that is slow, downloads data, or does geoprocessing.
Typical targets: `prepInputs()`, `fitLM()`, `rasterize()`, model calibration functions.

```r
# Good — wrapping prepInputs in Cache
sim$speciesLayers <- Cache(
  prepInputs,
  url             = extractURL("speciesLayers", sim),
  targetFile      = "speciesLayers.tif",
  destinationPath = inputPath(sim),
  studyArea       = sim$studyArea,
  rasterToMatch   = sim$rasterToMatch,
  cachePath       = cachePath(sim),
  userTags        = c("speciesLayers", currentModule(sim))
)
```

---

## What Invalidates the Cache

- Any change to an input argument value
- Any change to the function body
- Explicit `clearCache(cachePath(sim), userTags = "myTag")`
- Explicit `clearCache(cachePath(sim))` — clears everything

**Implication:** If you change an algorithm inside a cached function, the cache will invalidate
automatically. You do not need to manually clear it (but you can with `clearCache()` if needed).

---

## `prepInputs()`

The canonical SpaDES way to download, verify, reproject, crop, and mask spatial inputs.

```r
sim$myRaster <- Cache(
  prepInputs,
  url             = "https://example.com/data.zip",
  targetFile      = "data.tif",          # file to extract from archive
  alsoExtract     = "data.tif.aux.xml",  # additional files to extract (optional)
  destinationPath = inputPath(sim),      # where to save downloaded files
  studyArea       = sim$studyArea,       # crop/mask to this polygon
  rasterToMatch   = sim$rasterToMatch,   # reproject/resample to match this raster
  fun             = terra::rast          # function to read the file (optional; inferred if omitted)
)
```

**What `prepInputs()` does automatically:**
- Downloads the file if not present
- Verifies checksum if `CHECKSUMS.txt` exists
- Extracts from zip/tar archives
- Reprojects to match `rasterToMatch` CRS
- Resamples to match `rasterToMatch` resolution
- Crops and masks to `studyArea`

Always wrap in `Cache()`. `prepInputs()` itself is not cached internally.

---

## `suppliedElsewhere()`

```r
if (!suppliedElsewhere("objectName", sim)) {
  sim$objectName <- defaultValue
}
```

Returns `TRUE` if `objectName` will be provided by:
- Another module's `createsOutput` declaration
- An object supplied directly in `simInit(objects = list(...))`

**Use in the `init` event function before creating default objects.** Without this check, a module that creates
a default value for `sim$studyArea` will overwrite the real study area provided by the user.

```r
Init <- function(sim) {
  # Only create a default study area if nothing else is providing one
  if (!suppliedElsewhere("studyArea", sim)) {
    sim$studyArea <- terra::vect(...)  # fallback default
  }
  # ...
}
```

---

## `reproducible` Package Philosophy

The `reproducible` package enforces the R principle that pure functions always return the
same output for the same input — but extends it to disk-based caching across R sessions.

Key options:
```r
options(
  reproducible.cachePath  = file.path("cache"),  # where Cache() stores results
  reproducible.useCache   = TRUE,                # set FALSE to bypass caching temporarily
  reproducible.overwrite  = TRUE                 # overwrite existing files in prepInputs
)
```

Set `reproducible.useCache = FALSE` during debugging when you want fresh results every run.
Reset to `TRUE` before committing.
