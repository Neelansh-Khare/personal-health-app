# 05 — HealthKit and Health Connect

Both platforms ship a system-level health data store. Your app reads from it, writes to it,
and — crucially — is **not** stored in it. This doc covers both APIs in depth, the mapping
between them, and the abstraction to put in front of them.

---

## 0. The governing principle

> **The platform health store is an integration surface, not your database.**

Your SQLite database is the source of truth for everything the app cares about: sets, reps,
RPE, food entries, portions, routines, targets. None of that is representable in HealthKit,
and only some of it in Health Connect.

What the health store is *for*:

- **Reading data you don't produce** — bodyweight from a smart scale, steps, heart rate,
  active energy, sleep, workouts logged in other apps.
- **Writing summaries other apps consume** — so your workouts close the Apple Watch rings
  and your calories show up in the Health app's nutrition section.

Getting this backwards — treating HealthKit as your persistence layer — is the most common
architectural mistake in this category. You'd immediately lose set-level structure, you'd
have no query performance for charts, and you'd have nothing on Android to match it.

---

## 1. HealthKit (iOS)

### Setup

- Enable the **HealthKit capability** on the App ID (requires a paid developer account).
- Info.plist:
  - `NSHealthShareUsageDescription` — why you read. A real sentence.
  - `NSHealthUpdateUsageDescription` — why you write.
  - Vague strings are a documented App Review rejection reason.
- HealthKit is unavailable on iPad for some types and absent on Mac Catalyst — always gate
  on `HKHealthStore.isHealthDataAvailable()`.

### Authorization — and the one thing that surprises everyone

```swift
try await store.requestAuthorization(toShare: writeTypes, read: readTypes)
```

**You cannot find out whether you have read permission.** `authorizationStatus(for:)` is
only meaningful for *write* types. For reads, Apple deliberately returns
`.notDetermined` or `.sharingDenied` in a way that doesn't distinguish "user said no" from
"user said yes but there's no data" — because revealing the difference would itself leak
health information.

**Design consequences, and they're real:**

- An empty result set is ambiguous. Never render "You have no weight data" as a fact.
  Render "No weight data found — check Health permissions" with a deep link to Settings.
- Ask for permission at the moment of use, with in-app context first. A cold system sheet
  listing 30 data types gets denied.
- Re-requesting is cheap and silent if already granted, so you can safely re-ask when a
  feature is first opened.
- Permissions are **per data type**, and the user can grant read but deny write, or grant
  9 of 10 types. Every read path must degrade gracefully.

### The types you'll use

**Nutrition** (`HKQuantityType`, all `.dietary*`):
```
dietaryEnergyConsumed (kcal)   dietaryProtein (g)        dietaryCarbohydrates (g)
dietaryFatTotal (g)            dietaryFatSaturated (g)   dietaryFiber (g)
dietarySugar (g)               dietarySodium (mg)        dietaryCholesterol (mg)
dietaryWater (mL)              + ~20 vitamins/minerals
```

To make a food entry appear as a **meal** in the Health app rather than as loose nutrient
samples, wrap them in an `HKCorrelation` of type `.food`, with `HKMetadataKeyFoodType` set
to the food's name. This is not optional if you want the Health app to look sane.

**Body:** `bodyMass`, `bodyFatPercentage`, `leanBodyMass`, `height`, `waistCircumference`
**Activity:** `activeEnergyBurned`, `basalEnergyBurned`, `stepCount`, `appleExerciseTime`
**Vitals:** `heartRate`, `restingHeartRate`, `heartRateVariabilitySDNN`
**Characteristics** (read-only, never change): `dateOfBirth`, `biologicalSex`
**Sleep:** `HKCategoryType.sleepAnalysis`

**Workouts:** `HKWorkout`, built via `HKWorkoutBuilder`. Relevant activity types:
`.traditionalStrengthTraining`, `.functionalStrengthTraining`, `.coreTraining`,
`.highIntensityIntervalTraining`, `.running`, `.cycling`.

> **HealthKit has no concept of sets and reps.** A strength workout is one object with a
> start, an end, an activity type, and an energy total. iOS 17+ `HKWorkoutActivity` lets you
> subdivide a workout into intervals, which is the closest you'll get, and it still has no
> rep or load field. Health Connect is genuinely better here (§2). Your own database carries
> the detail; HealthKit gets the summary.

### Reading: pick the right query type

| Need | Query | Notes |
|---|---|---|
| "Everything new since last sync" | **`HKAnchoredObjectQuery`** | The one you want for sync. Returns added *and deleted* objects plus a new anchor. Persist the anchor. |
| "Daily active energy for 90 days" | **`HKStatisticsCollectionQuery`** | Aggregates in the framework. Never fetch raw samples and sum them yourself — it's orders of magnitude slower. |
| "One value right now" | `HKStatisticsQuery` / `HKSampleQuery` | Fine for one-offs. |
| "Tell me when something changes" | **`HKObserverQuery`** + `enableBackgroundDelivery` | Wakes your app in the background. Must call the completion handler or iOS throttles you. |

The canonical sync loop:

```
persisted anchor  ──▶ HKAnchoredObjectQuery(type, anchor)
                        ├── addedSamples   → upsert into local DB
                        ├── deletedObjects → soft-delete locally  ← don't skip this
                        └── newAnchor      → persist
```

Handling `deletedObjects` is the step everyone forgets, and it's why apps show a weight
entry the user deleted from Health three months ago.

### Writing: idempotency and provenance

Two metadata keys make writes safe to repeat:

```
HKMetadataKeySyncIdentifier   — your stable ID for this logical record
HKMetadataKeySyncVersion      — monotonically increasing integer
```

Saving a sample with an existing sync identifier and a **higher** version replaces the
previous one. This turns "write my food log to Health" from a duplicate-generating hazard
into a safe, retryable upsert. Use your local row's UUID as the sync identifier and bump the
version on every edit.

**The read-back loop — the classic HealthKit bug.** You write `dietaryEnergyConsumed`. Later
you read `dietaryEnergyConsumed` to show total intake. You now count your own data twice, or
worse, feed it back into the TDEE estimator. Always filter by source:

```swift
let notMine = HKQuery.predicateForObjects(from: HKSource.default())   // = this app
let predicate = NSCompoundPredicate(notPredicateWithSubpredicate: notMine)
```

Use `HKSourceRevision` and `HKDevice` on every sample you read to know where it came from —
and record that provenance in your own database. Which scale wrote this weight matters when
two scales disagree.

**Testing:** the Simulator has HealthKit but almost no data, and background delivery behaves
differently there. Test on a device with real data. Seed the Simulator via the Health app's
sample data or by writing from a scratch build.

---

## 2. Health Connect (Android)

### Status and availability

Health Connect replaced the Google Fit APIs, which are deprecated and being shut down
(⚠️ Verify current dates — this has moved more than once). Any new Android health
integration targets Health Connect.

- **Android 14+ (API 34):** part of the platform, always present.
- **Android 9–13:** a Play Store app the user must install.

Always branch on availability:

```kotlin
when (HealthConnectClient.getSdkStatus(context)) {
    SDK_UNAVAILABLE -> // hide health features entirely
    SDK_UNAVAILABLE_PROVIDER_UPDATE_REQUIRED -> // prompt Play Store install/update
    SDK_AVAILABLE -> HealthConnectClient.getOrCreate(context)
}
```

Dependency: `androidx.health.connect:connect-client`.

### Permissions

Strongly typed, resolving to manifest-declared strings:

```kotlin
val perms = setOf(
    HealthPermission.getReadPermission(WeightRecord::class),      // android.permission.health.READ_WEIGHT
    HealthPermission.getWritePermission(NutritionRecord::class),
    HealthPermission.getWritePermission(ExerciseSessionRecord::class),
)
// request via PermissionController.createRequestPermissionResultContract()
```

Two extra permissions matter a lot and are easy to miss:

- **`PERMISSION_READ_HEALTH_DATA_HISTORY`** — without it you can only read the **last 30
  days**. If you want to backfill years of weight history on first install, you need this.
- **`PERMISSION_READ_HEALTH_DATA_IN_BACKGROUND`** — required to read while your app isn't in
  the foreground. Without it, background sync silently returns nothing.

Unlike HealthKit, granted permissions **are** queryable (`getGrantedPermissions()`), which
makes the UX considerably more honest. But permissions are revocable at any time, and if the
user denies twice you can no longer prompt — you must deep-link to Health Connect settings.

You must also provide a **permissions rationale activity** handling the
`ACTION_SHOW_PERMISSIONS_RATIONALE` intent (plus the `ViewPermissionUsageActivity` pattern on
Android 14+), linking to your privacy policy. Missing this is a Play rejection.

### Record types

```kotlin
ExerciseSessionRecord(startTime, endTime, exerciseType, title, notes, segments, laps)
ExerciseSegment(startTime, endTime, segmentType, repetitions)    // ← reps!
NutritionRecord(startTime, endTime, energy, protein, totalFat, totalCarbohydrate,
                dietaryFiber, sugar, sodium, ..., mealType, name)
WeightRecord · BodyFatRecord · LeanBodyMassRecord · HeightRecord
TotalCaloriesBurnedRecord · ActiveCaloriesBurnedRecord · StepsRecord
HeartRateRecord · RestingHeartRateRecord · SleepSessionRecord
```

**Health Connect models strength training substantially better than HealthKit.**
`ExerciseSegment` has typed segment kinds — `EXERCISE_SEGMENT_TYPE_BARBELL_SHOULDER_PRESS`,
`_BENCH_PRESS`, `_SQUAT`, `_DEADLIFT`, `_BICEP_CURL`, and so on — **and a `repetitions`
field**. You can write a session whose segments describe the actual exercises and rep counts.

Still no *load* field, so it isn't a substitute for your own schema. But it means your
Android export is richer than your iOS one, and your abstraction layer has to tolerate that
asymmetry rather than flattening to the lowest common denominator (§5).

`NutritionRecord` is also nicer than HealthKit's approach: one record holds every nutrient
plus `mealType` and a name, so no correlation-wrapping ceremony.

### Reading and the Changes API

The direct analogue of `HKAnchoredObjectQuery`:

```kotlin
val token = client.getChangesToken(ChangesTokenRequest(setOf(WeightRecord::class)))
// later, and repeatedly:
val response = client.getChanges(token)
response.changes.forEach {
    when (it) {
        is UpsertionChange -> upsertLocal(it.record)
        is DeletionChange  -> softDeleteLocal(it.recordId)
    }
}
persist(response.nextChangesToken)
```

**Change tokens expire after ~30 days of non-use.** `response.changesTokenExpired` tells you;
when it's true you must fall back to a full read and start a fresh token. Handle this — a
phone left in a drawer for a month will hit it.

For aggregates use `aggregate()` / `aggregateGroupByPeriod()` rather than summing records
yourself, same reasoning as HealthKit.

### Writing: idempotency and provenance

```kotlin
Metadata(clientRecordId = localRow.uuid, clientRecordVersion = localRow.version)
```

Same semantics as HealthKit's sync identifier: insert with an existing `clientRecordId` and
a higher `clientRecordVersion` replaces the earlier record. Pleasingly symmetric — one
concept, two names, so your abstraction layer has an easy job here.

Provenance is `DataOrigin`, which is the writing app's **package name**. Filter your own
package out when reading back, exactly as on iOS.

---

## 3. The mapping table

The most useful artifact in this document. This is the contract your `HealthGateway`
implements.

| Domain concept | HealthKit | Health Connect | Direction |
|---|---|---|---|
| Bodyweight | `bodyMass` | `WeightRecord` | read + write |
| Body fat % | `bodyFatPercentage` | `BodyFatRecord` | read + write |
| Lean mass | `leanBodyMass` | `LeanBodyMassRecord` | read + write |
| Height | `height` | `HeightRecord` | read |
| Calories eaten | `dietaryEnergyConsumed` | `NutritionRecord.energy` | write |
| Protein | `dietaryProtein` | `NutritionRecord.protein` | write |
| Carbs | `dietaryCarbohydrates` | `NutritionRecord.totalCarbohydrate` | write |
| Fat | `dietaryFatTotal` | `NutritionRecord.totalFat` | write |
| Fibre | `dietaryFiber` | `NutritionRecord.dietaryFiber` | write |
| Sodium | `dietarySodium` | `NutritionRecord.sodium` | write |
| Meal grouping | `HKCorrelation(.food)` + `HKMetadataKeyFoodType` | `NutritionRecord.mealType` + `.name` | write |
| Water | `dietaryWater` | `HydrationRecord` | read + write |
| Strength session | `HKWorkout(.traditionalStrengthTraining)` | `ExerciseSessionRecord` | write |
| Per-exercise detail | **not representable** | `ExerciseSegment` (+ `repetitions`) | write (Android only) |
| Session energy | `activeEnergyBurned` on the workout | `ActiveCaloriesBurnedRecord` | write |
| Steps | `stepCount` | `StepsRecord` | read |
| Active energy (all sources) | `activeEnergyBurned` | `ActiveCaloriesBurnedRecord` | read |
| Resting energy | `basalEnergyBurned` | `BasalMetabolicRateRecord` | read |
| Heart rate | `heartRate` | `HeartRateRecord` | read |
| Sleep | `sleepAnalysis` | `SleepSessionRecord` | read |
| **Incremental sync cursor** | `HKQueryAnchor` | Changes token | — |
| **Idempotent write key** | `HKMetadataKeySyncIdentifier` + `…SyncVersion` | `clientRecordId` + `clientRecordVersion` | — |
| **Provenance** | `HKSource` / `HKSourceRevision` | `DataOrigin` (package name) | — |
| **Read-permission introspection** | ✗ not possible | ✓ `getGrantedPermissions()` | — |
| **History beyond 30 days** | unrestricted | needs `READ_HEALTH_DATA_HISTORY` | — |

---

## 4. What to read vs. what to write — for this app

**Read (in priority order):**

1. **Bodyweight.** The single most valuable read. A smart scale (Withings, Renpho, Eufy,
   Garmin) writes into the health store, and your TDEE algorithm consumes it without the
   user ever opening your app. This is what makes daily weigh-ins actually happen, and daily
   weigh-ins are what make the algorithm work ([doc 02](02-calorie-trackers-breakdown.md)).
2. **Steps and active energy.** Not for the calorie math — the adaptive model already
   absorbs activity. Use them as *context*: "your expenditure estimate dropped 180 kcal;
   your step count also dropped 3,000/day." That's a genuinely useful insight, and it's only
   possible because you read it.
3. **Workouts from other apps.** Runs, rides, classes — so your training calendar is complete.
4. **Body fat %, sleep, resting HR.** Nice-to-have context for recovery and phase decisions.

**Write:**

1. **Nutrition totals per meal.** So Apple Health / Health Connect show a complete picture
   and other apps can use it.
2. **Strength workouts.** So Apple Watch rings close and the Fitness app is accurate.
   On Android, include `ExerciseSegment`s with reps — it costs nothing and the data is better.
3. **Bodyweight**, when entered manually in your app.

**Deliberately don't write:** anything you read. Never echo another app's data back into
the store. That's how the health ecosystem develops feedback loops and duplicate entries.

**The ecosystem bonus, worth stating plainly:** Whoop, Oura, Garmin, Withings, Peloton, and
essentially every fitness device write into HealthKit and Health Connect. Integrating with
*two* platform APIs gets you *all* of them, for free, with no per-vendor OAuth, no rate
limits, and no partner agreements. Never integrate a wearable vendor's API directly unless
you need something the health store genuinely can't express.

---

## 5. The abstraction to build

One interface, two implementations, and nothing else in the app knows which platform it's on.

```
interface HealthGateway {
    isAvailable(): Availability                    // unavailable | needs_provider_update | available
    requestPermissions(scopes: Set<Scope>): PermissionResult
    grantedPermissions(): Set<Scope> | Unknown     // Unknown on iOS — see §1

    // incremental read; cursor is opaque, persisted by the caller
    pull(scope: Scope, cursor: Cursor?): PullResult
       // → { upserts: HealthRecord[], deletions: ExternalId[], nextCursor, cursorWasReset }

    // idempotent write; safe to call repeatedly with the same (id, version)
    push(records: HealthRecord[]): PushResult

    // aggregates delegated to the platform, never computed client-side
    aggregate(scope, bucket: Day|Week, range): Bucket[]
}
```

**Design rules that make this hold up:**

1. **`Cursor` is opaque.** An `HKQueryAnchor` on one side, a changes token on the other.
   Callers persist it and pass it back; they never inspect it.
2. **`cursorWasReset` is part of the contract.** Both platforms can invalidate a cursor
   (Health Connect after 30 days, HealthKit on restore-from-backup). The caller must have a
   documented full-resync path, not a crash.
3. **`grantedPermissions()` may return `Unknown`.** Don't paper over HealthKit's opacity by
   guessing — model it in the type so every call site is forced to handle it.
4. **Don't flatten to the lowest common denominator.** `HealthRecord` for a workout carries
   optional per-exercise segments. iOS drops them; Android writes them. Losing Android's
   richer model to make the interface symmetric is the wrong trade.
5. **Provenance is a first-class field** on every record read, carried into your database.
6. **The gateway does no business logic.** It translates. Deduplication, source filtering,
   and conflict resolution live in a layer above it, where they're testable without a device.

**Testing:** the gateway is the hardest thing in the app to test, because it needs a real
device with real data. Mitigate by keeping it *thin* — translation only — so the logic worth
testing lives above it in pure functions with fixtures. A fake `HealthGateway` returning
canned records covers the layer above; the real implementations get manual device testing
against the release checklist in [doc 04](04-productionization-and-deployment.md) §6.

---

## 6. Sync architecture end to end

Three independent sync loops. Don't entangle them — different failure modes, different
cadences, different consequences.

```
┌──────────────┐   pull(cursor)    ┌──────────────┐
│  HealthKit / │ ────────────────▶ │              │
│ Health Conn. │ ◀──────────────── │  Local       │  ← source of truth
└──────────────┘   push(records)   │  SQLite      │
                                   │              │
┌──────────────┐   pull(since)     │              │
│  Your        │ ────────────────▶ │              │
│  backend     │ ◀──────────────── │              │
└──────────────┘   push(outbox)    └──────────────┘
        ▲                                  │
        └──── other device ────────────────┘
```

- **Health ⇄ local** — this document. Cursor-based, per data type.
- **Local ⇄ backend** — [doc 06](06-data-architecture.md). Outbox + pull-since-cursor.
- **Local is always the source of truth** for app-owned entities (sets, foods, routines).

**Loop-prevention rules, which are non-negotiable:**

- Records ingested *from* the health store are flagged `origin = 'health'` and are never
  pushed back to it.
- Records read from the health store that your own app wrote are discarded at ingest by
  source/package filtering.
- A bodyweight entry has exactly one owner. If it came from a scale via HealthKit, your app
  displays it but doesn't re-write it.

**Cadence:** health pull on app foreground and on `HKObserverQuery` / background-read wake;
health push on transaction commit, queued and retried through the same outbox as backend
sync so a failed write isn't silently lost.

---

## Takeaways

1. The health store is an **integration surface, not a database**. Your SQLite schema owns
   sets, reps, foods, and portions. HealthKit gets summaries.
2. **You cannot query read permission on iOS.** Model that ambiguity in your types rather
   than guessing, or you'll show "no data" when the truth is "no permission".
3. Use **anchored queries / the Changes API** for reads. Handle deletions. Handle cursor
   expiry. Persist the cursor.
4. Use **sync identifiers / `clientRecordId` + version** for writes. This is what makes them
   idempotent and retryable.
5. **Always filter out your own writes when reading**, or you'll double-count and poison the
   TDEE estimator.
6. **Health Connect models strength training better than HealthKit** (segments with reps).
   Let the abstraction expose that rather than flattening it away.
7. Reading **bodyweight from a smart scale** is the highest-leverage integration in the
   whole app — it's what makes the adaptive algorithm work without daily manual effort.
8. Integrating two platform APIs gets you every wearable for free. Don't integrate vendors
   directly.
