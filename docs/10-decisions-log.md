# 10 — Decisions and Open Questions

A running record so decisions don't get silently remade, and so the open ones don't get
silently made by accident.

**Status legend:** ✅ decided · 🟡 recommended, not yet confirmed · ❓ open

---

## Decisions

### ✅ D1 — Distribution: personal now, public later
*Decided 2026-08-17.*
Tier 0 (TestFlight + sideloaded APK) is the near-term target; the App Store stays an option.
**Consequence:** design keeps the door open — privacy-clean data handling, no architecture
that assumes a single user *identity*, algorithm guardrails that double as review defence —
but pays nothing upfront for it. See [doc 04](04-productionization-and-deployment.md).

### ✅ D2 — Local-first with cloud sync
*Decided 2026-08-17.*
SQLite on device is the source of truth. Backend exists for durability and multi-device.
**Consequence:** every syncable table carries `id`/`updated_at`/`deleted_at`/`server_seq`
from migration `0001`, even though the sync engine isn't built until Phase 7. Adding those
columns later is a painful migration; adding them now is free.
See [doc 06](06-data-architecture.md).

### ✅ D3 — Food data: Open Food Facts + USDA, self-hosted
*Decided 2026-08-17. Refined during research.*
No commercial API, no licence that blocks personal use.
**Refinement:** v1 ships bundled USDA (Tier 1) + the **public OFF API** for barcodes +
a permanent local cache (Tier 3). The self-hosted mirror is deferred to Phase 4c and built
only if offline scanning, brand search, or rate limits demand it. Same pipeline either way —
this is a sequencing change, not a change of direction. See [doc 07](07-food-database.md) §2.

### ✅ D4 — One combined app, one user
Not two apps with a shared backend. The whole value is in the cross-domain insights
([doc 08](08-combined-app-design.md) §1), which need one database.
**Consequence:** no multi-tenancy, no permissions model, no onboarding funnel, no
subscription infrastructure. A large amount of work simply removed.

### ✅ D5 — Wearables only via the platform health stores
No direct Whoop/Oura/Garmin/Withings API integrations. They all write to HealthKit and
Health Connect, so two integrations get you all of them — no OAuth, no rate limits, no
partner agreements. See [doc 05](05-health-integrations.md) §4.

### ✅ D6 — Canonical units: kg · m · s · g · kcal · mL
Stored canonically, converted at the edges. Display units are a user preference.
See [doc 06](06-data-architecture.md) Trap 3.

### 🟡 D7 — Expenditure via Kalman filter over [weight, TDEE]
Recommended over a rolling-window estimator: handles missing data natively, produces
uncertainty, self-tunes its smoothing. ~100 lines of pure testable code.
**Confirm during Phase 2** by validating against your own historical data.
See [doc 02](02-calorie-trackers-breakdown.md) §2.

### 🟡 D8 — e1RM via Brzycki, capped at 10 reps
Recommended. The formula matters far less than never changing it, since the trend is what
you read. Capping avoids fantasy numbers from high-rep sets.
See [doc 01](01-strong-app-breakdown.md) §4.

### 🟡 D9 — Sync: outbox + last-write-wins + server sequence cursor
Recommended. One user means conflicts are rare and LWW is genuinely acceptable. No CRDTs.
**Confirm at Phase 7.** See [doc 06](06-data-architecture.md) §4.

### 🟡 D10 — Backend: Supabase
Recommended. Postgres (so the schema transfers verbatim), RLS, auth, free tier with vast
headroom for one user, self-hostable escape route.
**Confirm at Phase 7.** See [doc 06](06-data-architecture.md) §5.

---

## Open questions

Ordered by how much they block. The first four affect Phase 0, which is next.

### ❓ O1 — The stack *(blocks everything)*
**Deliberately unresolved.** Closed by the Phase 0 spike with evidence, not by argument.
See [doc 03](03-cross-platform-approaches.md) §3.

### ❓ O2 — Which two stacks to spike? *(blocks Phase 0)*
Recommendation: **Flutter vs React Native + Expo**. Both genuinely deliver two platforms from
one codebase, both are fast to evaluate, and the comparison is close. Add KMP as a third only
if both feel wrong on the logging screen.
*Override this if you already have strong experience in one of them — existing fluency is
worth more than any row in the comparison table.*

### ❓ O3 — Which devices do you actually own? *(affects Phase 0 and testing)*
The spike must run on your real hardware, and "oldest device I own" is the performance bar.
Specifically: iPhone model, Android model, and whether both are in daily use or one is a
test device.

### ❓ O4 — Does a watch app matter to you? *(shifts the stack weighting)*
Currently weighted at 1/5 in the comparison table, treated as a v2 concern. If logging sets
from an Apple Watch is something you'd genuinely use, that weight should go up sharply and
it favours KMP or native — no cross-platform framework can build a watchOS UI.

### ❓ O5 — What history do you have to import? *(affects Phases 3, 4, 5)*
Strong CSV? MyFitnessPal or MacroFactor export? Years of it, or months?
This matters more than it sounds: importing real history makes every chart useful from day
one, and it's the only way to validate the expenditure algorithm against reality in Phase 5.

### ❓ O6 — Do you have a smart scale? *(affects Phase 6's value)*
A scale that writes to HealthKit/Health Connect is what makes daily weigh-ins actually
happen, and daily weigh-ins are what make the adaptive algorithm work. If you don't have
one, Phase 6 is worth less and a good manual weigh-in flow is worth more.
(If you're going to buy one: any model that syncs to Apple Health / Health Connect works —
Withings, Renpho, Eufy, Garmin.)

### ❓ O7 — Which countries for the food database? *(affects Phase 4a)*
Filtering Open Food Facts by `countries_tags` is the main lever on data size. Where do you
actually buy groceries?

### ❓ O8 — Display units: kg or lb? *(cosmetic, but affects seeded data)*
Storage is kg regardless. Affects default increments (2.5 kg vs 5 lb) and the seeded
exercise library's defaults.

### ❓ O9 — Do you want a programming engine? *(scope decision, v1 vs v2)*
Currently a v2 feature. It's the biggest genuine gap in Strong
([doc 01](01-strong-app-breakdown.md) §5) — percentage-based targets, week-over-week
progression, 5/3/1-style templates. If you run structured programming and currently maintain
it in a spreadsheet, this may deserve promotion into v1.0. If you train more intuitively,
leave it in v2 forever.

### ❓ O10 — Is public release a real intention or just optionality?
Currently treated as optionality, which costs almost nothing. If it's a real intention,
two things change: the ODbL share-alike obligation on Open Food Facts data becomes a
decision to make deliberately ([doc 07](07-food-database.md) §1), and the Play Store's
12-testers-for-14-days wall should be started far earlier than Phase 9
([doc 04](04-productionization-and-deployment.md) §4).

---

## Things deliberately *not* decided yet

Listed so their absence doesn't read as an oversight:

- **Visual design and theming.** Deferred until after the Phase 0 spike, since the stack
  shapes what's cheap. Doc 08 specifies information architecture, not aesthetics.
- **Exercise library contents.** ~200 seeded exercises; the actual list is a Phase 3 data task.
- **Notification strategy.** Rest timer only, at first. Anything beyond that risks violating
  the no-nagging principle ([doc 08](08-combined-app-design.md) §2).
- **Whether to keep paying for Strong/MacroFactor during the build.** Yes, obviously — until
  each phase's daily-driver gate passes.

---

## How to use this file

Update it when a decision is made or reversed. Add a dated line; don't rewrite history.
When a 🟡 gets confirmed in its phase, flip it to ✅ and note what confirmed it. When a
decision turns out wrong, add a new entry that supersedes the old one rather than editing it
away — the reasoning that led somewhere wrong is worth keeping.
