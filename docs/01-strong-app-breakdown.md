# 01 — Strong: A Teardown

**What it is:** Strong (Strong Fitness Systems) is the reference-standard workout logger on
iOS and Android. It has a watchOS and Wear OS companion. Freemium — the free tier caps you
at 3 custom routines; Strong PRO unlocks everything else (~$4.99/mo, ~$29.99/yr ⚠️ Verify).

**Why it's the one to copy:** Strong is not the most *featureful* workout app. It is the
fastest one to use with sweaty hands, one thumb, 30 seconds of rest, and a phone you'd
rather not be looking at. Everything below exists in service of that.

---

## 1. The core mental model

Four nouns. Everything in the app is one of these:

```
Exercise    a movement definition          "Barbell Bench Press"
Routine     a reusable template            "Push Day A"
Workout     one actual session, in time    "Push Day A — Tue 14 Jan, 62 min"
Set         one logged effort              "100 kg × 8, RPE 8"
```

The relationship that makes the app work: **a Workout is instantiated from a Routine, but
immediately becomes independent of it.** You can add exercises, skip exercises, change the
weights — and the Routine is untouched unless you explicitly say "update routine". This is
the single most important structural decision in the app and it's easy to get wrong by
over-normalizing.

### Exercise measurement types

This enum drives the entire logging UI. Get it right early; retrofitting it is painful.

| Type | Fields logged | Examples |
|------|---------------|----------|
| `weight_reps` | weight, reps | Bench press, squat, curls |
| `bodyweight_reps` | reps (+ optional added weight) | Pull-ups, dips, push-ups |
| `assisted_bodyweight` | reps, assistance weight (subtracted) | Assisted pull-up machine |
| `weighted_bodyweight` | reps, added weight | Weighted dips |
| `duration` | seconds | Plank, wall sit |
| `duration_weight` | seconds, weight | Farmer's carry (time) |
| `distance_duration` | metres, seconds | Running, rowing, cycling |
| `reps_only` | reps | Ab wheel, band pull-aparts |

Each type implies a different keypad layout, a different "previous" display, and a
different volume formula. `bodyweight_reps` volume needs the user's bodyweight to be
meaningful — which is the first place the nutrition side of a combined app becomes useful
(doc 08).

---

## 2. The active workout screen — the whole product

If you build one screen extremely well, build this one. The anatomy:

```
┌──────────────────────────────────────────────┐
│  Push Day A            00:42:17      [Finish]│   ← elapsed timer, always visible
├──────────────────────────────────────────────┤
│  Rest: 01:28  ▓▓▓▓▓▓▓▓░░░░░░  [+15s] [Skip] │   ← appears only while resting
├──────────────────────────────────────────────┤
│  Barbell Bench Press                    ⋯    │
│  ┌────┬───────────┬────────┬──────┬────────┐ │
│  │ SET│ PREVIOUS  │  KG    │ REPS │   ✓    │ │
│  ├────┼───────────┼────────┼──────┼────────┤ │
│  │  W │ —         │  60    │  10  │  ✓     │ │   ← warm-up set (W)
│  │  1 │ 100×8     │  100   │   8  │  ✓     │ │   ← done, filled green
│  │  2 │ 100×8     │  100   │   7  │  ✓     │ │
│  │  3 │ 100×7     │ [100]  │ [ ]  │  ○     │ │   ← active row
│  └────┴───────────┴────────┴──────┴────────┘ │
│  + Add Set                                   │
├──────────────────────────────────────────────┤
│  Incline Dumbbell Press                 ⋯    │
│  ...                                         │
└──────────────────────────────────────────────┘
```

**The details that make it fast — in rough order of how much they matter:**

1. **The `PREVIOUS` column.** Shows what you did for *this set index* of *this exercise*
   last time you performed it. This is the highest-value feature in the entire app. It
   removes the need to remember anything, and it turns logging into confirmation rather
   than data entry.

2. **Tap-to-autofill.** Tapping the ✓ on an empty row copies the previous values in and
   marks it complete. Typical set = one tap. This is why people don't mind logging.

3. **Rest timer auto-starts on set completion.** Not a separate button. The timer's
   default duration is per-exercise (big compounds 3 min, curls 60 s) and remembered.
   It must fire a notification when the app is backgrounded or the screen is off.

4. **Persist every keystroke.** The workout must survive the app being killed, the phone
   dying mid-set, or a call coming in. Write each set to disk on change, not on "Finish".
   Losing an in-progress workout is the one unforgivable bug in this category of app.

5. **Big tap targets, numeric keypads with a decimal, `+`/`−` steppers.** Increment size
   is unit-aware (2.5 kg / 5 lb) and ideally per-exercise (dumbbells go in 2 kg jumps,
   the leg press in 10 kg).

6. **Never navigate away.** Adding an exercise, checking history, starting a rest timer —
   all of it happens in sheets over this screen. You never lose your place.

7. **Set types.** Normal, Warm-up (`W`, excluded from volume and PRs), Drop set (`D`),
   Failure (`F`). Rendered as a leading badge you can tap to cycle.

8. **Supersets.** Exercises grouped by a colour bar on the left edge; completing a set in
   one jumps focus to the next exercise in the group.

9. **Plate calculator.** Given target weight + bar weight, show plates per side. Two taps
   from any weight field. Surprisingly beloved.

10. **Keep-awake + background survival.** iOS: Live Activity for the rest timer and
    elapsed time. Android: foreground service with an ongoing notification. Non-optional —
    the OS will kill you otherwise.

---

## 3. Data model (reverse-engineered)

This is close to what Strong must be doing, and is a reasonable starting point for the
combined app. Full DDL lives in [doc 06](06-data-architecture.md).

```
exercise
  id, name, measurement_type, primary_muscle, other_muscles[],
  equipment, is_custom, default_rest_seconds, notes

routine
  id, name, folder_id, sort_order, updated_at

routine_exercise
  id, routine_id, exercise_id, sort_order, superset_group, rest_seconds, note

routine_set                          -- the *plan*
  id, routine_exercise_id, sort_order, set_type, target_reps, target_weight

workout                              -- the *session*
  id, name, routine_id (nullable, historical reference only),
  started_at, ended_at, notes, bodyweight_kg

workout_exercise
  id, workout_id, exercise_id, sort_order, superset_group, notes

workout_set                          -- the *fact*
  id, workout_exercise_id, sort_order, set_type,
  weight_kg, reps, distance_m, duration_s, rpe, completed_at

body_measurement
  id, kind, value, unit, measured_at        -- weight, bodyfat, waist, arms, ...
```

**Notes that matter:**

- `workout.routine_id` is a weak reference. Deleting a routine must not delete history.
- `workout_set` is append-mostly and is the largest table by far. A serious lifter puts
  down ~25 sets/session × 4 sessions/week × 52 weeks ≈ **5,200 rows/year**. Trivially
  small. Don't over-engineer for scale; do index `(exercise_id, completed_at)` because
  every chart in the app is that query.
- Store `weight_kg` canonically. Display units are a user preference, never a storage
  concern. The number of apps that get this wrong and then can't change units is high.
- `bodyweight_kg` snapshotted on the workout is what makes historical bodyweight-exercise
  volume correct.

---

## 4. The math

### Estimated 1RM

Strong uses **Brzycki** ⚠️ Verify:

```
e1RM = weight / (1.0278 − 0.0278 × reps)
```

Epley is the common alternative and diverges above ~10 reps:

```
e1RM = weight × (1 + reps / 30)
```

Both are unreliable past ~12 reps — cap the rep count you'll estimate from (10 is a sane
limit) and mark higher-rep sets as "not estimated" rather than producing a fantasy number.
The *trend* in e1RM matters far more than its absolute accuracy, which is the argument for
picking one formula and never changing it.

### Volume

```
volume = Σ (weight × reps)        over non-warm-up sets
```

For bodyweight exercises, `weight` = bodyweight + added (or bodyweight − assistance).
Per-muscle volume attributes each set to its exercise's primary muscle; "fractional"
attribution to secondary muscles (0.5 credit) is more scientifically defensible and more
annoying to explain in a UI. Pick primary-only for v1.

**Set count per muscle per week** is arguably a better training metric than tonnage and is
trivially cheaper to compute. Show both.

### PRs

Strong tracks several, and separating them is the right call because they answer different
questions:

- **Heaviest weight** for the exercise (any reps)
- **Best e1RM**
- **Best volume in a single set** (weight × reps)
- **Best weight at each rep count** — the "rep max table" (best 1RM, best 3RM, best 5RM…).
  This is the most useful one and the one most apps skip.
- **Best session volume** for the exercise

PRs should be computed on write (flag the set) *and* be re-derivable from scratch, because
editing history must be able to revoke a PR.

---

## 5. What Strong does well, and what it doesn't

**Does well**
- Logging speed. Nothing beats it. Every design choice is subordinated to this.
- Offline-first. It works in a basement gym with no signal, always.
- Data export. A full CSV of every set you've ever logged (see below).
- Restraint. No feed, no gamification, no coach nagging you.

**Doesn't do well — and these are the openings for a personal build**
- **No programming engine.** It cannot express "Week 3 of 5/3/1, 85% × 3+". There is no
  progression logic, no percentage-of-1RM targets, no periodization, no auto-regulation.
  You maintain the program in your head or a spreadsheet, and Strong just records it.
- **Shallow analytics.** Volume and e1RM charts and little else. No fatigue modelling, no
  weekly set-count-per-muscle view, no plateau detection.
- **Routine editing is clunky.** Reordering, superset creation, and bulk edits are fiddly.
- **Paywalls basic charts.** Progress graphs behind PRO is an aggressive cut.
- **No web app.** Reviewing training history on a laptop is where a big screen genuinely
  helps, and it isn't an option.
- **No nutrition context whatsoever.** It doesn't know you're in a 500 kcal deficit, so it
  can't tell you that your stalled bench is expected rather than alarming. This is the gap
  the combined app exists to fill — see [doc 08](08-combined-app-design.md).

---

## 6. Getting your history out

Strong exports a CSV of every set. Columns are roughly ⚠️ Verify against a real export:

```
Date, Workout Name, Duration, Exercise Name, Set Order,
Weight, Reps, Distance, Seconds, Notes, Workout Notes, RPE
```

One row per set, workout-level fields repeated on every row. It's denormalized and a
little lossy (superset grouping and set types are not cleanly represented), but it is
enough to reconstruct history.

**Build the importer early.** Starting a personal training app with an empty history is
demoralizing and makes every chart useless for months. Importing years of Strong data on
day one means the analytics are worth looking at immediately. This is a Phase 3 deliverable
in [doc 09](09-execution-plan.md).

---

## 7. Worth studying alongside Strong

- **Hevy** — the closest competitor, arguably a better free tier and a nicer routine
  editor. Adds a social feed. Its exercise-picker UX is worth stealing.
- **Boostcamp** — solves what Strong won't: real programs (5/3/1, nSuns, PPL) with
  week-by-week progression and percentage-based targets.
- **Liftosaur** — a program *language*. Programs are literally scripted with a small DSL.
  Overkill for most people, but the right idea if you run structured programming.
- **Apple Fitness / Fitbod** — Fitbod's auto-generated sessions based on recovery are the
  most interesting thing in the category, and the least trustworthy.

**The synthesis for a personal app:** Strong's logging screen, Boostcamp's program model,
Hevy's routine editor, and analytics no one currently ships.

---

## Takeaways for the build

1. The active-workout screen *is* the app. Budget disproportionate effort there.
2. The `PREVIOUS` column and tap-to-autofill are the features. Everything else is support.
3. Get `measurement_type` right before writing any logging UI.
4. Persist continuously. Never lose an in-progress session.
5. Store kg/metres/seconds canonically; units are a display concern.
6. Build the Strong CSV importer in the first month so history exists from day one.
7. The obvious feature gap worth building is **programming + real analytics**, not more
   logging polish — Strong already won that.
