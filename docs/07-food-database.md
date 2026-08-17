# 07 — The Food Database

**This is the largest hidden cost in any calorie tracker.** The logging UI is a week. The
algorithm is a week. The food database is a rolling commitment: acquisition, licensing,
normalization, deduplication, search relevance, portion parsing, and quality control, forever.

MyFitnessPal's moat isn't its app. It's 20 million food entries. MacroFactor's advantage
isn't size — it's that someone curates theirs.

**Decision: Open Food Facts + USDA FoodData Central, self-hosted.** No API fees, no
licensing that blocks personal use. This doc covers how, plus one refinement that lets you
defer most of the infrastructure.

---

## 1. The sources

### USDA FoodData Central — the quality tier

US Government work, **public domain**. Bulk downloads as CSV or JSON, plus a REST API.
Four datasets that matter:

| Dataset | Size | What it is | Use it for |
|---|---|---|---|
| **Foundation Foods** | ~hundreds | Lab-analysed single ingredients, full nutrient panels, provenance | Gold standard whole foods |
| **SR Legacy** | ~7,800 | The old Standard Reference. Broad, reliable, well-portioned | The backbone of a whole-foods DB |
| **FNDDS (Survey)** | ~15,000 | Foods as *eaten* — "chicken, roasted, skin not eaten" | Prepared and mixed dishes |
| **Branded Foods** | ~1.9M | Manufacturer label data via GS1 | Packaged goods, US-centric |

Foundation + SR Legacy + FNDDS ≈ **23,000 high-quality foods** covering essentially
everything you cook. That is a startlingly small, high-value dataset — a few tens of MB
with nutrients, portions, and a search index. Small enough to **ship inside the app**.

Nutrient IDs you'll want: `1008` energy (kcal), `1003` protein, `1004` total fat,
`1005` carbohydrate by difference, `1079` fibre, `2000` sugars, `1093` sodium,
`1258` saturated fat, `1253` cholesterol, `1092` potassium.

### Open Food Facts — the coverage tier

~3–4 million packaged products worldwide, crowd-sourced, barcode-indexed. This is what makes
scanning work.

- **Licence: ODbL** for the database, **DbCL** for individual contents, **CC-BY-SA** for
  images. ⚠️ Verify current terms.
- **Personal use: no obligations in practice.**
- **If you publish (Tier 1 in [doc 04](04-productionization-and-deployment.md)):** ODbL
  requires **attribution** and is **share-alike** — a derived database that you distribute
  must be offered under ODbL too. For a client app shipping a filtered OFF subset, that
  means crediting Open Food Facts visibly and making your derived extract available. This is
  entirely livable, but decide it deliberately rather than discovering it at submission.
- **Quality is uneven.** Crowd-sourced means missing nutrients, wrong serving sizes,
  duplicate barcodes, and occasional nonsense. §6 is about defending against that.
- Access: bulk JSONL/CSV/Parquet exports (with daily deltas), a MongoDB dump, or the public
  REST API.

**Cronometer's source (NCCDB)** is better than both but is commercially licensed. Out of scope.

---

## 2. The architecture — three tiers

The insight that makes this tractable:

> **Your personal food universe is about 200 items, and about 30 in any given month.**

You don't need 4 million foods. You need *your* foods, instantly, offline — plus the ability
to look up anything else when you're standing in a supermarket.

```
┌─────────────────────────────────────────────────────────────┐
│ TIER 1 — Bundled with the app.  ~23k foods, ~20-30 MB       │
│ USDA Foundation + SR Legacy + FNDDS. FTS5 indexed.          │
│ Always available, always offline, always high quality.      │
├─────────────────────────────────────────────────────────────┤
│ TIER 2 — Remote barcode & brand lookup.                     │
│ Open Food Facts. Hit on scan / brand search only.           │
│ Needs network. Fails gracefully to manual entry.            │
├─────────────────────────────────────────────────────────────┤
│ TIER 3 — Local cache + your own foods.  Grows to ~200-500.  │
│ EVERY food you scan or log is copied here permanently.      │
│ After ~a month, this serves the overwhelming majority of    │
│ your logging, entirely offline.                             │
└─────────────────────────────────────────────────────────────┘
```

Tier 3 is what makes the whole thing feel fast. A cache hit rate that starts at 0% and is
above 90% within six weeks means Tier 2's latency and availability barely matter.

### A refinement worth taking: defer the self-hosting

You chose self-hosted, and that's the right end state. But for **v1, Tier 2 can just be the
public Open Food Facts API**:

```
GET https://world.openfoodfacts.org/api/v2/product/{barcode}.json
    ?fields=code,product_name,brands,nutriments,serving_size,serving_quantity,quantity
```

Send a descriptive `User-Agent` identifying your app (they ask for this, and they enforce
it), respect the rate limits, and cache every response forever in Tier 3. For one person
scanning a handful of barcodes a week, this is comfortably inside acceptable use and costs
**zero infrastructure**.

Build the self-hosted mirror (§3) when one of these becomes true:

- You want offline barcode scanning (a real gym/supermarket-basement problem).
- You want brand *search*, not just barcode lookup — the public API's search is slow and
  the good text search really does want to be local.
- You get rate-limited, or you're publishing and want to control your own availability.

**Recommendation: ship Tier 1 + public API + Tier 3 cache in the MVP. Build the mirror in
Phase 4b, once you know from real use whether you need it.** This is a deferral, not a
change of direction — the ingestion pipeline below is the same either way.

---

## 3. Ingestion pipeline

Runs offline, on your laptop, producing artifacts. Not app code — a directory of scripts.

```
  download          normalize          filter           index            emit
     │                  │                 │               │                │
 USDA CSV/JSON ──▶ per-100g canon ──▶ quality gate ──▶ FTS5 build ──▶ core.sqlite  (bundled)
 OFF JSONL     ──▶ per-100g canon ──▶ country/qual ──▶ pg indexes ──▶ Postgres     (mirror)
```

**Step 1 — Download.** USDA bulk CSVs per dataset. OFF JSONL or Parquet, plus daily deltas
so refreshes are incremental rather than full re-downloads.

**Step 2 — Normalize to one canonical shape.** Everything becomes **per 100 g**:

- USDA nutrients arrive already per 100 g. Map nutrient IDs to your column names.
- OFF fields are `energy-kcal_100g`, `proteins_100g`, `fat_100g`, `carbohydrates_100g`,
  `fiber_100g`, `sugars_100g`, `salt_100g`.
- **Salt vs sodium**: OFF often gives salt. `sodium_g = salt_g / 2.5`. Getting this backwards
  is a factor-of-2.5 error in a nutrient people actually watch.
- **Energy in kJ vs kcal**: OFF has both. `kcal = kJ / 4.184`. Prefer the declared kcal; fall
  back to converting kJ; flag the entry if they disagree by more than a few percent.

**Step 3 — Filter.** This is what makes the data manageable:

- Drop OFF products with no energy value — a large fraction of the corpus, and useless.
- Filter by `countries_tags` to the one or two markets you shop in. Global → UK-and-US alone
  cuts the corpus by roughly an order of magnitude.
- Drop products failing the quality gate in §6.
- For the bundled Tier 1 file, keep only USDA and only the ~12 core nutrients (push the rest
  into a sparse `micros_json` column).

**Step 4 — Deduplicate.** Same barcode appearing multiple times, and the same product under
several barcodes. Prefer the entry with more complete nutrition, a more recent edit, and a
higher OFF `completeness` score. Keep the loser's barcode as an alias rather than discarding
it — that's a cheap row that prevents a failed scan later.

**Step 5 — Index and emit.** Build FTS5 over the bundled file; build Postgres indexes on the
mirror. Version the output (`core-2026-08.sqlite`) so the app knows whether to replace its
bundled copy on update.

**Refresh cadence.** USDA updates a few times a year. OFF changes daily but you don't care —
your Tier 3 cache is the copy that matters. Regenerate the bundled file when you ship an
app update; refresh the mirror monthly if you build one.

---

## 4. Barcodes — the details that break scanners

```
UPC-A    12 digits   North America
EAN-13   13 digits   International.  A UPC-A is an EAN-13 with a leading zero.
EAN-8     8 digits   Small packages
ITF-14   14 digits   Shipping cases — not retail units
```

**The leading-zero bug.** Your scanner returns a 12-digit UPC-A. OFF stores barcodes mostly
normalized to 13 digits. Look up `"012345678905"` and you get nothing; look up
`"0012345678905"` and you find it. **Always try both forms** — and normalize on the way in,
so your local cache doesn't accumulate both.

```
candidates(scanned) = {
    scanned,
    scanned.padStart(13, '0'),          # UPC-A → EAN-13
    scanned.replace(/^0+/, ''),         # EAN-13 → UPC-A
    ean8_to_ean13(scanned) if len == 8
}
```

**Check digits.** Both formats use a mod-10 weighted checksum (alternating ×3 / ×1). Validate
before querying — a misread is common with a scratched label at an odd angle, and a checksum
failure is a much better UX ("try again") than a false "product not found".

**In-store / variable-measure barcodes.** Prefixes `02` and `20`–`29` are reserved for
retailer-assigned codes — deli counters, weighed produce, butcher packs. They are **only
meaningful inside that one store chain** and often encode the price rather than the product.
Detect them and skip the lookup entirely; go straight to manual entry or a search. Otherwise
you'll get bewildering false matches.

**When a scan misses**, which will happen constantly for local and store-brand products:
offer, in this order — (1) search by name, (2) create a custom food *pre-filled from the
barcode* so the next scan hits your cache, (3) quick-add macros. Never dead-end. The
scan-miss flow is used more than you'd expect and is worth designing properly.

---

## 5. Search

Search is the fallback path, but it's the one people judge the app on.

**FTS5, with a real ranking function:**

```sql
CREATE VIRTUAL TABLE food_fts USING fts5(
  name, brand,
  content='food', content_rowid='rowid',
  tokenize='porter unicode61 remove_diacritics 2'
);
```

**Ranking = BM25 plus priors that matter more than BM25 does:**

```
score = bm25(food_fts)
      − 8.0 × is_personal_food          -- you created it: almost always the answer
      − 6.0 × log(1 + times_logged)     -- you've eaten it before
      − 3.0 × is_favourite
      − 2.0 × source_quality            -- USDA Foundation > SR > FNDDS > OFF-complete > OFF-sparse
      − 1.0 × has_complete_nutrition
      + 2.0 × is_generic_when_brand_queried
```

(FTS5's `bm25()` returns lower-is-better, hence the subtractions.)

**The single biggest relevance win is personal history.** If you've logged "Chobani Plain
0%" forty times, it belongs at the top of a search for "cho" regardless of what BM25 thinks.
Weight personal signals far above textual relevance — this is the difference between search
that feels psychic and search that feels like a database.

**Typo tolerance.** FTS5's `trigram` tokenizer (SQLite 3.34+) handles fuzzy matching, or use
`spellfix1`. Worth having; the average logging query is typed one-handed.

**As-you-type** wants prefix queries (`chick*`) and a debounce around 150 ms. Query Tier 1
and Tier 3 locally and instantly; fire Tier 2 in parallel and merge results when they land,
never blocking the local ones.

---

## 6. Data quality — defending against crowd-sourced data

Cheap automated checks that catch most bad entries. Run them in the pipeline **and** at
log time on anything from Tier 2.

**The Atwater cross-check — the single most effective one:**

```
computed_kcal = 4×protein_g + 4×carb_g + 9×fat_g
if |declared_kcal − computed_kcal| / max(declared_kcal, 1) > 0.20:
    flag as suspect
```

Catches unit errors, decimal-place slips, and per-serving values mistakenly entered as
per-100 g. Allow more slack when fibre or sugar alcohols are significant (their effective
factors are lower than 4).

**Bounds, per 100 g:**

```
0 ≤ kcal ≤ 900          (pure fat is 900; anything above is impossible)
0 ≤ protein, fat, carb ≤ 100
protein + fat + carb ≤ 105        (rounding slack; higher means someone entered nonsense)
fibre ≤ carb
sat_fat ≤ fat
sugar ≤ carb
```

**Completeness score**, so ranking can prefer better data:

```
quality = 0.40 × has_all_four_macros
        + 0.20 × passes_atwater
        + 0.15 × has_serving_size
        + 0.15 × source_tier            (USDA Foundation 1.0 → OFF-sparse 0.2)
        + 0.10 × has_micronutrients
```

**Show provenance in the UI.** A small "USDA" / "Open Food Facts" / "Yours" badge on each
search result, the way Cronometer does. It costs nothing, it takes ten minutes to build, and
it lets you make an informed choice between two entries that differ by 30 calories.

**Let yourself correct things.** Any food can be edited into a personal override that shadows
the source entry and is preferred forever after. Over a year you'll accumulate corrected
versions of the 50 foods you eat constantly, and your database quietly becomes better than
anyone's.

---

## 7. Portions — the genuinely hard part

Nutrients are per 100 g. Humans do not eat in grams. Bridging that is where most of the
friction in food logging actually lives.

**USDA** provides a proper `food_portion` table: `amount`, `measure_unit`, `modifier`,
`gram_weight` — "1 cup, chopped = 128 g". Good data; use it directly.

**OFF** provides `serving_size` as free text: `"30g"`, `"1 cup (240 ml)"`,
`"2 biscuits (25 g)"`, `"1/4 pizza"`. Parse with a regex cascade, and fall back to OFF's
computed numeric `serving_quantity` when parsing fails. Accept that some fraction is
unparseable and just offer grams there.

**Volume → mass needs density.** `1 cup of X` is meaningless without `density_g_ml`. Defaults:
water 1.0, milk 1.03, oil 0.92, honey 1.42, flour ~0.53. Store density where known; where
unknown, don't offer volume units at all rather than guessing.

**The feature that beats all of the above: learned personal portions.**

```sql
-- per food, what you actually log
SELECT food_id, grams, COUNT(*) AS n
FROM food_log_entry
WHERE food_id = ? AND deleted_at IS NULL
GROUP BY food_id, grams
ORDER BY n DESC, MAX(logged_at) DESC
LIMIT 1;
```

Pre-fill the quantity field with that. If you always log 180 g of chicken breast, the field
says 180 g before you touch it, and logging is one tap. This single behaviour does more for
logging speed than any amount of portion-data curation — and it's about twenty lines of code.

**Also worth having:** ×0.5 / ×2 quick multipliers, a "last time you had 180 g" hint, and
recipe scaling (cook 6 servings, log 1).

---

## 8. Recipes and meals

Two distinct things people confuse:

- **Recipe** — ingredients plus a servings count; nutrients derived per serving. Weight loss
  during cooking matters: 500 g of raw chicken becomes ~375 g cooked, so support an optional
  **cooked yield weight** and scale accordingly. Skipping this makes every home-cooked meal
  systematically wrong.
- **Meal** — a saved *combination* of already-defined foods logged as a unit. "My breakfast"
  = 60 g oats + 200 g yoghurt + 100 g blueberries. No derivation, just a bundle.

Both are high-value for personal use because personal eating is extremely repetitive. **Copy
yesterday's breakfast** is one tap replacing six, and is probably the highest
value-per-line-of-code feature in the entire nutrition side of the app.

---

## 9. Deliberately out of scope for v1

- **Natural-language logging** ("two eggs and a slice of toast"). Delightful, and a v2
  feature. It's an LLM call plus a matching layer over the database you already have — which
  is exactly why it can wait until the database exists.
- **Photo/meal recognition.** Impressive, unreliable, and it needs the portion problem solved
  to be worth anything.
- **Restaurant menu data.** The commercial APIs' real advantage. Handle it with quick-add
  macros and move on.
- **Full micronutrient tracking against RDAs.** The schema carries `micros_json` so the data
  is there when you want it; the UI can wait.

---

## Takeaways

1. **Ship USDA Foundation + SR Legacy + FNDDS bundled in the app** — ~23k excellent foods,
   tens of MB, always offline. This alone covers most home cooking.
2. **Cache every food you ever touch, forever.** Your personal universe is ~200 items;
   within six weeks the cache serves almost everything.
3. **Use the public OFF API for barcodes in v1.** Build the self-hosted mirror in Phase 4b,
   when offline scanning or brand search proves it's needed. Same pipeline either way.
4. **Handle the UPC-A/EAN-13 leading zero**, validate check digits, and detect in-store
   `02`/`2x` prefixes. These three things are most of what makes a scanner feel reliable.
5. **Rank search by personal history far above text relevance.**
6. **Snapshot nutrients onto the log entry** ([doc 06](06-data-architecture.md) Trap 2) — the
   source data is mutable and your history must not be.
7. **Learned personal portions** are twenty lines of code and the biggest single win in
   logging speed.
8. **The Atwater cross-check** catches most bad crowd-sourced data for almost no effort.
9. ODbL means attribution and share-alike **if you publish**. Irrelevant for personal use;
   decide it deliberately before Tier 1.
