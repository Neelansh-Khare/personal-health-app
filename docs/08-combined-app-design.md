# 08 — The Combined App

What Strong and MacroFactor look like as one application, built for one person.

---

## 1. The thesis

You could just pay for both. Strong PRO plus MacroFactor is roughly $110/year, both are
excellent, and neither is going away.

So the combined app has to be worth more than the sum, or there's no reason to build it. It
is — because **the two apps are sitting on data the other one needs, and neither can see it.**

```
        Strong knows                    MacroFactor knows
  ┌──────────────────────────┐    ┌──────────────────────────┐
  │  every set, rep, load    │    │  every calorie, macro    │
  │  training volume         │    │  trend weight            │
  │  session frequency       │    │  actual expenditure      │
  │  strength trajectory     │    │  diet phase & rate       │
  └────────────┬─────────────┘    └────────────┬─────────────┘
               │                                │
               └──────────────┬─────────────────┘
                              ▼
                  Questions neither can answer
```

Every one of the following is trivial to compute with both datasets and impossible with
either alone. This list *is* the product:

**1. Is my strength holding up during this cut?**
Chart estimated 1RM with diet-phase bands shaded behind it. "You lost 6.2 kg over 14 weeks
and your squat e1RM went from 165 → 162 kg." That is the single question every lifter who
diets actually has, and no shipping app answers it.

**2. What does training actually cost me?**
Your expenditure estimate already contains your training. Regress it against weekly training
volume and you get a real, personal number: *"weeks where you train 5× run about 190 kcal/day
higher than weeks where you train 3×."* Not a MET-table guess — your own data.

**3. Am I actually recomping?**
Weight flat + strength up + waist down = recomposition, the thing every tracker misreports
as "no progress" because it only sees the scale.

**4. Relative strength over time.**
e1RM ÷ bodyweight, and DOTS/IPF-style scores. Requires both datasets by definition, and it's
about ten lines of code once you have them.

**5. Why did I stall?**
Bench flat for three weeks. The app can look at deficit size, trend-weight rate, weekly set
count, session RPE, sleep, and steps, and say which of them changed. Not a diagnosis — a
short list of candidates, which is far more than "you have plateaued 😞".

**6. Am I fuelling the hard days?**
Protein in g/kg on training days vs rest days. Calories on the day before a heavy session.
Most people under-eat on exactly the days it matters and never notice.

**7. Fatigue and deload signals.**
Trend weight dropping faster than target *and* RPE rising at unchanged loads *and* volume
falling = a deload, or a diet break, or both. Combining the signals is what makes the call
credible.

**8. Correct bodyweight-exercise volume.**
Pull-up "volume" needs your bodyweight *on that day*. Only a combined app has it.

**9. One weekly check-in instead of two.**
Training and nutrition reviewed together, in one ritual, because they're one system.

That's the app. Everything below is in service of those nine things.

---

## 2. Design principles

1. **Never lose data.** The one unforgivable failure. Persist continuously, back up
   automatically, test the restore.
2. **Offline is the normal case**, not the degraded one. Gyms are basements.
3. **One thumb, thirty seconds, sweaty hands.** Every gym interaction is designed for this.
4. **No nagging.** No streak guilt, no notifications about calories, no coach persona. It's
   an instrument, not a personality.
5. **Honest uncertainty.** "2,610 ± 180 kcal" beats a confident wrong number. Show
   confidence intervals; grey out insights that don't have enough data yet.
6. **Everything is exportable.** You should be able to leave your own app.
7. **Insights are earned, not decorative.** Every card on the insights screen must be
   traceable to real data, and must say so when it isn't confident.

---

## 3. Information architecture

Five tabs. Resist a sixth.

```
┌──────┬───────┬────────┬────────┬──────────┐
│Today │ Train │  Food  │  Body  │ Insights │
└──────┴───────┴────────┴────────┴──────────┘
```

- **Today** — the hub. What's left to eat, what's planned to train, trend weight.
- **Train** — routines, the active workout screen, training history.
- **Food** — the diary, log entry, foods/recipes/meals.
- **Body** — weigh-ins, trend chart, expenditure chart, measurements, photos.
- **Insights** — the nine things above, plus the weekly check-in.

Settings lives behind an icon on Today. It is not a tab.

### Today

```
┌────────────────────────────────────────────┐
│  Monday, 17 August                    ⚙︎   │
├────────────────────────────────────────────┤
│                                            │
│         ╭──────────╮      1,840 / 2,320    │
│         │   79%    │      480 kcal left    │
│         ╰──────────╯                       │
│    P 142/175 ▓▓▓▓▓▓▓▓░░   F 58/68 ▓▓▓▓▓▓▓░│
│    C 178/245 ▓▓▓▓▓▓░░░░   Fibre 24/33      │
│                                            │
│    [ + Log food ]              [ Scan 📷 ] │
├────────────────────────────────────────────┤
│  TRAINING                                  │
│  Push Day A · 5 exercises · last Thu       │
│    [ ▶ Start workout ]                     │
├────────────────────────────────────────────┤
│  BODY                                      │
│  83.4 kg trend  ▼0.4 this week (−0.48%)   │
│  Expenditure 2,875 ± 90 kcal               │
│  ⚪️ Not weighed in today   [ + Weigh in ]  │
├────────────────────────────────────────────┤
│  💡 Squat e1RM up 2.5 kg since this cut   │
│     started 6 weeks ago.                   │
└────────────────────────────────────────────┘
```

One screen, four verbs — log food, start training, weigh in, and one earned insight. If a
day's interaction never leaves this screen, the design is working.

### Train → active workout

Fully specified in [doc 01](01-strong-app-breakdown.md) §2. This is the screen the project
lives or dies on and the one the Phase 0 stack spike exists to evaluate. Non-negotiables:
the `PREVIOUS` column, tap-to-autofill, auto-starting rest timer that survives backgrounding,
and continuous persistence.

### Food → log

Search is the fallback. The default view is **recents and frequents**, because that's what
you're actually going to log. Barcode scan is one tap from the diary. Quantity pre-fills to
your learned personal portion ([doc 07](07-food-database.md) §7). Quick-add macros always
available as an escape hatch.

### Body

Raw weigh-ins as faint dots; the **trend line** is the foreground. Expenditure charted with
its uncertainty band. Diet phases shaded behind both. Measurements and photos on a secondary
tab.

### Insights → the weekly check-in

The ritual. Once a week, same day, one screen:

```
┌────────────────────────────────────────────┐
│  Week 6 · Cut                              │
├────────────────────────────────────────────┤
│  Nutrition                                 │
│  Averaged 2,285 kcal (target 2,320) · 6/7  │
│  days logged · protein hit on 5/7          │
│                                            │
│  Body                                      │
│  Trend 83.8 → 83.4 kg  (−0.48%, target     │
│  −0.50%) · on track                        │
│  Expenditure 2,840 → 2,875 ± 90            │
│                                            │
│  Training                                  │
│  4 sessions · 18 sets chest/back/legs      │
│  Squat e1RM 162 → 164 kg  ▲                │
│  Bench e1RM 118 → 118 kg  —  (3 weeks flat)│
│                                            │
│  💡 Bench has been flat for 3 weeks. Your  │
│     deficit is on target and volume is     │
│     unchanged — but bench sets dropped     │
│     from 14 to 9 per week in that period.  │
│                                            │
│  New targets                               │
│    2,340 kcal · 175 P · 68 F · 249 C       │
│              [ Accept ]  [ Adjust ]        │
└────────────────────────────────────────────┘
```

**That screen is the product.** Nothing else in the app is unavailable elsewhere.

---

## 4. Version plan

Ruthless about what "done" means at each stage. Maps directly onto phases in
[doc 09](09-execution-plan.md).

### v0.1 — "It's on my phone" (Phase 3)
Training logging, local only.
- Exercise library (seeded), custom exercises
- Routines: create, edit, reorder
- Active workout: previous column, autofill, rest timer, continuous persistence
- Workout history, per-exercise history
- **Strong CSV import** — so history exists from day one
- Export to JSON

*Not included:* nutrition, charts, health integration, sync, cloud, accounts.
Done when you've used it for every session for two weeks and haven't opened Strong.

### v0.2 — Nutrition (Phase 4)
- Bundled USDA food DB with FTS search
- Barcode scan → OFF public API → permanent local cache
- Custom foods, recipes, saved meals, quick-add
- Diary with meal sections, copy-previous-day
- Learned portions
- Manual weigh-in

*Not included:* the algorithm. Targets are manually set for now.
Done when you've logged every day for two weeks and haven't opened MacroFactor.

### v0.3 — The algorithm (Phase 5, using Phase 2's code)
- Kalman trend weight + expenditure estimate with uncertainty
- Weekly target generation, guardrails, three coaching modes
- The weekly check-in screen
- Diet phases

Done when a week's targets are generated automatically and you trust them.

### v0.4 — Health integrations (Phase 6)
- Read bodyweight (smart scale → no more manual weigh-ins), steps, active energy, workouts
- Write nutrition and workouts back
- Both platforms behind one `HealthGateway`

### v0.5 — Sync and backup (Phase 7)
- Supabase backend, outbox sync, multi-device
- Automated weekly export to cloud storage
- **Tested restore path**

### v1.0 — The payoff (Phase 8)
The nine insights from §1. This is the version that justifies the project.

### v2 and beyond — only if you still want them
- **Programming engine.** Percentage-based targets, week-over-week progression, 5/3/1 and
  friends. The biggest genuine gap in Strong ([doc 01](01-strong-app-breakdown.md) §5).
- **Apple Watch / Wear OS** logging. Native work regardless of stack.
- **Natural-language food logging.** An LLM call over the database you already have.
- **Web dashboard.** Reviewing a year of training on a laptop is where a big screen earns
  its keep.
- Self-hosted OFF mirror, micronutrients, progress photos with alignment.

---

## 5. Explicit non-goals

Writing these down is what keeps the project finishable.

| Not building | Why |
|---|---|
| Social feed, sharing, friends | You are the only user. |
| Streaks, badges, gamification | Nagging. Principle 4. And a real harm risk in a diet app. |
| AI coach chat persona | The weekly check-in is the coaching. A chatbot adds nothing. |
| Meal plans / recipe discovery | You know what you eat. |
| Exercise demo videos | YouTube exists. |
| Multi-user, teams, trainer mode | Enormous complexity for zero personal benefit. |
| Subscriptions, paywalls, ads | Nobody to charge. |
| Restaurant menu database | Quick-add macros covers it. |
| Micronutrient RDA tracking (v1) | Schema supports it; UI can wait. |
| Sleep / HRV / recovery scoring | Read the data for context; don't invent a readiness score. |

---

## 6. What being the only user actually buys you

Worth being explicit, because it removes a genuinely large amount of work:

- **No onboarding funnel.** A settings screen you fill in once.
- **No auth complexity.** One account, magic-link email. No password reset, no OAuth
  providers, no session management edge cases.
- **No multi-tenancy.** One row-level-security policy, not a permissions model.
- **No i18n, no analytics, no A/B tests, no feature flags, no crash-free-rate dashboards.**
- **No migration ceremony for breaking changes.** You can wipe and re-import your own data.
- **Conflicts are rare enough that last-write-wins is correct** ([doc 06](06-data-architecture.md) §4).
- **You can build for exactly your devices**, your gym, your units, your foods.

Keep basic accessibility (dynamic type, contrast, touch targets) — you'll want it at 6 a.m.
in bad light, and it's cheap when done from the start.

---

## 7. Risks, honestly

| Risk | Severity | Mitigation |
|---|---|---|
| The active-workout screen isn't fast enough in the chosen stack | **High** | Phase 0 spike exists precisely to find this out before committing ([doc 03](03-cross-platform-approaches.md) §3) |
| Food database quality/coverage disappoints and logging becomes annoying | **High** | Bundled USDA core + permanent cache + easy custom foods + quick-add ([doc 07](07-food-database.md)) |
| Project stalls at 70% and you go back to the real apps | **High** | v0.1 must fully replace Strong before nutrition work starts. Ship usable increments, never a big bang. |
| Data loss during a migration or sync bug | **Critical** | Copy-before-migrate, tested restore, versioned exports ([doc 06](06-data-architecture.md) §6–7) |
| TDEE algorithm gives bad targets and you stop trusting it | Medium | Guardrails, uncertainty display, Collaborative mode by default, validate against your own historical data before trusting it |
| Health integration eats a month | Medium | Thin `HealthGateway`, logic above it in testable pure functions ([doc 05](05-health-integrations.md) §5) |
| Scope creep into a "real product" | Medium | §5 non-goals. Re-read it monthly. |

**The biggest risk on that list is the third one.** Personal projects don't fail on
technology; they fail by staying unusable long enough that you stop caring. Which is why the
plan in [doc 09](09-execution-plan.md) is ordered to give you a working, daily-driver
training logger before anything else exists.

---

## Takeaways

1. The reason to build this is the **nine cross-domain insights** in §1. Everything else is
   table stakes you're rebuilding to get there.
2. The **weekly check-in screen** is the product. Design it first, on paper.
3. Five tabs, one hub screen, four verbs. Resist growth.
4. **v0.1 is training-only and must fully replace Strong** before nutrition starts.
5. The non-goals list is what makes this finishable. Keep it.
6. Being the only user removes an enormous amount of work — take all of it.
7. The real risk is stalling at 70%, so every version must be independently usable.
