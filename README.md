# spades.ai

AI context documents for the [SpaDES](https://spades.predictiveecology.org) R ecosystem.

This repository provides a modular set of markdown files that give AI coding assistants (Claude, Copilot, Cursor, etc.) the domain knowledge needed to write, modify, and debug code in the SpaDES R meta-platform — without needing to be told the basics every session.

---

## What's in here

```
SPADES-AI.md                          # Start here — use as CLAUDE.md / AGENTS.md
core/
  perfict-principles.md               # The 7 PERFICT principles in depth
  module-anatomy.md                   # How to write and read a SpaDES module
  simlist-events.md                   # simList, simInit(), spades(), event scheduling
  caching-reproducibility.md          # Cache(), prepInputs(), suppliedElsewhere()
  require-packages.md                 # Package management with Require
workflows/
  LandR-biomass.md                    # LandR Biomass forest succession pipeline
templates/
  module-context-template.md          # Fill-in template for per-module CLAUDE.md
```

---

## How to use

### Option A — Drop into any SpaDES project

Copy `SPADES-AI.md` into your project root as `CLAUDE.md` (Claude) or `AGENTS.md` (other tools). The file is self-contained (~80 lines) and links to the other files for deeper context.

### Option B — Full context for complex work

Copy the files you need alongside your project code, or reference them by URL. The file map in `SPADES-AI.md` tells you which file to reach for based on what you're doing:

| Task | File |
|------|------|
| Writing or editing a module | `core/module-anatomy.md` |
| Debugging caching or data issues | `core/caching-reproducibility.md` |
| Understanding event scheduling | `core/simlist-events.md` |
| Working on a LandR Biomass project | `workflows/LandR-biomass.md` |
| Creating a module-level CLAUDE.md | `templates/module-context-template.md` |

### Option C — Per-module context

Use `templates/module-context-template.md` to create a `CLAUDE.md` inside an individual module repository. Fill in the inputs, outputs, events, and gotchas specific to that module. Reference the shared `spades.ai` core files from there.

---

## Adding a new workflow

1. Create `workflows/<name>.md` following the structure of `workflows/LandR-biomass.md`
2. Add an entry to the file map table in `SPADES-AI.md`
3. Done — no other files need to change

---

## Background

SpaDES is built on the [PERFICT](https://besjournals.onlinelibrary.wiley.com/doi/10.1111/2041-210X.13874) framework for reproducible, iterative ecological forecasting. These documents are not tutorials for human users — the [Predictive Ecology training book](https://predictiveecology.org) covers that. They are reference material for AI assistants being asked to work in SpaDES codebases.

Primary SpaDES GitHub org: https://github.com/PredictiveEcology
