# 04 — Productionizing and Deploying on iOS and Android

You've chosen **personal use now, public App Store later**. That's two different problems,
and the good news is that the second one is mostly paperwork you can defer — provided you
don't paint yourself into a corner in the meantime. This doc covers both, and flags the
handful of early decisions that are expensive to reverse.

---

## 1. The two tiers

| | **Tier 0 — Personal** | **Tier 1 — Public** |
|---|---|---|
| iOS channel | Dev-signed build, or TestFlight | App Store |
| Android channel | Sideloaded signed APK | Play Store |
| Review | None (TestFlight internal), or light | Full App Review + Play review |
| Cost | $99/yr (Apple) + $0 (Android) | + $25 one-time (Play) |
| Privacy paperwork | None | Nutrition labels, Data Safety, privacy policy, Health Connect declaration |
| Legal | None | Privacy policy, terms, medical disclaimer, account deletion |
| Realistic setup time | An afternoon | 1–2 weeks of non-coding work |

**Get to Tier 0 in month one.** An app you can't put on your phone isn't real, and dogfooding
a training app requires it to be on your phone at the gym. Tier 1 is a later, separate
project ([doc 09](09-execution-plan.md) Phase 9).

---

## 2. Tier 0 — getting it on your own phones

### iOS

You need the **Apple Developer Program, $99/year**. Free provisioning technically works but
caps you at 3 apps, expires builds after 7 days, and gates several capabilities behind paid
membership — you'd spend more time re-signing than building. ⚠️ Verify whether the HealthKit
entitlement specifically is available on free provisioning; either way, $99 is the practical
answer.

Three ways to get a build onto your device, in increasing order of ceremony:

**(a) Direct development install — lowest friction, best for daily iteration.**
Build to a connected device from Xcode (or `eas build --profile development` + install).
A development provisioning profile from a *paid* account is valid for **one year**, so the
app just keeps working. This is the right default while you're building.

**(b) TestFlight — best for "it's just an app on my phone now".**
- Internal testers: up to 100, must be on your App Store Connect team, **no Beta App Review**.
- External testers: up to 10,000, requires Beta App Review on the first build of each version.
- **Builds expire after 90 days.** This is the recurring tax: a rebuild-and-upload roughly
  quarterly, forever, or the app stops launching.
- Upside: over-the-air install, no cable, easy to add a friend or your partner's phone.

**(c) Ad Hoc distribution** — signed for up to 100 specific registered devices, profile valid
one year. Useful if you want to hand someone an `.ipa` without TestFlight. Rarely worth it.

**Recommendation:** develop with (a), and once the app is stable enough to live on your phone
full-time, switch to (b) and accept the quarterly rebuild. Put a calendar reminder at 80 days.

### Android

Dramatically simpler. **No account, no fee, no expiry.**

```bash
# one-time: create an upload/signing key
keytool -genkey -v -keystore health-app.keystore \
        -alias health-app -keyalg RSA -keysize 4096 -validity 10000

# then, per release
./gradlew assembleRelease        # or: eas build -p android --profile preview
adb install -r app-release.apk   # or AirDrop-equivalent it to the phone
```

Enable "Install unknown apps" for your file manager, tap the APK, done. The install **never
expires**. Back up that keystore somewhere you won't lose it — losing it means you can never
update a Play-published app under the same identity.

**Health Connect note:** sideloaded builds can request Health Connect permissions without
Google's approval. That approval (the data-types declaration form) is a *Play Store*
requirement, so it's a Tier 1 problem. See [doc 05](05-health-integrations.md).

### The operational reality of Tier 0

Three recurring chores. Know them now so they're not surprises:

| Chore | Cadence | Consequence of forgetting |
|---|---|---|
| Rebuild + upload to TestFlight | Every 90 days | App stops launching mid-workout |
| Renew Apple Developer membership | Annually, $99 | Certificates invalidate; everything stops |
| Re-sign / rebuild after cert rotation | Annually | Dev-installed builds expire |
| **Verify your data backup works** | Monthly | The only irreversible one on this list |

That last row is the real production concern for a personal app. Nobody is paged if your
sync breaks; you find out two years later when you want a chart. See §7.

---

## 3. CI/CD

Even for a personal app, automate the build. Manual release steps are how quarterly
rebuilds get skipped.

| Stack | Recommended | Notes |
|---|---|---|
| React Native + Expo | **EAS Build / Submit / Update** | Genuinely one command for both platforms. Free tier is limited but workable; paid is cheap. |
| Flutter | **Codemagic** or GitHub Actions + Fastlane | Codemagic is Flutter-native and has a usable free tier. |
| KMP / Native | **Fastlane** on GitHub Actions, + Xcode Cloud if you like | Most assembly required. |

**A minimal pipeline that's worth having:**

```
push to main
  → run the pure-logic test suite          (fast, catches the important bugs)
  → lint + typecheck
  → build iOS + Android in parallel
  → auto-increment build number
  → upload iOS to TestFlight
  → attach the Android APK to a GitHub release
```

**Signing secrets to store in CI** (never in the repo):
- iOS: App Store Connect API key (`.p8`), key ID, issuer ID. Prefer this over
  username/password — it doesn't break on 2FA. Distribution certificate + provisioning
  profile, or let EAS/Fastlane Match manage them.
- Android: base64-encoded keystore, keystore password, key alias, key password. Plus a
  Play service-account JSON if you get to Tier 1.

**Versioning:** a human `1.4.0` marketing version plus a monotonically increasing build
number auto-incremented by CI. Both stores reject a build number they've seen before, and
this is the single most common cause of a failed upload.

**Over-the-air updates:** EAS Update (RN) and Shorebird (Flutter) push JS/Dart changes
without a store round-trip. Both stores permit this as long as you're not changing the app's
primary purpose. For a personal app this is a large quality-of-life win — bug fixes reach
your phone in minutes instead of at the next rebuild. Note that OTA can't update native
code, so a new health API or a new native dependency still requires a full build.

---

## 4. Tier 1 — going public

Deferred, but here's the full shape so nothing in Tier 0 blocks it.

### Apple, in order

1. **Bundle ID, App Store Connect record, screenshots** for every required device size.
2. **Privacy manifest** (`PrivacyInfo.xcprivacy`) — required for your app and for bundled
   SDKs. Declares "required reason" API usage (file timestamps, user defaults, disk space,
   etc.) and what data you collect. Third-party SDKs must ship their own; missing manifests
   are a hard upload rejection.
3. **Privacy nutrition labels** — the App Store Connect questionnaire. For this app: Health &
   Fitness data, linked to identity if you have accounts, not used for tracking.
4. **Privacy policy URL** — mandatory, and doubly so with HealthKit. A static page is fine.
5. **HealthKit specifics:**
   - `NSHealthShareUsageDescription` and `NSHealthUpdateUsageDescription` in Info.plist,
     written as real sentences explaining what you do with the data. Vague strings get
     rejected.
   - HealthKit entitlement enabled on the App ID.
   - You may **not** use HealthKit data for advertising or sell it. You may not store it in
     iCloud (Apple's guideline, not just advice).
6. **In-app account deletion** — if the app supports account creation, it must support
   account deletion from inside the app. Non-negotiable, and it shapes your sync backend
   design, so [doc 06](06-data-architecture.md) accounts for it.
7. **App Review.** Expect 24–48h, and expect at least one rejection on the first submission.

### Google Play, in order

1. **$25 one-time** developer registration.
2. ⚠️ **Verify this one carefully — it's the big gotcha.** Personal (non-organization)
   developer accounts created after late 2023 must run a **closed test with 12+ testers for
   14 continuous days** before they can apply for production access. If you plan to publish
   publicly, start that clock early — it's a two-week wall you can't code your way past.
   An organization account (requires a D-U-N-S number) is exempt.
3. **Data Safety form** — Play's equivalent of nutrition labels. Health and fitness data,
   encryption in transit, deletion request mechanism.
4. **Health Connect data-types declaration form** — you must declare each health data type
   you read or write and justify it. Approval is required before a Play-distributed app can
   use Health Connect. Budget days, not hours.
5. **Permissions rationale activity** — Health Connect requires an activity handling
   `ACTION_SHOW_PERMISSIONS_RATIONALE` / the permission-usage intent, linking to your privacy
   policy. Missing it causes a rejection.
6. **Target SDK requirements** — Play enforces a minimum `targetSdkVersion` that ratchets
   upward annually. Existing apps must keep up or get delisted from search.

### Review landmines specific to health, fitness, and diet apps

Worth internalizing now because two of them change how you build:

- **Guideline 1.4.1 — physical harm.** Apps that could encourage dangerous behaviour get
  rejected. For a calorie tracker this means **your algorithm's guardrails are also your
  review defence.** The calorie floor, the max weekly rate limit, and the protein/fat floors
  from [doc 02](02-calorie-trackers-breakdown.md) §3 aren't just good science — an app that
  will happily prescribe 900 kcal/day is a rejection risk and, more importantly, a bad app.
- **Medical claims.** Don't diagnose, don't treat, don't promise outcomes. Ship a plain
  disclaimer: informational and educational, not medical advice.
- **Eating-disorder sensitivity.** Weight-loss apps get extra scrutiny. Avoid shame
  mechanics, avoid unbounded "days under target" streaks, and consider an age gate.
- **Guideline 5.1.3 — health data.** No advertising use, no sale, no iCloud storage of
  HealthKit data, privacy policy required.
- **Guideline 4.2 — minimum functionality.** Not a risk here; the app does a great deal.

### If you ever monetize

Use **RevenueCat**. Native StoreKit 2 and Play Billing are both manageable alone, but
RevenueCat handles receipt validation, cross-platform entitlement, subscription state, and
the genuinely nasty edge cases (refunds, grace periods, billing retry, platform switching)
for free below a meaningful revenue threshold. Not needed for Tier 0 — and worth designing
your entitlement checks behind a one-function interface so adding it later is trivial.

---

## 5. Cost summary

| Item | Tier 0 | Tier 1 |
|---|---|---|
| Apple Developer Program | $99/yr | $99/yr |
| Google Play registration | — | $25 once |
| Backend (Supabase free tier, one user) | $0 | $0–25/mo |
| Food DB hosting (see [doc 07](07-food-database.md)) | $0–5/mo | $5–20/mo |
| CI (EAS / Codemagic free tiers) | $0 | $0–30/mo |
| Crash reporting (Sentry free tier) | $0 | $0–26/mo |
| Domain + privacy policy page | — | ~$12/yr |
| **Total** | **~$100–160/yr** | **~$250–800/yr** |

For comparison: MacroFactor + Strong PRO is roughly **$100–115/year**, forever. Tier 0 is
therefore roughly break-even on cash and enormously negative on time — which is fine, because
you're not building this to save $100. You're building it for the combined analytics that
neither app can give you ([doc 08](08-combined-app-design.md)), and for the fact that your
data stays yours.

---

## 6. Release checklist

Keep this in the repo and actually run it.

**Every release (Tier 0)**
- [ ] Pure-logic test suite green
- [ ] Build number incremented
- [ ] Installed on a real iPhone *and* a real Android device
- [ ] Start a workout, log 3 sets, background the app, confirm the rest timer fires
- [ ] Force-quit mid-workout, reopen, confirm nothing is lost
- [ ] Log a food by barcode; confirm it writes to Apple Health / Health Connect
- [ ] Weigh-in syncs and the trend updates
- [ ] Run a data export; open the file; confirm it's not empty
- [ ] Schema migration tested against a copy of the *real* production database

**Additionally for Tier 1**
- [ ] Privacy manifest present and accurate; SDK manifests present
- [ ] Nutrition labels / Data Safety form match what the app actually collects
- [ ] Privacy policy live and linked from inside the app
- [ ] In-app account deletion works end to end
- [ ] Medical disclaimer visible during onboarding
- [ ] Calorie floor and rate guardrails verified with adversarial inputs
- [ ] Health Connect declaration approved; rationale activity wired up
- [ ] Crash reporting receiving events from a release build

---

## 7. The thing that actually matters for a personal app

Store review, CI, and privacy manifests are all recoverable. **Data loss is not.** Your
training and nutrition history compounds in value every year and is irreplaceable.

Minimum viable production hygiene:

1. **Cloud sync is the primary backup** ([doc 06](06-data-architecture.md)) — but a sync bug
   can propagate a deletion, so it is not sufficient on its own.
2. **Automated periodic export** — a full JSON/SQLite dump to iCloud Drive or Google Drive,
   weekly, unattended. Versioned, not overwritten.
3. **Test the restore path.** An untested backup is a rumour. Restore from an export into a
   clean install at least once, and again after any schema migration.
4. **Never destructively migrate.** Migrations copy forward; the old database file is kept
   until the new one is verified.
5. **Schema migrations get tested against a copy of your real database**, not against a
   fixture. Your real data has ten years of weirdness in it that fixtures don't.

Put export and restore in the MVP, not in "later". It's a day of work that protects
everything else.

---

## Takeaways

1. **Tier 0 is an afternoon.** Apple Developer $99/yr + dev-install or TestFlight on iOS;
   a signed APK, free and non-expiring, on Android. Do it in month one.
2. TestFlight builds expire every 90 days. Calendar reminder at day 80.
3. Automate the build early. Manual releases don't happen.
4. Tier 1 is 1–2 weeks of non-coding work. The two long poles are **Play's 12-testers-for-14-days
   rule** for personal accounts and the **Health Connect declaration** — both are wall-clock
   waits, so start them before you need them.
5. Your algorithm's safety guardrails double as App Review protection. Build them in from
   the start.
6. **Backups and a tested restore path are the only genuinely non-recoverable part of
   production for a personal app.** Ship them in the MVP.
