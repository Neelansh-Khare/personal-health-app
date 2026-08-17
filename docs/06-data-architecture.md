# 06 — Data Architecture: Local Database and Sync

**Decision: local-first.** SQLite on the device is the source of truth. A backend exists so
that years of logs survive a dead phone and so an iPhone and an Android phone can both be
used. The app must be fully functional with the network off — gyms have concrete walls and
basements.

---

## 1. Principles

1. **The device is authoritative.** Every write goes to local SQLite first, synchronously,
   inside a transaction. Sync is a background reconciliation, never a blocking dependency.
2. **Never block the UI on the network.** No spinner has any business appearing when you
   tap ✓ on a set.
3. **Client-generated IDs.** Rows are created offline and must have permanent identity
   immediately. No server round-trip for a primary key, ever.
4. **Soft deletes only.** Tombstones, so deletions propagate. Hard deletes are a server-side
   garbage-collection concern, run long after the fact.
5. **Store canonical units. Always.** kg, metres, seconds, grams, kcal, millilitres. Display
   units are a presentation-layer concern. This is a one-line decision now and a
   multi-week migration later.
6. **Derived data is derived.** e1RM, volume, trend weight, PR flags, and expenditure
   estimates are all computed from raw rows. Cache them for speed, but never treat the cache
   as truth — you must be able to drop and rebuild every one of them.

---

## 2. The four traps

These are the ones that are cheap to handle now and genuinely expensive to retrofit.

### Trap 1 — the diary day

You eat a snack at 00:45. Which day's calorie total does it count toward? Yesterday's, by
every reasonable user's expectation. But `date(logged_at)` says today.

```sql
logged_at  INTEGER NOT NULL,   -- unix ms, UTC. The instant it happened.
log_date   TEXT    NOT NULL,   -- 'YYYY-MM-DD', local calendar date. The diary bucket.
```

Store both. Compute `log_date` **once, at write time**, from the device's local timezone and
a user-configurable day-boundary hour (default 04:00, so late nights land correctly). Never
re-derive it at query time — the user may have flown to another timezone since, and the
snack should not migrate to a different day because of it.

Every daily aggregate groups by `log_date`. Every chart's x-axis is `log_date`. The timestamp
exists for ordering within a day and for writing to HealthKit.

Same applies to workouts and weigh-ins.

### Trap 2 — mutable food data

You log "Tesco Greek Yoghurt, 200 g, 118 kcal". Six months later an Open Food Facts
contributor corrects that product's protein value. **Your historical calorie totals must not
change.** They're a record of what you believed and acted on, and the whole TDEE algorithm is
built on the assumption that history is stable.

So a log entry **snapshots** its nutrients:

```sql
food_id       TEXT,        -- reference, for "log this again" and for editing
grams         REAL,
kcal          REAL,        -- snapshot, computed at log time
protein_g     REAL,        -- snapshot
fat_g         REAL,        -- snapshot
carb_g        REAL,        -- snapshot
food_name     TEXT,        -- snapshot, so deleted foods still render
```

Denormalized on purpose. It also means a food can be deleted without orphaning history, and
daily totals are a single-table sum with no joins.

### Trap 3 — units

`weight_kg REAL`. Not `weight REAL` plus a `unit` column. Convert at the edges: parse on
input, format on output, store canonical in between. A user switching from lb to kg is a
settings change, not a data migration.

The corollary is display-precision: store full precision, round only for display, and pick
increments per unit (2.5 kg / 5 lb) as a UI concern.

### Trap 4 — in-progress state

An unfinished workout is device-local and must **not** sync. Two devices with a half-finished
session each is a merge problem with no good answer, and solving it correctly buys nothing —
you're one person, in one gym, holding one phone.

The rule is one line: **only sync `workout` rows where `ended_at IS NOT NULL`.**

---

## 3. Schema

SQLite. WAL mode, foreign keys on. IDs are **UUIDv7** — 128-bit, client-generatable offline,
and time-sortable, so they index well as primary keys and give you free chronological
ordering.

Every syncable table carries the same four columns; they're written out here once and elided
with `-- +sync` below.

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

-- The sync columns present on every syncable table:
--   id          TEXT PRIMARY KEY,        -- UUIDv7
--   updated_at  INTEGER NOT NULL,        -- unix ms UTC, client clock
--   deleted_at  INTEGER,                 -- tombstone
--   server_seq  INTEGER                  -- assigned by server; NULL = not yet synced
```

### Training

```sql
CREATE TABLE exercise (
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  name                 TEXT NOT NULL,
  measurement_type     TEXT NOT NULL,   -- weight_reps | bodyweight_reps | assisted_bodyweight
                                        -- | weighted_bodyweight | duration | duration_weight
                                        -- | distance_duration | reps_only
  primary_muscle       TEXT,            -- chest | back | quads | hamstrings | glutes | delts
                                        -- | biceps | triceps | calves | core | forearms
  secondary_muscles    TEXT,            -- JSON array
  equipment            TEXT,            -- barbell | dumbbell | machine | cable | bodyweight | band
  is_custom            INTEGER NOT NULL DEFAULT 0,
  default_rest_seconds INTEGER,
  notes                TEXT
);

CREATE TABLE routine (                                              -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  name TEXT NOT NULL, folder TEXT, sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE routine_exercise (                                     -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  routine_id     TEXT NOT NULL REFERENCES routine(id),
  exercise_id    TEXT NOT NULL REFERENCES exercise(id),
  sort_order     INTEGER NOT NULL,
  superset_group INTEGER,               -- NULL = not in a superset
  rest_seconds   INTEGER,
  note           TEXT
);

CREATE TABLE routine_set (              -- the plan                 -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  routine_exercise_id TEXT NOT NULL REFERENCES routine_exercise(id),
  sort_order    INTEGER NOT NULL,
  set_type      TEXT NOT NULL DEFAULT 'normal',   -- normal | warmup | drop | failure
  target_reps   INTEGER,
  target_weight_kg REAL,
  target_pct_1rm   REAL                -- for percentage-based programming (5/3/1 etc.)
);

CREATE TABLE workout (                  -- the session              -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  name          TEXT NOT NULL,
  routine_id    TEXT,                   -- weak ref: deleting a routine must not touch history
  started_at    INTEGER NOT NULL,
  ended_at      INTEGER,                -- NULL = in progress, and therefore NOT synced
  local_date    TEXT NOT NULL,          -- 'YYYY-MM-DD'
  notes         TEXT,
  bodyweight_kg REAL                    -- snapshot; makes historical bodyweight volume correct
);

CREATE TABLE workout_exercise (                                     -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  workout_id     TEXT NOT NULL REFERENCES workout(id),
  exercise_id    TEXT NOT NULL REFERENCES exercise(id),
  sort_order     INTEGER NOT NULL,
  superset_group INTEGER,
  notes          TEXT
);

CREATE TABLE workout_set (              -- the fact                 -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  workout_exercise_id TEXT NOT NULL REFERENCES workout_exercise(id),
  sort_order   INTEGER NOT NULL,
  set_type     TEXT NOT NULL DEFAULT 'normal',
  weight_kg    REAL,
  reps         INTEGER,
  distance_m   REAL,
  duration_s   INTEGER,
  rpe          REAL,                    -- 6.0 – 10.0, half-point increments
  completed_at INTEGER
);
```

### Body and nutrition

```sql
CREATE TABLE body_metric (                                          -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  kind        TEXT NOT NULL,        -- weight | bodyfat_pct | waist | chest | arm_l | ...
  value       REAL NOT NULL,        -- canonical unit for the kind (kg, %, cm)
  measured_at INTEGER NOT NULL,
  local_date  TEXT NOT NULL,
  source      TEXT NOT NULL DEFAULT 'manual',  -- manual | healthkit | health_connect
  source_id   TEXT,                 -- external record id, for dedup on re-sync
  UNIQUE(kind, local_date, source, source_id)  -- one weigh-in per day per source
);

CREATE TABLE food (                   -- see doc 07                 -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  source        TEXT NOT NULL,      -- usda | off | custom | quick_add
  source_id     TEXT,
  barcode       TEXT,
  name          TEXT NOT NULL,
  brand         TEXT,
  -- all nutrients normalized per 100 g
  kcal_100g     REAL NOT NULL,
  protein_100g  REAL NOT NULL,
  fat_100g      REAL NOT NULL,
  carb_100g     REAL NOT NULL,
  fibre_100g    REAL, sugar_100g REAL, sat_fat_100g REAL,
  sodium_mg_100g REAL, potassium_mg_100g REAL, cholesterol_mg_100g REAL,
  micros_json   TEXT,               -- everything else, sparse
  density_g_ml  REAL,               -- for volume→mass conversion
  quality_score REAL,               -- see doc 07 §6
  is_favourite  INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE food_portion (           -- '1 medium (118 g)'         -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  food_id TEXT NOT NULL REFERENCES food(id),
  label   TEXT NOT NULL,
  grams   REAL NOT NULL,
  is_default INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE recipe (                                               -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  name TEXT NOT NULL, servings REAL NOT NULL DEFAULT 1, notes TEXT
);

CREATE TABLE recipe_ingredient (                                    -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  recipe_id TEXT NOT NULL REFERENCES recipe(id),
  food_id   TEXT NOT NULL REFERENCES food(id),
  grams     REAL NOT NULL,
  sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE food_log_entry (                                       -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  log_date   TEXT NOT NULL,          -- diary bucket. See Trap 1.
  logged_at  INTEGER NOT NULL,       -- the instant
  meal       TEXT NOT NULL,          -- breakfast | lunch | dinner | snack
  sort_order INTEGER NOT NULL DEFAULT 0,

  food_id    TEXT,                   -- nullable: quick-add has no food
  recipe_id  TEXT,
  grams      REAL,
  portion_label TEXT,                -- what the user actually picked: '2 slices'

  -- IMMUTABLE SNAPSHOT. See Trap 2. Never recompute these.
  food_name  TEXT NOT NULL,
  kcal       REAL NOT NULL,
  protein_g  REAL NOT NULL,
  fat_g      REAL NOT NULL,
  carb_g     REAL NOT NULL,
  fibre_g    REAL, sugar_g REAL, sodium_mg REAL,
  micros_json TEXT
);
```

### Algorithm state

Derived, but persisted so history is auditable and charts are fast. All of it must be
rebuildable from scratch — that's the test suite's job
([doc 02](02-calorie-trackers-breakdown.md)).

```sql
CREATE TABLE daily_rollup (           -- local cache, NOT synced, rebuildable
  local_date    TEXT PRIMARY KEY,
  kcal_in       REAL, protein_g REAL, fat_g REAL, carb_g REAL, fibre_g REAL,
  entry_count   INTEGER,
  weight_kg     REAL,                 -- raw scale, if any
  trend_kg      REAL,                 -- Kalman posterior for weight
  expenditure   REAL,                 -- Kalman posterior for E
  expenditure_sd REAL,                -- sqrt(P[1][1]) — the uncertainty band
  logging_complete INTEGER,           -- 0/1 heuristic: was intake plausibly fully logged?
  training_volume_kg REAL,
  steps         INTEGER
);

CREATE TABLE nutrition_target (       -- the weekly program         -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  effective_from TEXT NOT NULL,       -- 'YYYY-MM-DD'
  kcal REAL NOT NULL, protein_g REAL NOT NULL, fat_g REAL NOT NULL, carb_g REAL NOT NULL,
  fibre_g REAL,
  goal_rate_pct_per_week REAL NOT NULL,   -- negative = cut
  expenditure_at_issue   REAL NOT NULL,
  mode TEXT NOT NULL,                 -- coached | collaborative | manual
  accepted_at INTEGER
);

CREATE TABLE diet_phase (             -- cut / bulk / maintain      -- +sync
  id TEXT PRIMARY KEY, updated_at INTEGER NOT NULL, deleted_at INTEGER, server_seq INTEGER,
  kind TEXT NOT NULL,                 -- cut | bulk | maintain
  started_on TEXT NOT NULL, ended_on TEXT, notes TEXT
);
```

`diet_phase` earns its place twice over: the Kalman filter inflates process noise for ~14
days after a phase boundary (the glycogen trap,
[doc 02](02-calorie-trackers-breakdown.md) §2), and every strength chart in the app gets
shaded phase bands behind it — which is the core insight of the combined app
([doc 08](08-combined-app-design.md)).

### Sync bookkeeping (local only, never synced)

```sql
CREATE TABLE sync_outbox (
  seq         INTEGER PRIMARY KEY AUTOINCREMENT,
  table_name  TEXT NOT NULL,
  row_id      TEXT NOT NULL,
  queued_at   INTEGER NOT NULL,
  attempts    INTEGER NOT NULL DEFAULT 0,
  last_error  TEXT,
  UNIQUE(table_name, row_id)          -- coalesce repeated edits to one pending push
);

CREATE TABLE sync_state (
  key TEXT PRIMARY KEY,               -- 'backend_cursor', 'device_id',
  value TEXT                          -- 'healthkit:bodyMass', 'hc:WeightRecord', ...
);
```

### Indexes

Every chart in the app is one of these queries. Without these indexes the app feels slow
after about a year of data; with them it stays instant essentially forever.

```sql
CREATE INDEX idx_set_exercise_time ON workout_set(workout_exercise_id, sort_order);
CREATE INDEX idx_we_workout        ON workout_exercise(workout_id, sort_order);
CREATE INDEX idx_we_exercise       ON workout_exercise(exercise_id);   -- per-exercise history
CREATE INDEX idx_workout_date      ON workout(local_date DESC);
CREATE INDEX idx_log_date_meal     ON food_log_entry(log_date, meal, sort_order);
CREATE INDEX idx_log_food          ON food_log_entry(food_id);          -- 'recents' / frequency
CREATE INDEX idx_body_kind_date    ON body_metric(kind, local_date DESC);
CREATE INDEX idx_food_barcode      ON food(barcode) WHERE barcode IS NOT NULL;
-- one per syncable table:
CREATE INDEX idx_workout_set_seq   ON workout_set(server_seq);
```

Plus an FTS5 virtual table for food search — see [doc 07](07-food-database.md) §5.

**Scale check.** A serious user generates ~5,000 sets and ~2,000 food entries per year.
After a decade that's ~70,000 rows total. SQLite does not notice this. **Do not
pre-optimize.** Correctness, migration safety, and query clarity matter far more than
performance here, and reaching for a fancier storage engine is wasted effort.

---

## 4. The sync engine

The requirements are unusually forgiving and you should exploit that:

- **One user.** No multi-tenancy, no permissions model, no collaborative editing.
- **Rarely two devices at once.** You don't lift on your iPhone and Android simultaneously.
- **Conflicts are genuinely rare**, and when they happen, last-write-wins is an acceptable
  outcome for every entity in this schema.

So: **an outbox plus last-write-wins plus a server-assigned sequence cursor.** No CRDTs, no
operational transforms, no vector clocks. Roughly 300 lines, fully testable as pure
functions, and you will understand it in two years when it breaks.

### Push

```
1. Read a batch from sync_outbox (oldest first, ~200 rows).
2. For each, load the current row. Skip in-progress workouts (ended_at IS NULL).
3. POST the rows to the backend.
4. Server upserts, assigning each row a monotonic server_seq.
5. Store the returned server_seq on each local row; delete the outbox entries.
6. On failure: increment attempts, exponential backoff, leave the entry queued.
```

The `UNIQUE(table_name, row_id)` constraint means editing the same set ten times before the
network returns produces one push, not ten.

### Pull

```
1. cursor = sync_state['backend_cursor']            (last seen server_seq)
2. GET /changes?since={cursor}&limit=500
3. For each incoming row, resolve against local:
      local missing                  → insert
      local.updated_at <  remote     → overwrite
      local.updated_at >  remote     → keep local, re-queue to outbox
      equal                          → tiebreak on device_id, deterministically
4. Apply tombstones (deleted_at) as local soft-deletes.
5. cursor = max(server_seq); repeat while a full page returns.
6. Invalidate daily_rollup for affected dates; recompute lazily.
```

### Server side (Postgres, ~15 lines)

The whole server-side sync mechanism is a sequence and a trigger:

```sql
CREATE SEQUENCE sync_seq;

CREATE OR REPLACE FUNCTION assign_server_seq() RETURNS trigger AS $$
BEGIN
  NEW.server_seq := nextval('sync_seq');
  RETURN NEW;
END $$ LANGUAGE plpgsql;

-- for each synced table:
CREATE TRIGGER t_workout_set_seq BEFORE INSERT OR UPDATE ON workout_set
  FOR EACH ROW EXECUTE FUNCTION assign_server_seq();

CREATE INDEX ON workout_set(server_seq);
```

Pull is then literally `SELECT * FROM t WHERE server_seq > $1 ORDER BY server_seq LIMIT 500`.
A single global sequence across all tables gives you a total order and therefore a single
cursor, which keeps the client trivial.

### The details that will bite you

- **Clock skew.** `updated_at` comes from the client and clients lie — timezone changes,
  manual clock adjustments, NTP jumps. Ordering for *transport* uses the server sequence
  (authoritative); ordering for *conflict resolution* uses `updated_at` (best-effort).
  Clamp any client timestamp more than a few minutes in the future to server time on ingest.
- **Tombstone retention.** Keep tombstones ≥ 90 days server-side. A device offline longer
  than that must do a full resync, so store a `min_retained_seq` and have the server reply
  `cursor_too_old` when the client asks for something older.
- **Partial failures.** Push in batches with an idempotency key per batch. Retrying a
  half-applied batch must be safe — upserts by primary key make it so.
- **Never trust `deleted_at` to mean gone.** Every read query filters `deleted_at IS NULL`.
  Put it in a view per table so it can't be forgotten.
- **First sync on a new device** downloads everything from `server_seq = 0`. For a decade of
  data that's a few MB — fine, but paginate and show progress.
- **Account deletion** (an App Store requirement if you ever go public — see
  [doc 04](04-productionization-and-deployment.md)) must hard-delete server rows and wipe
  local storage. Design the backend so this is one cascade, not a scavenger hunt.

---

## 5. Backend options

| Option | Fit | Notes |
|---|---|---|
| **Supabase** ⭐ | **Recommended** | Postgres + auth + Row Level Security + storage. Free tier covers one user with enormous headroom. The sequence-and-trigger scheme above is native. Self-hostable if you ever want out. |
| Firebase / Firestore | Good | Excellent offline SDK does much of this for you. But: document model fights a relational schema, queries are limited, and pricing is per-read, which punishes chart-heavy apps. |
| Cloudflare D1 + Workers | Good, more DIY | SQLite on both ends is pleasingly symmetric. Cheap. You write the auth. |
| Turso (embedded replicas) | Interesting | SQLite-native with built-in replication; the sync story is partly solved for you. Newer, so less settled. |
| PowerSync / ElectricSQL | Overkill | Purpose-built Postgres⇄SQLite sync. Genuinely good, and genuinely more machinery than one user needs. Revisit only if hand-rolled sync becomes a maintenance burden. |
| Plain Postgres on a VPS | Fine | Most control, most ops. You'll be patching a server at 1 a.m. for an app only you use. |
| MongoDB Atlas Device Sync | ✗ | Deprecated. ⚠️ Verify, but do not build on it. |

**Recommendation: Supabase.** Postgres means the schema above transfers almost verbatim,
RLS means "one user, one row set" is a one-line policy, auth is solved, and the free tier is
never going to be exceeded by a single person's training log.

**Also worth saying:** sync is a Phase 7 feature ([doc 09](09-execution-plan.md)). Build the
entire app local-only first. Every table already has its sync columns from day one — that's
the cheap part — but the engine itself waits until the app is worth syncing. Building sync
before you have data to sync is a classic way to spend a month and ship nothing.

---

## 6. Migrations

```
db/migrations/
  0001_initial.sql
  0002_add_rpe_to_workout_set.sql
  0003_food_quality_score.sql
```

Rules:

1. **Raw SQL, numbered, forward-only.** Not ORM model diffs. The schema then survives an ORM
   change or a whole-stack change, and it's readable in five years.
2. `PRAGMA user_version` tracks the applied version. Apply pending migrations in a
   transaction at startup.
3. **No down-migrations.** Roll forward. A "fix" migration is cheaper than a reversible one,
   and reversibility is a lie the moment data is involved.
4. **Copy the database file before migrating.** Keep the copy until the app has launched
   successfully once. This is the cheapest insurance you will ever buy.
5. **Test every migration against a copy of your real database**, not a fixture. Your real
   data contains years of edge cases fixtures don't have.

---

## 7. Export, backup, restore

The most important non-feature in the app ([doc 04](04-productionization-and-deployment.md) §7).

**Export formats:**
- **Full SQLite file copy** — the real backup. Lossless, restorable, one file.
- **JSON** — a documented, versioned dump. Portable, diffable, survives a stack change.
- **CSV per table** — for spreadsheet analysis, which you will want.
- **Strong-compatible workout CSV** — so you're never locked into your own app either.

**Import:**
- **Strong CSV** — build this early ([doc 01](01-strong-app-breakdown.md) §6). Starting with
  years of history makes every chart useful on day one instead of in month six.
- MyFitnessPal / MacroFactor CSV export, if you have history worth keeping.
- Your own JSON, for restore.

**Automated backup:** weekly, unattended, to iCloud Drive or Google Drive, versioned rather
than overwritten. And **test the restore path** — into a clean install, at least once, and
again after any migration. An untested backup is a rumour.

---

## Takeaways

1. Local SQLite is the source of truth. Sync is reconciliation, never a dependency.
2. UUIDv7 primary keys, generated on device. Soft deletes everywhere.
3. Handle the four traps now: **diary day**, **nutrient snapshots**, **canonical units**,
   **don't sync in-progress workouts**.
4. Sync = outbox + last-write-wins + server-assigned sequence cursor. One user makes this
   easy; don't reach for CRDTs.
5. Server-side sync is a sequence and a trigger. Supabase for the rest.
6. Raw SQL migrations, forward-only, tested against a copy of your real database.
7. Add the sync *columns* on day one; build the sync *engine* in Phase 6.
8. Export and restore ship in the MVP, and the restore path gets tested.
