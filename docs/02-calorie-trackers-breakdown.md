# 02 — Calorie Trackers: MacroFactor, MyFitnessPal, Cronometer

Three apps, three philosophies. The interesting one is MacroFactor, because it is the only
mainstream tracker whose core is an actual *algorithm* rather than a database plus a form.

| | MacroFactor | MyFitnessPal | Cronometer |
|---|---|---|---|
| **Built by** | Stronger by Science (Nuckols / Trexler / Helms) | Under Armour → private equity | Cronometer Software |
| **Core idea** | Estimate your real expenditure from your own data, weekly | Static equation + "eat back" exercise calories | Micronutrient accuracy |
| **Food DB** | Curated, verified, moderate size | ~20M crowd-sourced entries, much of it wrong | NCCDB + USDA, high quality, smaller |
| **Price** ⚠️ Verify | ~$11.99/mo, ~$71.99/yr, no free tier | Free w/ ads; Premium ~$19.99/mo | Free tier; Gold ~$50/yr |
| **Steal this** | The expenditure algorithm, weekly check-in ritual | DB coverage, recipe import, meal structure | Micronutrient model, data provenance |

---

## 1. The central problem every calorie tracker solves badly

You want to lose 0.5 kg/week. To do that you need to eat some number of calories. That
number is your **energy expenditure minus a deficit**. So: what's your expenditure?

**The MyFitnessPal answer** — plug your stats into Mifflin-St Jeor, multiply by an activity
factor you self-select from a dropdown, then *add back* the calories your workout tracker
claims you burned.

```
BMR  = 10×kg + 6.25×cm − 5×age + 5           (male; −161 female)
TDEE = BMR × {1.2 sedentary … 1.9 very active}
Daily target = TDEE − 500 + (exercise calories burned today)
```

This is wrong in three compounding ways:

1. **The equation has ±15–20% error on individuals.** It's a population regression. For a
   2,800 kcal person that's a ±500 kcal band — bigger than the deficit you're trying to create.
2. **The activity multiplier is a self-assessment,** and everybody picks one tier too high.
3. **Adding back exercise calories double-counts.** The activity multiplier already includes
   your exercise. Then the watch adds it again — and consumer wearables overestimate
   exercise energy expenditure substantially (error is routinely 20–50%+ for
   strength training, which they're especially bad at). A "700 calorie" lifting session that
   actually cost 250 kcal of *net* energy turns your 500 kcal deficit into a surplus.

The net effect: your daily target bounces around by hundreds of calories for reasons
unrelated to your body, you plateau, and you conclude your metabolism is broken.

**The MacroFactor answer** — don't estimate expenditure from a formula. *Measure* it, from
the only two things you actually have data on: what you ate, and what the scale said.

---

## 2. The adaptive expenditure algorithm

### The physics

Energy balance, over any window:

```
Δ(body energy) = intake − expenditure
```

Rearranged, expenditure is fully determined by things you can observe:

```
expenditure = intake − Δ(body energy)
```

Body energy change is approximated from weight change via an energy density constant:

```
ρ ≈ 7,700 kcal/kg   (3,500 kcal/lb)
```

So the naive estimator over an *n*-day window is:

```
TDEE ≈ mean_daily_intake − (Δ_trend_weight_kg × ρ) / n
```

**Worked example.** Over 14 days you averaged 2,000 kcal/day and your trend weight fell
0.5 kg:

```
TDEE = 2000 − (−0.5 × 7700) / 14
     = 2000 + 275
     = 2,275 kcal/day
```

No formula, no activity multiplier, no dropdown. That number already contains your NEAT,
your training, your job, your fidgeting, and your particular metabolism.

### Why "trend weight" and not scale weight

Daily scale weight has a standard deviation of roughly **0.7–1.2 kg** from water, glycogen,
gut contents, sodium, and menstrual cycle. Fat mass changes at maybe 0.07 kg/day. **The
noise is 10–15× the signal.** Any algorithm fed raw scale weight produces garbage, and any
*user* shown raw scale weight makes bad decisions.

Two ways to smooth it:

**(a) Exponentially weighted moving average** — simple, adequate, one line of code:

```
trend_t = α × weight_t + (1 − α) × trend_{t−1}        α ≈ 0.10
```

α = 0.10 gives roughly a 10-day time constant. Downside: it lags. During rapid change the
trend sits behind reality by several days, which matters when the trend is an *input to a
weekly decision*.

**(b) Kalman filter** — more code, meaningfully better, and it gives you a confidence
interval for free. This is the recommended approach and it's a beautiful fit for the
problem because the energy-balance equation *is* the state transition.

### The Kalman formulation (recommended)

State is two things you can't observe directly:

```
x = [ W ]     true weight (kg)
    [ E ]     true expenditure (kcal/day)
```

**Transition** — tomorrow's weight is today's weight adjusted by yesterday's energy
balance; expenditure is a slow random walk:

```
W_t = W_{t−1} + (I_{t−1} − E_{t−1}) / ρ
E_t = E_{t−1} + w_E                        w_E ~ N(0, σ²_E)

F = [ 1   −1/ρ ]      B = [ 1/ρ ]  applied to control input I_{t−1}
    [ 0    1   ]          [  0  ]
```

**Observation** — the scale sees weight plus a lot of noise; expenditure is never observed:

```
z_t = W_t + v_t        v_t ~ N(0, σ²_scale)

H = [ 1  0 ]
```

**Starting parameter values** (tune against your own data):

| Parameter | Value | Reasoning |
|---|---|---|
| ρ | 7,700 kcal/kg | Standard fat-mass energy density |
| σ_scale | 0.9 kg | Measured day-to-day scale SD |
| σ_E (process) | ~20 kcal/day per day | How fast can true TDEE actually drift? Slowly. |
| σ_W (process) | ~0.05 kg/day | Unmodelled real weight change |
| Initial E | Mifflin-St Jeor × 1.4 | A prior, discarded within ~3 weeks of data |
| Initial P | diag(1.0, 300²) | Wide — say loudly that you don't know E yet |

**Why this beats the rolling window:**

- **Missing weigh-ins** are handled natively: run the predict step, skip the update step.
  Uncertainty grows, which is correct.
- **Missing intake days** are handled by inflating process noise for that step rather than
  by pretending intake was zero (which would be catastrophic and is a real bug in
  naive implementations).
- **You get uncertainty.** `P[1][1]` is the variance of your expenditure estimate. Show it.
  "2,610 kcal ± 180" after two weeks, tightening to "± 60" after two months, is honest and
  it tells the user when to trust a program change.
- **It self-tunes the smoothing.** The Kalman gain automatically weights recent data more
  when the estimate is uncertain and less when it's settled — which is exactly the adaptive
  behaviour you'd otherwise hand-roll.

### The under-reporting insight (this is the important part)

Everybody under-reports intake. Studies put habitual under-reporting at 10–30%. You'd
expect this to destroy the algorithm. **It mostly doesn't, and understanding why is the
key to trusting the whole approach.**

Suppose you truly eat 2,500 kcal but log 2,000 — a constant 500 kcal under-report. The
algorithm observes intake 2,000 and a flat weight, so it concludes your expenditure is
2,000. That's wrong by 500 kcal in absolute terms. But now it prescribes a 500 kcal deficit
as "eat 1,500 logged calories" — and since you under-report by 500, you'll actually eat
2,000, against a true expenditure of 2,500. **You get exactly the 500 kcal deficit you
asked for.** The bias cancels.

The estimate is wrong; the *prescription* is right. That is a genuinely elegant property
and it's why the algorithm works on real humans instead of only on people with food scales.

**What does break it:**

- **Drifting bias.** Getting sloppier (or stricter) over time is a change the model reads
  as a change in metabolism. It will chase it and give you wrong targets.
- **Weekend-only logging.** Systematically missing your highest-intake days is drifting bias
  in disguise.
- **Non-fat weight change.** See below.

### The early-diet trap

Week 1 of any deficit produces 1–2 kg of loss that is glycogen and its associated water,
not fat. Glycogen binds roughly 3 g of water per gram. The naive estimator sees a huge
"energy" loss and concludes your TDEE is enormous — then prescribes a target that's several
hundred calories too high, and you stall in week 3 wondering what happened.

Mitigations, roughly in order of preference:

1. **Detect phase changes** (deficit → surplus or vice versa) and inflate the weight process
   noise for ~14 days afterward so the filter attributes the swing to noise rather than to E.
2. Require a minimum of ~14 days of data before showing an expenditure number at all.
3. Down-weight or exclude the first two weeks of a new phase from the estimate.

The same thing happens in reverse when you start a surplus, and around creatine loading,
carb refeeds, high-sodium days, and illness.

---

## 3. From expenditure to targets

Once you have `E` (with uncertainty), the weekly program update is straightforward arithmetic.

**Step 1 — goal rate.** Expressed as % of bodyweight per week, not absolute kg, because it
scales correctly across body sizes.

```
Cutting:   0.5 – 1.0 %BW/week    (1.0% is aggressive; >1% costs lean mass)
Gaining:   0.125 – 0.5 %BW/week  (0.25% is the sane default; faster is mostly fat)
Maintain:  0
```

**Step 2 — calorie target.**

```
daily_delta  = (goal_rate × bodyweight_kg × ρ) / 7
kcal_target  = E − daily_delta          (delta positive when cutting)
```

Example: 85 kg, cutting at 0.6%/week, E = 2,900.
```
weekly_kg    = 0.006 × 85 = 0.51 kg
daily_delta  = 0.51 × 7700 / 7 = 561
kcal_target  = 2900 − 561 = 2,339 kcal
```

**Step 3 — guardrails.** Non-negotiable, because the algorithm will happily walk you into
a 900 kcal target if your logging is bad:

```
floor    = max(1200,  22 × bodyweight_kg,  0.85 × estimated_BMR)
max Δ    = ±15% change to the target in any single week   (avoid whiplash)
```

**Step 4 — macro split.**

```
protein_g = clamp(1.6 … 2.2 g per kg bodyweight)      ← higher end when cutting
                 (use lean or goal bodyweight if BF% is high)
fat_g     = max(0.6 g/kg,  0.20 × kcal_target / 9)    ← hormonal/absorption floor
carb_g    = (kcal_target − protein_g×4 − fat_g×9) / 4 ← remainder
fibre_g   = 14 g per 1000 kcal
```

Protein and fat are floors; carbs are the flex. If the remainder goes negative, the target
is too low — raise the floor or reconsider the goal rate.

**Step 5 — the weekly check-in.** Present it as a ritual, once a week, same day:

> Last week you averaged **2,285 kcal** and your trend weight went **83.4 → 83.0 kg**
> (−0.48%, target was −0.5%). Your expenditure estimate moved **2,840 → 2,875 ± 90**.
> New targets: **2,320 kcal · 175 P · 68 F · 245 C**.

That single screen is the product. It's also what a spreadsheet can't do for you.

### The coaching modes

MacroFactor offers three, and it's a good pattern to copy because it lets the algorithm
be helpful without being authoritarian:

- **Coached** — algorithm sets calories and macros weekly. You just log.
- **Collaborative** — algorithm proposes; you accept or adjust. Its estimate keeps updating.
- **Manual** — you set everything. The expenditure estimate is still computed and displayed
  as information, which is genuinely useful on its own.

For a personal app, build **Collaborative** first. It's the honest default: the algorithm
knows more than you about your expenditure, and you know more than it about your life.

### Why there's no "exercise calories" concept

Worth stating explicitly because it's counterintuitive to anyone coming from MFP: in an
adaptive-expenditure model **you never log exercise for calorie purposes and you never eat
calories back.** Your training is already inside `E` — it's part of what caused the weight
trend the estimate is derived from. Adding it separately double-counts, exactly as MFP does.

Workouts are logged for *training* reasons (progression, volume, PRs), not energy reasons.
Which is convenient, because the combined app has an excellent workout logger sitting right
next to it and no temptation to wire it into the calorie math.

---

## 4. Food logging UX — where the app is actually used

The algorithm runs once a week. Logging happens 4–6 times a day, and if it takes more than
~15 seconds people quit. Ranked by impact:

1. **Recents / frequents, surfaced first.** Most people eat ~150 distinct foods, ever, and
   perhaps 30 in a given month. The search box should be the *fallback*, not the entry point.
   An empty-state list of "your usual" covers the large majority of logging events.
2. **Remembered portions.** If you always log 180 g of chicken breast, the quantity field
   should pre-fill with 180 g. Store a per-(user, food) last/modal quantity.
3. **Barcode scan.** One tap from the log screen, camera opens, done. See
   [doc 07](07-food-database.md) for the checksum and UPC/EAN pitfalls.
4. **Copy a previous day / previous meal.** People eat the same breakfast for months.
   "Copy yesterday's breakfast" is one tap replacing six.
5. **Meals / recipes.** A saved combination logged as one unit. Recipes need a servings
   count and per-serving nutrient derivation.
6. **Quick add macros.** For restaurant meals you're estimating. Log "600 kcal / 30 P / 25 F
   / 60 C" with no food entity at all. Essential escape hatch — without it people log
   nothing on the days that matter most.
7. **Meal sections** (breakfast / lunch / dinner / snacks). Mostly organizational, but they
   make the day scannable and they map cleanly onto HealthKit and Health Connect meal types.
8. **Natural-language / photo logging.** "two eggs, toast with butter, a coffee" → parsed
   entries. MacroFactor and MFP both ship AI versions of this now. High effort, high delight,
   and firmly a v2 feature.

### The diary-day problem

A snack at 1:00 AM belongs to *yesterday's* diary. Store `log_date` (a local calendar date)
separately from `logged_at` (a UTC instant). Do not derive one from the other at query time,
and let the user set the day-boundary hour. This is a small decision that is extremely
annoying to retrofit — see [doc 06](06-data-architecture.md).

---

## 5. What each app is worth stealing from

**From MacroFactor**
- The expenditure algorithm and trend weight, in full.
- The weekly check-in as a ritual screen with a narrative, not a number.
- Showing uncertainty rather than false precision.
- Curated food data over crowd-sourced volume.
- The discipline of *not* adding exercise calories back.

**From MyFitnessPal**
- Database coverage — it's the only real reason anyone stays.
- Recipe import from a URL.
- Meal-level diary structure.
- Streaks. They're cheap and they work.

**From Cronometer**
- Micronutrient tracking against RDAs, with the data quality to justify it.
- Explicit data provenance on every entry — you can see *which* database an entry came from
  and how trustworthy it is. Underrated, and easy to build if you plan for it in the schema.

**Deliberately from none of them**
- Ads, streak-guilt notifications, social feeds, and "you have 47 calories remaining, keep
  it up!" Nobody building for themselves wants this.

---

## Takeaways for the build

1. Implement the **Kalman filter** for `[weight, expenditure]`. It's ~100 lines of pure,
   testable, stack-agnostic code and it is the single highest-value thing in the app.
2. Show **trend weight** everywhere. Show raw scale weight only as faint dots behind it.
3. Show **uncertainty** on the expenditure estimate. It's the honest thing to do and it
   prevents over-reacting to two weeks of data.
4. Never add exercise calories back. Log workouts for training, not for energy.
5. Guardrails on targets: calorie floor, max weekly change, protein and fat floors.
6. Handle the early-diet glycogen swing explicitly, or the estimate lies for a month.
7. Optimize the log screen for **recents + remembered portions**, not for search.
8. Store `log_date` separately from `logged_at` from day one.
9. Snapshot nutrients onto the log entry — see [doc 07](07-food-database.md) for why.
