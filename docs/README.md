# Health App — Planning Docs

Planning-phase research and design for a **combined workout tracker + nutrition tracker** —
essentially Strong and MacroFactor merged into one app, built for personal use, on iOS and Android.

Nothing here is code. This is the research that makes the code obvious.

---

## Reading order

If you read three of these, read **01**, **08**, and **09**.

| # | Doc | What it answers |
|---|-----|-----------------|
| [01](01-strong-app-breakdown.md) | **Strong teardown** | How a great workout logger actually works — data model, the logging screen, PR/1RM math, what Strong gets wrong |
| [02](02-calorie-trackers-breakdown.md) | **MacroFactor / MyFitnessPal teardown** | The adaptive-TDEE algorithm in full, weight-trend smoothing, why "eat back your exercise calories" is broken, food-logging UX |
| [03](03-cross-platform-approaches.md) | **iOS + Android simultaneously** | React Native, Flutter, KMP, and full-native compared on the axes that matter *for this app*; the iOS-first fallback; a 1-week spike protocol to decide |
| [04](04-productionization-and-deployment.md) | **Shipping it** | Personal builds (TestFlight / sideload) today, App Store + Play Store later; signing, CI, privacy manifests, review landmines for health apps |
| [05](05-health-integrations.md) | **HealthKit & Health Connect** | Both APIs in depth, the concept→type mapping table, incremental sync, deduplication, the abstraction layer to write |
| [06](06-data-architecture.md) | **Databases & sync** | Local-first SQLite schema (full DDL), the sync engine, backend options, the timezone/units/immutability traps |
| [07](07-food-database.md) | **Food data** | Open Food Facts + USDA ingestion pipeline, licensing, search, barcodes, portion normalization, quality heuristics |
| [08](08-combined-app-design.md) | **The product** | What the merged app *is*, screen by screen; the insights only a combined app can produce; the MVP cut and the non-goals |
| [09](09-execution-plan.md) | **The plan** | Phased build plan mapped onto the Superpowers skills, with review gates and what's parallelizable |
| [10](10-decisions-log.md) | **Decisions & open questions** | What's locked, what's open, what needs your input |

### Mapping to the original request

Your items 1–7 map to docs 01, 02, 03, 04, 05, 08, 09. Three docs were added because
they're each big enough to sink the project if handled casually:

- **06 (databases & sync)** — split out of your item 5. Sync is where local-first apps die.
- **07 (food database)** — the single largest hidden cost in any calorie tracker.
- **10 (decisions log)** — so the open stack decision doesn't get silently made by accident.

---

## Decisions already made

From the planning conversation on 2026-08-17:

- **Distribution** — personal use now (TestFlight / sideloaded APK), public App Store later.
  Every design decision keeps that door open without paying for it upfront.
- **Data** — local-first. SQLite on device is the source of truth; cloud sync exists so
  years of logs survive a dead phone and so iPhone + Android can both be used.
- **Food data** — Open Food Facts + USDA FoodData Central, self-hosted. No commercial
  API fees, no licensing that blocks personal use.
- **Stack** — **deliberately open.** Doc 03 compares all four approaches; doc 09 Phase 0
  is a timeboxed spike that closes it with evidence rather than vibes.

See [10-decisions-log.md](10-decisions-log.md) for the full log and the open questions.

---

## Conventions in these docs

- **⚠️ Verify** marks a fact that was accurate at time of writing but is the kind of thing
  that drifts — pricing, API deprecation dates, store policy. Check it before you depend on it.
- Code is illustrative pseudo-code or SQL unless a doc says otherwise. None of it is
  copy-paste ready, and it shouldn't be — the stack isn't chosen yet.
- Energy is **kcal**, mass is **kg**, duration is **seconds** everywhere in these docs.
  Display units are a presentation concern; see doc 06.
