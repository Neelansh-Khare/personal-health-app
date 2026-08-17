# 03 — Building for iOS and Android Simultaneously

**Short answer: yes, it's possible, and it's the normal way to build an app like this now.**
But "one codebase" never means "one platform's worth of work". Budget for it honestly:

```
┌───────────────────────────────────────────────────────────┐
│  Shared across platforms  (~85%)                          │
│  business logic · algorithms · data layer · UI · sync     │
├───────────────────────────────────────────────────────────┤
│  Per-platform, unavoidable  (~15%)                        │
│  HealthKit vs Health Connect · background execution ·     │
│  notifications · Live Activities vs foreground services · │
│  signing · store submission · device testing              │
└───────────────────────────────────────────────────────────┘
```

That 15% is where the calendar time hides. You still need two physical devices, two
toolchains, two developer accounts, and two test passes for every release. Cross-platform
frameworks compress the 85%; nothing compresses the 15%.

---

## 1. The four real options

### A. React Native + Expo

TypeScript, one codebase, native UI components. Expo's managed workflow plus config plugins
plus EAS Build handles the toolchain misery.

**Strengths**
- Fastest iteration loop in the category. Hot reload is genuinely instant, and EAS Update
  ships JS-only fixes over-the-air without a store round-trip.
- Best-in-class build/deploy tooling. `eas build --platform all` genuinely does build both.
- Enormous ecosystem, and TypeScript means your algorithms, a potential web dashboard, and
  any backend share a language.
- Strongest AI-assist quality of the four — there's more RN/TS in training data than
  anything else here, which matters when you're building this mostly by directing an agent.

**Weaknesses for *this* app**
- **The active-workout screen is RN's historical weak spot.** A scrolling list containing
  many inline numeric `TextInput`s, with keyboard avoidance, focus management, and
  tap-to-advance, is precisely the thing that has always been fiddly in React Native. It's
  workable — the New Architecture and Reanimated improved it a lot — but expect to spend
  real time there. This is the single biggest RN-specific risk in the project.
- Health integrations are community-maintained: `@kingstinct/react-native-healthkit` and
  `react-native-health-connect`. Both are good; neither is Expo-official, so you need a
  **development build** (not Expo Go) from day one, and you're exposed if a maintainer
  stops.
- Two separate health libraries to learn and wrap, versus Flutter's one.

**Ecosystem** — DB: `expo-sqlite` or `op-sqlite` + Drizzle ORM · Charts: Victory Native XL
(Skia) · Barcode: `expo-camera` · Background: `expo-background-task` + `expo-notifications`

---

### B. Flutter

Dart, one codebase, everything drawn by the Impeller engine rather than mapped to native
widgets.

**Strengths for *this* app**
- **Pixel-identical on both platforms, and fast.** For a data-dense app full of custom
  charts, tables, and a bespoke logging screen, "we draw everything ourselves" is an
  advantage, not a compromise. Charts and 60/120fps scrolling are simply easier here.
- **One health package for both platforms.** The `health` package wraps HealthKit *and*
  Health Connect behind a single Dart API. That is a meaningful reduction in the
  per-platform 15%, and it's the strongest single argument for Flutter here.
- **`drift` is the best local database library in any of these ecosystems** — type-safe,
  codegen'd, reactive queries, proper migrations. For a local-first app that matters a lot.
- Text input in lists is a solved problem, not a known pain point.

**Weaknesses**
- Dart is a language you'd use only for this. No sharing with a web dashboard or backend.
- Smaller ecosystem overall; for niche needs you write the platform channel yourself.
- Slightly weaker AI-assist quality than TypeScript, though still good.
- Google's long-term commitment gets questioned periodically. Development remains very
  active and the framework is widely used in production; treat this as a mild risk, not a
  disqualifier. ⚠️ Verify current state before committing.

**Ecosystem** — DB: `drift` · Charts: `fl_chart` · Barcode: `mobile_scanner` (MLKit) ·
Background: `workmanager` / `flutter_background_service`

---

### C. Kotlin Multiplatform + native UI

Share the domain layer, data layer, and algorithms in Kotlin. Write SwiftUI on iOS and
Jetpack Compose on Android.

**Strengths**
- **Your algorithms get written and tested once**, in a real typed language, and they are
  the crown jewels here — the Kalman filter, e1RM math, macro allocation, sync engine.
- **Native UI on both platforms.** The logging screen is as good as it can possibly be.
- **Health Connect integration is literally native Kotlin** — no wrapper, no bridge, full
  API surface, day-one access to new record types.
- SQLDelight (or Room KMP) gives you one schema and one set of queries across both.
- Compose Multiplatform is now a viable escape hatch if you later decide to share UI too.

**Weaknesses**
- **You write the UI twice.** For an app that's ~70% UI, that's a large tax.
- Swift↔Kotlin interop has rough edges — generics, coroutines, and error types translate
  awkwardly. SKIE smooths this considerably but it's another tool in the chain.
- Smallest community of the four; the fewest worked examples for an agent to draw on.
- Build setup is the most complex here by a distance.

**Best fit if:** you care most about the two UIs being excellent, and you have enough
non-trivial shared logic to justify the machinery. This app does have that logic.

---

### D. Full native — Swift/SwiftUI + Kotlin/Compose

Two apps. Nothing shared but the schema design and your intent.

**Strengths**
- Ceiling on quality is highest, full stop. Best logging screen, best HealthKit, best
  Health Connect, best Live Activities and widgets, best watchOS and Wear OS.
- Zero dependency risk on a framework or a community wrapper.
- **Cleanest match to "iOS first, Android later"** — you just don't start the Android app.
- SwiftUI + HealthKit is very well represented in training data; Compose + Health Connect
  is decent.

**Weaknesses**
- Roughly **1.8–2×** the work, and worse, 2× the *ongoing* work. Every feature, every bug,
  twice, forever. For a personal project you'll touch a few times a quarter, this
  compounds badly.
- The Kalman filter, the sync engine, and the food-matching logic all get implemented
  twice, with two sets of bugs and two sets of tests.

---

### Honourable mentions (and why they're out)

- **Capacitor / Ionic** — a web app in a native shell. The logging screen would feel wrong,
  and health APIs are entirely plugin-dependent. Out.
- **.NET MAUI** — works, but the health-integration ecosystem is thin and you'd be writing
  the HealthKit and Health Connect bindings yourself. Out unless you already love C#.
- **Compose Multiplatform (shared UI incl. iOS)** — increasingly viable, and a real option
  if you like Kotlin. Treat it as a variant of option C rather than a separate path.

---

## 2. Comparison on the axes that matter for *this* app

Scored 1–5. Weights reflect this specific app, not apps in general.

| Axis | Weight | RN+Expo | Flutter | KMP | Native ×2 |
|---|---|---|---|---|---|
| Active-workout screen (lists + inline inputs) | 5 | 3 | 4 | 5 | 5 |
| HealthKit + Health Connect depth | 5 | 3 | 4 | 5 | 5 |
| Effort to cover *both* platforms | 5 | 5 | 5 | 3 | 1 |
| Local DB / offline-first | 4 | 4 | 5 | 5 | 4 |
| Charts & analytics rendering | 4 | 3 | 5 | 4 | 4 |
| Background exec, Live Activities, notifications | 3 | 3 | 3 | 5 | 5 |
| Barcode scanning | 3 | 5 | 5 | 4 | 4 |
| Build/CI/deploy tooling | 3 | 5 | 4 | 3 | 3 |
| Ecosystem + AI-assist quality | 4 | 5 | 4 | 3 | 4 |
| Long-term maintenance burden (personal project) | 4 | 4 | 4 | 3 | 2 |
| watchOS / Wear OS (v2 concern) | 1 | 2 | 2 | 4 | 5 |
| **Weighted total** | | **~150** | **~166** | **~161** | **~141** |

**Read this as a thinking tool, not a verdict.** The scores are close enough that the
weights are doing most of the work, and the weights are judgement calls. What the table
actually shows:

- **Flutter and KMP score highest**, for opposite reasons — Flutter by removing per-platform
  work, KMP by maximizing per-platform quality.
- **RN+Expo's deficit is concentrated in exactly one place**: the logging screen. If that
  turns out to be fine in practice, RN moves to the front on tooling and ecosystem alone.
- **Native ×2 loses on effort, not quality.** For a personal app maintained in spare time,
  effort *is* the binding constraint.

---

## 3. How to actually decide: the Phase 0 spike

Don't decide from this table. Decide from evidence, in about a week. This is
[doc 09](09-execution-plan.md) Phase 0.

**Build the same vertical slice in two candidates. Timebox: 2–3 days each, hard stop.**

The slice — deliberately chosen to hit every risky part of the app at once:

1. A screen listing 3 exercises, each with 4 set rows containing **inline weight and reps
   inputs**, a "previous" column, and a completion checkmark.
2. Tapping ✓ autofills from previous, marks complete, and **starts a 90-second rest timer
   that survives backgrounding** and fires a notification.
3. Every edit **persists to SQLite immediately**. Kill the app mid-set; reopen; state is
   intact.
4. On finish, **write an HKWorkout / ExerciseSessionRecord** to the platform health store,
   and read your bodyweight back out of it.
5. Render one chart: estimated 1RM over 20 fake sessions.
6. Get it onto a **real device** through the real distribution channel (TestFlight and a
   sideloaded APK), not a simulator.

**Acceptance criteria — write these down before you start, and be strict:**

| Criterion | Bar |
|---|---|
| Set-entry latency | Typing feels instant; no dropped keystrokes; keyboard never obscures the active row |
| Scroll performance | 60fps with 40 set rows on the oldest device you own |
| Rest timer | Fires reliably with the app backgrounded and the screen locked |
| Crash-resilience | Force-quit mid-set loses nothing |
| Health write | Workout appears in Apple Health / Health Connect with correct type and duration |
| Chart | Renders 200 points without jank |
| Device install | Both platforms, real distribution channel, under an hour once set up |
| Vibe check | Would you want to open this file again next weekend? |

**Which two to spike?** Recommended pairing: **Flutter vs React Native + Expo**. They're the
two that actually deliver both platforms from one codebase, they're the two you can
evaluate fastest, and the comparison is genuinely close. Add KMP as a third only if the
first two both feel wrong on the logging screen.

**The one thing that should decide it:** how the active-workout screen feels on a real
device with a real keyboard. Everything else in this document is secondary to that,
because that screen is the app ([doc 01](01-strong-app-breakdown.md) §2).

---

## 4. Insurance: keeping the decision cheap to reverse

Whatever you pick, structure the project so the stack choice is not load-bearing:

1. **Pure-logic layer with zero framework imports.** The Kalman filter, e1RM math, macro
   allocation, volume aggregation, food matching, and sync reconciliation are all pure
   functions over plain data. No UI types, no DB types, no HTTP. In any of the four stacks
   this is a standalone module with its own test suite. **Write these first**
   ([doc 09](09-execution-plan.md) Phase 2) — they're the most valuable and most portable
   code in the project, and porting them between languages is mechanical.

2. **One SQLite schema, expressed as raw SQL migrations.** Not as ORM model classes. The
   schema then survives an ORM change or a whole-stack change, and the database file itself
   is portable. See [doc 06](06-data-architecture.md).

3. **A single `HealthGateway` interface** with a handful of methods, implemented per
   platform ([doc 05](05-health-integrations.md) §5). Nothing else in the app knows that
   HealthKit or Health Connect exist.

4. **Screen-level, not app-level, framework coupling.** If the logging screen turns out to
   be miserable in your chosen stack, you want the option to rewrite that one screen
   natively rather than the whole app. RN and Flutter both permit embedding a native view.

Do these four things and switching stacks in month three costs you the UI layer, not the
project.

---

## 5. If you go iOS-first anyway

You said iOS first, Android second if simultaneous doesn't work out. Here's what that
actually costs under each option:

| Option | Cost of deferring Android |
|---|---|
| **RN + Expo** | Near zero. You're already building it; you just don't test or ship the Android target. Turn it on later with a few config-plugin fixes and a Health Connect implementation. |
| **Flutter** | Near zero, same reasoning. |
| **KMP** | Moderate. The shared module is already Kotlin (Android-native, in fact). You'd write the Compose UI later — a real but bounded chunk of work, and the domain layer is done. |
| **Native ×2** | Full price. The Android app is a from-scratch project. |

**The honest recommendation:** if there's any real chance you want Android, pick a
cross-platform stack now. Deferring Android inside RN or Flutter is free; deferring it in a
native Swift codebase means writing a second app from nothing, later, with less motivation
than you have today.

**The one exception:** if the spike shows the logging screen is genuinely unacceptable in
both RN and Flutter, then Swift-first is the right call — and you build the domain layer in
a way that a future Kotlin port can mirror.

---

## Takeaways

1. Simultaneous iOS + Android is normal and achievable. Roughly 85% shares; the last 15%
   is irreducibly per-platform, and it's where the schedule risk lives.
2. **Don't decide from the table. Decide from the Phase 0 spike.** Two stacks, 2–3 days
   each, one vertical slice, written acceptance criteria, real devices.
3. The deciding factor is how the **active-workout screen** feels with a real keyboard on a
   real phone. Everything else is negotiable.
4. Regardless of stack: pure-logic layer with no framework imports, raw-SQL migrations,
   one `HealthGateway` interface. That's what makes the decision reversible.
5. iOS-first is nearly free inside RN or Flutter and very expensive inside native Swift.
   Choose accordingly.
