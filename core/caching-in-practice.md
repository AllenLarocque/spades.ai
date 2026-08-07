# `Cache()` in Practice — Making a No-Change Re-Run Free

A field companion to [`caching-reproducibility.md`](caching-reproducibility.md), which covers what
`Cache()` **is**. This covers why a correct-looking `Cache()` call still recomputes for hours, and
how to tell a cache that is merely wasteful from one that is quietly **wrong**.

Everything here was measured on a 31-module amplicon pipeline across five production runs, not
inferred. Verified against `reproducible` 3.1.1.9063. Net effect of applying it: a no-change re-run
went from **13 h to 3.1 h with 154 cache hits and 0 misses** — the residual is work that was never
cached at all, not cache failure.

---

## 1. The Invariant

**A re-run with no changes should be essentially free.**

That is a testable property, not an aspiration. When it fails you have a bug, not a fact of life.
Measure it before believing anything else:

```bash
grep -ciE 'loaded *!'  run.log     # cache HIT
grep -ciE 'saved *!'   run.log     # cache MISS
grep -inE 'saved *!'   run.log     # ...and WHICH function missed
```

A function under `Saved!` on **every** run and never under `Loaded!` is **Saved-never-Loaded**: it
is being cached and the cache is never used. That is the signature of an unstable key.

One measured production run:

| module | fn | status | cost, **every** run |
|---|---|---|---|
| read trimming | `removeAdaptersStep` | Saved-never-Loaded | 37 min |
| denoising | `denoiseReads` | Saved-never-Loaded | 47 min |
| phylo distance | `buildPdistBig` | Saved-never-Loaded | **~8 h** |

**9.4 of 12 h — 78% — recomputing byte-identical results.** None of it looked wrong in the code.

---

## 2. Failure Mode A — Non-Content in the Key (never hits)

`Cache()` hashes **every** captured argument. An argument that changes between runs but has nothing
to do with *what is computed* moves the key, and the cache can never hit.

The culprits are **locations and clocks**, not data:

- `SpaDES.core::scratchPath(sim)`, `tempdir()`, `tempfile()` — session-specific by construction
- absolute paths embedding a working directory, worktree, or container mount
- timestamps, run IDs, PIDs
- a connection, driver, or cluster handle

`scratchPath()` is the one that bites in SpaDES, because it looks like a project path and is not.
Three consecutive runs of the same pipeline recorded:

```
scratchPath = '/tmp/RtmphGiM75/myproject'
scratchPath = '/tmp/RtmpCFOr97/myproject'
scratchPath = '/tmp/RtmpdMegDb/myproject'
```

`Rtmp*` is R's per-session temp directory, regenerated every session.

```r
# BEFORE — the defect
params <- list(nullModel = P(sim)$nullModel, nRuns = P(sim)$nRuns,
               wd = file.path(SpaDES.core::scratchPath(sim), "myModule"),   # ⇠ poison
               cachePath = cachePath(sim))

res <- Cache(computeSomethingExpensive, sim$bigObject, params = params,
             cachePath = cachePath(sim), userTags = "myModule")
```

`params` is hashed **whole**, so a session path contaminates the key of an 8-hour computation.

**Note what is not wrong:** the module-level `Cache()` seam exists and is correctly placed. Only the
key is broken. **Reach for the key before reaching for more caching.**

### The diagnostic that settles it

Stop reasoning about which argument moved. `reproducible` will tell you:

```r
options(reproducible.showSimilar = TRUE)
```

> If `Cache` finds no identical archive, it reports the next most recent *similar* archive and
> **indicates which argument(s) is/are different**.

For a stubborn case, `debugCache = "complete"` attaches `debugCache1` (the raw `list(...)`) and
`debugCache2` (post-`.robustDigest`) to the result, so two runs can be diffed element by element.

---

## 3. Failure Mode B — A Key Too Weak (hits, and is WRONG)

The obvious fix for A is `omitArgs`. Read its contract first:

> `omitArgs`: a character vector of **argument names in `FUN`** to omit from the cache digest, or
> `TRUE` to omit *every* captured argument.

It names **top-level arguments only**. So on the code above, `omitArgs = "params"` would fix the
miss and simultaneously drop `nullModel` and `nRuns` from the key. Change the null model, silently
get the old result back.

**A wasteful cache costs hours; a wrong cache produces a fabricated finding no test is looking for.**

> ⚠️ **A stale *hit* is more dangerous than a miss.** One run died 24 h in on a 17-day-old
> working-directory descriptor — and dying was the *lucky* outcome; the alternative was silently
> analysing the wrong distance matrix.

### `omitArgs` can be worse than the bug it fixes

If the cached function returns **file paths** — a manifest, or an object carrying a working
directory — then exempting the path stabilises the **key** while the **files** still vanish with the
session. A hit then returns paths into a dead `/tmp`. The instability was *accidentally protective*.

Check what your function returns before reaching for `omitArgs`. If a path is anywhere in the return
value, `omitArgs` alone is not a fix.

---

## 4. The Fix — Content-Keyed Paths Under a Persistent Root

**Separate WHAT is computed from WHERE it is written.** The first is content and belongs in the key;
the second is plumbing. Then make the plumbing a *function of* the content:

```r
#' <cacheRoot>/work/<label>-<hash>, hash = md5 of the sorted, de-duplicated content
contentWorkDir <- function(cacheRoot, label, contentKey) {
  stopifnot(is.character(cacheRoot), length(cacheRoot) == 1L, nzchar(cacheRoot))
  if (grepl("[/\\\\]|\\.\\.", label)) stop("unsafe label: it becomes a directory name")
  if (!length(contentKey)) stop("need at least one content element to fingerprint")
  if (anyNA(contentKey))   stop("contentKey contains NA")

  # method = "radix" forces C-locale byte order. Plain sort() collates by SESSION LOCALE,
  # so identical content can hash differently on another machine and orphan a directory
  # already holding hours of computation.
  key <- paste(sort(unique(as.character(contentKey)), method = "radix"), collapse = "\n")
  # serialize = FALSE hashes the STRING. TRUE folds in R's serialization header, making
  # the hash a property of the R VERSION rather than of the content.
  hash <- substr(digest::digest(key, algo = "md5", serialize = FALSE), 1L, 8L)
  file.path(cacheRoot, "work", paste0(label, "-", hash))
}
```

```r
# AFTER — params carries science only; the path derives from content
params <- list(nullModel = P(sim)$nullModel, nRuns = P(sim)$nRuns)
res <- Cache(computeSomethingExpensive, sim$bigObject, params = params,
             cacheRoot = cachePath(sim), cachePath = cachePath(sim),
             userTags = "myModule")
# ...and inside, wd <- contentWorkDir(cacheRoot, "subsetA", taxaNames)
```

The path is now a pure function of content **already in the key**, so hashing it is redundant but
harmless. **No `omitArgs` anywhere, so no key is ever weakened** — the danger in §3 is designed out
rather than managed.

⚠️ Those three comments in `contentWorkDir` are not decoration. Each was lost by rewriting the
function from scratch and caught only by a test that pinned the expected hash **literally**. If you
already have a working fingerprint, **delegate to it; do not reimplement it.**

### Working files belong under `cachePath()`, not `outputPath()`

Derived, regenerable data should be cleared *with* the cache. If scratch outlives the cache, then
`clearCache()` followed by a re-run does **not** recompute — the module finds its old artifact and
announces it is "directly used". That is how the 24-hour failure above happened.

### The content key CHAINS — and that is load-bearing

Key each step on its **input manifest plus its own parameters**. Step 2 then hashes paths that
already encode step 1's content, so a change anywhere upstream propagates to every directory below
it. **No step needs to know how far upstream a change happened.**

⚠️ This only holds if the chain is unbroken. One step left on `scratchPath` re-poisons everything
downstream of it. Fingerprinting input *paths* is legitimate **only** because those paths are
themselves content-derived.

### A working directory can outlive its cache entry — keep the clear-on-miss

Counter-intuitively, content-keying does **not** remove the need to clear stale artifacts:

```r
buildExpensiveThing <- function(x, wd) {
  stale <- list.files(wd, pattern = "^artifact", full.names = TRUE)
  if (length(stale)) unlink(stale)          # runs ONLY on a cache MISS
  someTool(x, wd = wd)
}
Cache(buildExpensiveThing, x = x, wd = wd)
```

The working directory is not governed by any cache key, so it survives `clearCache()`. Afterwards
the content is unchanged, so the **same** directory is reached — still populated — while the cache
misses. A tool that refuses a dirty directory then hard-errors, and "clear the cache and recompute"
stops working. Clearing inside the cached closure runs only on a miss; on a hit the closure never
fires and the valid cached result is reused untouched.

---

## 5. `Cache()` and File-Backed Objects

It is easy to conclude from §3 that `Cache()` is blind to files. **It is not.** `reproducible` ships
real machinery: `Filenames()` (S4 methods for `Raster*`, `list`, `environment`, `data.table`,
`Path`) locates backing files, `linkOrCopy()` brings them into the cache repository, and
`.prepareOutput()` / `remapFilenames()` restore and re-point them on load. Large `terra`/`raster`
objects are a first-class use case.

The machinery dispatches **on class**, and a bare character string is not a file to it:

```r
writeLines("VERSION-A", f); a1 <- .robustDigest(f); b1 <- .robustDigest(asPath(f))
writeLines("VERSION-B", f); a2 <- .robustDigest(f); b2 <- .robustDigest(asPath(f))

identical(a1, a2)   # TRUE  — bare string: hashed by NAME, contents ignored
identical(b1, b2)   # FALSE — asPath():    hashed by CONTENT
```

`asPath()` marks a string with class `c("Path", "character")`, which dispatches the file-aware
digest. **Two tools, two intents** — picking the wrong one is the actual bug:

| the path is… | use | effect |
|---|---|---|
| **an input whose contents matter** (a reference DB, a config, a model on disk) | `asPath(p)` | key tracks file **contents**; edit the file and the cache correctly invalidates |
| **pure plumbing** (where to write working files) | content-derived path (§4) | stable across sessions, and safe to hash |
| a bare string, either way | — | ⚠️ **worst of both**: keyed by the *name*, blind to the *contents* |

That third row is the common defect: a session-derived path string moves the key every run **while
being blind to whether the files changed**. Unstable where it should be stable; blind where it
should be sensitive.

> ⚠️ **The real gap is third-party handles.** `Filenames()` recognises the spatial classes and
> containers of them; it has no method for, say, a `bigmemory` descriptor. Such a handle is an
> unmanaged reference — neither copied into the cache nor remapped on restore — so a hit can return
> a valid-looking object pointing at something deleted or *stale*. For those, **validate the target
> on load**:

```r
readBackedThing <- function(handle) {
  target <- file.path(handle$wd, handle$file)
  if (!file.exists(target))
    stop("backing file missing at '", target, "'. Recompute rather than attaching to ",
         "whatever is there.", call. = FALSE)
  attachIt(target)
}
```

---

## 6. Failure Mode C — The Function Mutates Global State

The subtlest of the three, and the hardest to see: **a cached function that mutates
process-global state makes cache keys depend on cache *history* rather than on content.**

`Cache()` digests the arguments **before** calling. So on a HIT the function never runs and the
state is untouched; on a MISS it runs and the state advances. Anything digested *afterwards* then
depends on whether an earlier call hit.

Measured on a real pipeline. `heatTreeSweep()` sweeps a `metacoder::Taxmap`, and sweeping mutates a
shared taxon-ID pool inside `metacoder`/`taxa`. Two fixtures built identically, only `A` swept:

```
before sweep:   A = 8a5e13ba   B = 8a5e13ba
after  sweep:   A = 8cd5c82a   B = 185d1099    <- B was NEVER passed to the sweep
fresh build C after the sweep = 185d1099       <- and new objects match B, not the original
```

The consequence in production, over five runs of a 31-module pipeline: **exactly one spurious cache
miss per run**, always in this function, always a *different* subset, **marching one position later
each run and never converging** — at ~25 min a time.

```
run 1:  d48cfba8  33bb5f0d  93638d22*MISS  f2fb101a        20b1bc17
run 2:  d48cfba8  33bb5f0d  93638d22       043bcffe*MISS   20b1bc17
run 3:  d48cfba8  33bb5f0d  93638d22       043bcffe        534ce4e9*MISS
```

Read run 2: subset 3 now *hits*, so the sweep never runs, so the global state is never advanced —
and subset 4, digested under the un-advanced state, no longer matches the key it was stored under.
It misses, computes, advances the state, and subset 5 hits again. Next run the boundary moves one
position later. Forever.

### How to recognise it

- exactly one miss per run, in the same function, on a rotating target
- every input provably unchanged, and the producer provably deterministic
- the miss **moves** rather than converging — the signature that separates this from a cold cache
- ⚠️ **two runs cannot distinguish this from inputs settling.** Run 2 looks like convergence. It
  takes a **third** run to see the pattern — or a scaled-down reproduction (below).

### The fix

Do not digest the contaminated object. Digest its **content**, and pin the key with `.cacheExtra`:

```r
Cache(heatTreeSweep, tm, params,
      omitArgs    = "daTaxmap",              # <- the FORMAL name, see the trap below
      .cacheExtra = taxmapContentKey(tm),    # coefficients + taxon names + edge list
      cachePath = cachePath(sim), userTags = "heatTree")
```

The content fingerprint is unaffected by any of it — identical for `A`, `B` and `C`, before and
after. **This is not the `omitArgs` trap of section 3.** There, omitting *drops* information and
weakens the key. Here the omitted argument is *replaced* by an equivalent stable fingerprint, and
`params` stays fully hashed. The test to apply before ever writing `omitArgs`:

> **Does `.cacheExtra` restore everything `omitArgs` removed?** If not, you are weakening the key.

⛔ **Cloning the argument does not help** and was tried first. The contaminated state is global, not
in the object — `B` moved without ever being passed in.

### ⚠️ `omitArgs` names the FORMAL, and fails silently

```r
heatTreeSweep <- function(daTaxmap, params) { ... }

Cache(heatTreeSweep, tm, params, omitArgs = "tm", ...)        # matches NOTHING
Cache(heatTreeSweep, tm, params, omitArgs = "daTaxmap", ...)  # correct
```

`omitArgs` matches **argument names in `FUN`**, not the caller's local variable. A name that matches
nothing is **not an error and produces no warning** — the argument stays in the digest and the fix
is silently inert. This exact mistake made a first attempt at the fix a no-op that looked correct.

Worth a test, because it is cheap and the failure is invisible:

```r
test_that("every omitArgs name is a real formal of the cached function", {
  src <- paste(readLines("myModule.R"), collapse = "\n")
  for (one in regmatches(src, gregexpr('omitArgs\\s*=\\s*"[^"]+"', src))[[1]]) {
    nm <- sub('.*"([^"]+)".*', "\\1", one)
    expect_true(nm %in% names(formals(myCachedFn)))
  }
})
```

### Reproduce it in seconds, not in days

A history-dependent cache bug needs three full runs to even become visible. Do not pay that. Two
objects with different content, swept in sequence, across three short sessions reproduces it exactly:

```
             BROKEN            FIXED
session 1    X MISS Y MISS     X MISS Y MISS   (cold, expected)
session 2    X HIT  Y MISS     X HIT  Y HIT    <- the diagnostic row
session 3    X HIT  Y HIT      X HIT  Y HIT
```

Session 2 is the whole test: `X` hits, so nothing runs, so the state is not advanced — and a broken
key makes `Y` miss. This ran in two minutes and caught the inert-`omitArgs` bug that a 3-hour
production run would have reported as success.

---

## 7. Determinism Is a Prerequisite

An unseeded stochastic step produces a different result every run. Downstream `Cache()` calls hash
that result, so **one unseeded step invalidates the entire chain below it**, no matter how clean
those keys are.

```r
est <- withr::with_seed(rngSeed, iNEXT::iNEXT(x, base = "size", level = size, nboot = nboot))
```

Seed anything bootstrapped, resampled, rarefied, or fitted with random starts, and expose the seed
as a module parameter so it is recorded. In the measured pipeline this single discipline turned a
5-hour module into a free cache hit.

---

## 8. Checklist

Before merging any `Cache()` call:

- [ ] Does every hashed argument describe **what** is computed? No temp dirs, clocks, or handles.
- [ ] Is every path either `asPath()`-marked (contents matter) or content-derived (plumbing)? A bare
      path string is almost always a bug.
- [ ] Are working files under `cachePath()`, so `clearCache()` clears them too?
- [ ] If you reached for `omitArgs`, does the function return any path? If so, stop — see §3.
- [ ] Does the function mutate anything outside its return value -- including package-global
      state? A cached function must be pure, or its keys become a function of cache history.
- [ ] If you used `omitArgs`, is every name a real **formal** of the cached function? It fails
      silently otherwise.
- [ ] Is every stochastic step inside seeded from a module parameter?
- [ ] Does the call carry `userTags` naming the module, so you can inspect and clear selectively?
- [ ] If the result contains a path, is the target validated on load?
- [ ] **Run it twice with no changes. The second run reports `Loaded!`.** If not, it is a bug — fix
      it now, because the cost recurs on every future run and compounds with pipeline length.

## 9. Anti-Patterns

| pattern | why it bites |
|---|---|
| `scratchPath()` / `tempdir()` in a hashed argument | Saved-never-Loaded forever |
| `omitArgs` naming a list that holds science parameters | silently returns results from the wrong settings |
| `omitArgs` on a function that returns file paths | hit returns paths into a vanished directory |
| a path passed as a bare string when its contents matter | key tracks the name, not the file — use `asPath()` |
| working files under `outputPath()` | survive `clearCache()`, so "clear and recompute" silently does not |
| reimplementing an existing fingerprint | drops locale/version/NA safeguards that are invisible until they bite |
| clearing the whole cache to fix one module | discards unrelated valid work — use `userTags` |
| `reproducible.useCache = FALSE` left on after debugging | every run pays full cost; looks like a cache bug |
| a cached function that mutates global state | keys depend on whether earlier calls hit; one rotating miss per run, never converging |
| `omitArgs` naming the caller's variable instead of the formal | matches nothing, no warning, fix is silently inert |
| concluding from TWO runs that a cache is stable | a history-dependent miss looks like convergence on run 2; you need three, or a reproduction |
| trusting a green test suite to catch this | keys are a runtime property of production paths; fixtures use temp dirs where the bug cannot appear |

---

## See Also

- [`caching-reproducibility.md`](caching-reproducibility.md) — what `Cache()` is, `prepInputs()`,
  `suppliedElsewhere()`
- [`perfict-principles.md`](perfict-principles.md) — **P**redict frequently depends on re-runs being
  cheap; this document is how that is kept true
