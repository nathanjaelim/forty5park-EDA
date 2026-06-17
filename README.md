# Project 01 — Geospatial Expansion of the Multifamily Rent AVM

> **Welcome.** This is your first project at Forty5 Park. By the end of it, you'll have a path to owning our production multifamily rent AVM (Automated Valuation Model).
>
> Your eventual scope: **add new geospatial features that make the model smarter, and test it in markets beyond Atlanta** (Dallas, Charlotte, Nashville, etc.). The model that exists today is good — your job is to push it forward.
>
> This document is intentionally **a brief, not a tutorial**. You'll be given the destination, the data, and a roadmap — but the intuition has to develop in your head from looking at the data yourself.

---

## What you're walking into

Forty5 Park acquires and operates multifamily real estate. Decisions like "what rent should we set for unit 4B at our Buckhead building next month?" or "is the seller's pro-forma rent at this Charlotte property realistic?" are made hundreds of times a year. An accurate rent predictor turns gut-feel into a defensible number.

The current state:

- **One MSA in production.** Atlanta-Sandy Springs-Alpharetta (CBSA 12060). Our model trained on Yardi Matrix data covering this metro.
- **Three model variants** for different deployment regimes — repricing owned units, underwriting new acquisitions, comp pricing. (You'll learn what these are by working with the data, not by reading metrics. Resist looking at `artifacts/metrics*.json` until you've formed your own hypotheses.)
- **A solid foundation** but a limited geography. The Atlanta model is tuned to Atlanta-specific submarkets and landmarks. Porting to other MSAs requires curating new geographic anchors and retraining.

Your project is to push on two fronts:

1. **Better geospatial features.** Current geography is good (lat/lon, four landmark distances, H3 cells at two resolutions, submarket / ZIP / county target encoding). What else could matter? POIs, transit access, school catchments, walk scores, neighborhood polygons, density gradients. Hypothesize, build, measure.
2. **Multi-MSA generalization.** Once you understand what makes our Atlanta model work, can you train a Dallas model? A Charlotte model? What changes about the data? About the features? About the assumptions?

By the time you finish, you'll know this model and the data behind it better than anyone else on the team. That's the path to ownership.

---

## The Yardi data

Our primary data vendor is **Yardi Matrix**, a multifamily-specific data provider that aggregates rent, occupancy, transactions, and structural data on tens of thousands of properties across the U.S. We subscribe to four feeds for the Atlanta MSA, dropped monthly.

All four files are in `vignette/01_geospatial_expansion/data/`. They are small (~3.6 MB total) so you can poke at them locally without S3.

### File 1 — `042026-property-enriched-12060.parquet`

**One row per property.** ~2,100 rows × 59 columns. This is the building-level reference table.

| Column family | Examples | What it tells you |
|---|---|---|
| **Identity** | `source_name`, `source_property_id`, `property_name`, `property_id` | Stable IDs across snapshots |
| **Geography** | `latitude`, `longitude`, `street_address`, `city`, `state`, `zipcode`, `county`, `sub_market`, `market`, `census_*_geoid` | The geo dimensions you'll be enriching |
| **Structure** | `num_units`, `year_built`, `min_stories`, `max_stories`, `num_buildings`, `total_parking_spaces`, `lot_size_in_acres` | Physical size of the asset |
| **Quality grades** | `property_quality`, `location_quality` | Yardi's letter-grade scores (A+, A, A-, B+, ...). Read as "Yardi-analyst-rated" |
| **Amenities** | `has_fitness_center`, `has_clubhouse`, `has_business_center`, `has_controlled_access`, `has_garage`, `has_media_room`, `is_elevator_served`, `covered_parking`, `wd_in_unit`, `wd_hookup` | Yes/no flags |
| **BTR signals** | `is_yardi_btr`, `is_hstx_btr` | Build-to-rent typology — single-family-style rentals; behaves differently from apartments |
| **Economics** | `occupancy_rate`, `latest_sale_price_per_sqft`, `latest_sale_price_per_unit`, `latest_sale_price_total`, `latest_sale_date` | Transaction history; many will be null (only ~5-10 years of sale data exists) |
| **Status** | `is_leased_up`, `is_mixed_use` | Operational state |

**Why this matters geographically:** every geo feature in our model derives from columns in this table. Lat/lon → distance features. `sub_market` → target encoding. `zipcode` → ZIP-level aggregates. If you want to add a new geo feature, this table is where its raw inputs live.

### File 2 — `042026-unit-mix-enriched-12060.parquet`

**One row per (property × unit type).** ~6,900 rows × 12 columns. This is the unit-level table — the breakdown of how many 1BR / 2BR / etc. floor plans exist at each property.

| Column | What it tells you |
|---|---|
| `source_property_id` | FK back to property table |
| `unit_type` | The Yardi-canonical unit type code (e.g. `1BR/1.0`, `2BR/2.0`, `3BR/2.0`, `STUDIO`) |
| `unit_configuration` | Free-text floor plan name (less standardized) |
| `has_direct_access_garage` | Boolean |
| `num_units` | How many units of this floor plan exist at this property |
| `living_area_square_feet` | Average square footage for this floor plan |
| `number_of_bedrooms` | 0 = studio, 1, 2, 3, ... |
| `number_of_full_baths` | Numeric (can be fractional: 1.5, 2.5) |
| `rent_close_date` | When this rent observation was sampled |
| `rent_amount` | The asking rent at sample date |
| `haystacks_unit_type` | A finer-grained typology: `APARTMENT`, `TOWNHOUSE`, `ROWHOUSE`, `DETACHED_ENTRY`, `DETACHED_LUXURY`. The last three are BTR-style. |

**Why this matters:** The model is trained at this granularity — one prediction per (property, unit_type). That's the level your geospatial features need to make sense at.

### File 3 — `042026-rent-12060.parquet`

**One row per (property × unit_type) at the training snapshot.** ~6,890 rows × 7 columns. This is the *target* file — the rents we're predicting.

| Column | What it tells you |
|---|---|
| `property_id`, `unit_mix_type`, `unit_garage`, `unit_type` | Composite key |
| `period` | The training snapshot timestamp. All rows are `2026-03-01`. |
| `rent` | The asking rent we're trying to predict |
| `date` | Sample date |

This is the file the loss function is computed against. It joins to `property-enriched` via `property_id` and to `unit-mix-enriched` via `source_property_id` + `unit_type`.

### File 4 — `042026-hist-rent-12060-12060.parquet`

**The historical rent panel.** ~708,800 rows × 7 columns. Monthly asking rent per (property × unit_type) going back 11 years (~2015 to 2026-03).

| Column | What it tells you |
|---|---|
| `property_id`, `unit_mix_type`, `unit_garage`, `unit_type` | Composite key |
| `period` | Month of observation (the first of each month) |
| `rent` | Asking rent that month |
| `date` | Sample date for this observation |

**This panel is one of the most important files in the codebase.** Understanding what kinds of features you can derive from it — and what features would *leak* the target — is a major part of becoming an effective AVM engineer here. We'll come back to this in the EDA section.

### Bonus — `yardi_sales_atl.parquet`

A small sales-comparison table (~600 rows). Useful as a reference but not currently in the production feature set. You may or may not find it interesting.

### Reference — `atlanta_landmarks.{csv,json}`

11 curated Atlanta landmark coordinates (Buckhead, Midtown, Downtown, Hartsfield airport, Ponce City Market, Battery Atlanta, GA Tech, ...). These power the `dist_<landmark>_km` features. **Read this file early.** When you start proposing new geographic features, you'll be deciding whether to add more landmarks here.

---

## The investigation: how to get up to speed

Treat this as an investigation, not a checklist. The goal of each phase is **intuition**, not output.

### Phase 0 (½ day) — Get the environment running

```bash
# From the repo root
pip install -e ".[dev]"
prime-mfr list
pytest
```

If all three commands succeed, you have a working install. Read the top-level `README.md` once — particularly the **architecture flowchart** and the **YAML configs section**. You don't need to internalize it yet; you'll come back to it.

Open this folder (`vignette/01_geospatial_expansion/`) in your editor and look at the four parquets. Don't load them yet — just notice they're there.

### Phase 1 (week 1) — Yardi data fluency

The goal is to **build mental models for what Yardi actually contains**, with no model-related lookups. Pretend the AVM doesn't exist. You're an analyst trying to understand a dataset.

#### Day 1-2: Schema and shape

For each of the four parquets:

1. **Shape and dtypes.** `df.shape`, `df.dtypes`, `df.info()`. How many rows? How many columns? What types?
2. **Nullness.** `df.isna().sum().sort_values(ascending=False)`. Which columns are sparsely populated? Are some columns 90%+ null? (Those tell you something: either Yardi doesn't track them well for Atlanta, or they're forward-looking fields not yet populated.)
3. **Cardinality of categoricals.** `df.select_dtypes('object').nunique()`. How many distinct submarkets? ZIP codes? unit types? Property names?
4. **Distinct values for low-cardinality columns.** `df['sub_market'].value_counts()` — which submarkets are most represented? `df['property_quality'].value_counts()` — what's the grade distribution?

**Notebook tip:** Open a Jupyter notebook in `vignette/01_geospatial_expansion/notebooks/`. Use `pandas`, `pyarrow`, `matplotlib`, `seaborn`. Keep it scrappy — you're exploring, not publishing.

#### Day 3: The target — `rent`

This is the variable the model exists to predict. You need to know it cold.

1. **Distribution.** Plot `rent`. Histogram. KDE. Box-plot. Is it normal? Skewed? Bimodal?
2. **Log distribution.** Plot `np.log1p(rent)`. Does it look more symmetric? (This is one of those hints: think about what that implies for modeling.)
3. **By unit type.** `df.groupby('unit_type')['rent'].describe()`. Studios vs 1BR vs 2BR vs 3BR. How big are the gaps?
4. **By submarket.** Box-plot of rent by `sub_market`. Order submarkets by median rent on the x-axis. Where's Buckhead? Where's South Atlanta?
5. **By square footage.** Scatter of `rent` vs `sqft` (you'll need to compute sqft from unit-mix-enriched first). Where's the variance? Is there a clear linear trend or is rent-per-sqft (PSF) a better unit?
6. **PSF analysis.** Compute `psf = rent / sqft`. Plot the distribution. Compare to `rent`. Which is more normally distributed? (Hint: this question matters for the target transformation choice. Don't peek at how we resolved it.)

**Hypothesis to test yourself:** Rent and sqft are dominantly linearly related. Is that true? Where does the relationship break down?

#### Day 4: Geography

You're going to live in this part of the data.

1. **Coordinate sanity.** Plot `(longitude, latitude)` on a scatter. All points should be in the Atlanta MSA bounding box. Are there outliers? (Sometimes a property has a typo'd lat/lon.)
2. **Geographic distribution.** Color the scatter by `rent`. What patterns do you see? Are there obvious high-rent clusters? Are they where you'd expect (Buckhead, Midtown)?
3. **Submarket boundaries.** Color the scatter by `sub_market`. Are submarkets spatially coherent or scattered?
4. **Distance from a reference.** Pick one anchor point (e.g. downtown Atlanta at `(-84.388, 33.749)`). Compute Euclidean distance from every property. Plot rent vs distance. Is there a gradient?
5. **Open `atlanta_landmarks.json`.** Read the curated landmark list. For each one, compute distance to every property and plot rent vs that distance. Which landmark distance shows the strongest rent gradient? **Write down your top three guesses before you compute, then check.**
6. **H3 cells (optional this week).** Install `h3-py`. Compute H3 res-6 and res-8 cells for each property. Look at rent variance within vs between cells. What does this tell you about the right "neighborhood" granularity?

**Open question for yourself:** If our current model uses four landmark distances (Buckhead, Midtown, Downtown, Atlanta Airport), what was the team thinking? Could you propose a fifth landmark? A sixth? What criteria would you use?

#### Day 5: Property attributes and quality

1. **Vintage distribution.** Plot `year_built` distribution. When was the Atlanta MFR boom? Is there a "1990s gap"?
2. **Vintage × rent.** Box-plot rent by decade of construction. Where does the premium / discount sit?
3. **Quality grades.** Yardi assigns `property_quality` (A+ through D-). Plot rent by grade. How much premium does an A property command vs B+?
4. **Amenities.** For each amenity flag (`has_fitness_center`, `has_clubhouse`, ...), compute mean rent with vs. without. Which amenities matter most? Be skeptical: some of this is correlation with property quality, not causation.
5. **Unit count tier.** Group properties into bins by `num_units` (e.g. <100, 100-200, 200-400, 400+). Does size matter for rent? Why might bigger properties have different rent levels?

#### Day 6 — The historical rent panel

This is the heaviest single piece of data you've got. It's also the most important to internalize.

1. **Shape and time coverage.** What's the date range? How many distinct properties have ≥24 months of history? How many have ≥10 years?
2. **Per-property rent trajectories.** Pick 5 random `property_id`s. For each, plot `rent` over `period` as a time series. What do they look like? Smooth? Spiky? Strict monotone increases? Are there gaps?
3. **Autocorrelation.** For one property and one unit_type, plot `rent[t]` vs `rent[t-1]`. How correlated? `rent[t]` vs `rent[t-12]` (a year ago)? Compute the Pearson correlation for both.
4. **Within-MSA trends.** Group all properties by `period` and plot the median asking rent over time. What's the long-run trend? Are there structural breaks (COVID, 2022 rate hikes)?
5. **The leakage question.** Look at the very last month of the panel — what's `period`? Compare it to the `period` in the target file (`042026-rent-12060.parquet`). What do you notice? *This is the most important question in the entire onboarding.* (Hint: if you use last-month rent as a "lag" feature, you need to be sure you're not just reading the target.)

**Hypothesis to test:** Most of the variance in next-month rent is explained by *this-month* rent. How strong is that effect? What does this imply for what the model could possibly learn from purely-structural features?

### Phase 2 (week 2) — Identify the geospatial gaps

By the end of week 1 you should have intuitions like:
- "Rent in Atlanta is dominated by submarket and proximity to a handful of high-end nodes."
- "The dataset has obvious geographic gradients but also significant within-submarket variance that pure lat/lon can't explain."
- "Two properties on opposite sides of Peachtree might be in the same submarket but command different rents — what explains the difference?"

In week 2, **propose** new geospatial features that might capture what current features miss. Don't implement yet — write down hypotheses and how you'd test them.

Examples of the kind of feature you might propose (without saying these are right or wrong):
- **Walk score / transit score.** Properties near MARTA stations may carry a transit premium. Sources: Walk Score API, OSM stations + Haversine.
- **POI density.** Count of restaurants/coffee shops/grocery within 0.5mi (OSM Overpass query).
- **School catchment quality.** Top-tier public school districts (Atlanta Public Schools, Decatur City, Fulton/Cobb) may command family-rental premia.
- **Highway proximity.** Distance to I-285, GA-400 — convenient at 1 mi, noisy at 0.1 mi.
- **Green-space proximity.** Distance to Piedmont Park, Beltline trail.
- **Block-level density.** Properties in dense urban blocks vs detached neighborhoods.
- **Polygonal regions.** Inside the Beltline / outside the Beltline binary.
- **Neighborhood "vibe" clusters.** Auto-discovered via embedding lat/lon + amenity profile.

For each proposed feature, write:

| Hypothesis | Data source | Cost to build | How you'd validate |
|---|---|---|---|
| ... | ... | ... | ... |

The point is to develop a **portfolio of hypotheses** before building anything. The team will pick the 2-3 most promising with you.

Read [`docs/proposal_geographic_expansion.md`](../../docs/proposal_geographic_expansion.md) at this point — it'll tell you how the codebase is structured to make adding geo features cheap. Don't read it before week 2; it'll bias your hypothesis-generation.

### Phase 3 (weeks 3-4) — Implement and measure

By now you have hypotheses, you know the codebase, and you've read the architecture proposal. Pick one or two features, implement them in `src/prime_mfr/features/geographic.py`, register them via the YAML config system, and train a comparison variant.

- Create a new YAML: `configs/models/primary_<your_feature>.yaml`
- Add the feature to the right group in `configs/features/numeric.yaml`
- Run `prime-mfr train --model primary_<your_feature>`
- Compare metrics against the baseline

**Only at this point should you read** `docs/results.md` — the technical results writeup. By then you'll have your own numbers to compare against, and you'll understand the framing rather than just accepting it.

### Phase 4 (week 5+) — A new MSA

Once you've added at least one geographic feature that moves the needle, port the model to a second MSA — Dallas-Fort Worth or Charlotte are sensible candidates (similar growth markets, Yardi has good coverage).

This phase is unknown territory and we'll plan it together. Expect:
- Re-curating landmark coordinates for the new MSA
- Refitting submarket target encodings (the values will differ)
- Validating that the model architecture generalizes (it should — but verify)
- Comparing against a simple baseline (median rent by ZIP code)

### What ownership looks like

When you finish this vignette, you should be able to:

- Explain what each Yardi feed contains, what it's missing, and what its sample rate is
- Argue for or against any individual feature in the production model
- Add a new geographic feature without supervision
- Train a new model variant for a new market
- Be the person other team members ask when they have a question about the AVM

That's ownership.

---

## EDA starter — load everything in 10 lines

Save this as `vignette/01_geospatial_expansion/notebooks/00_load_data.py` and run it as a sanity check before you start the real exploration:

```python
from pathlib import Path
import pandas as pd

DATA = Path(__file__).resolve().parent.parent / "data"

property_df = pd.read_parquet(DATA / "042026-property-enriched-12060.parquet")
unit_mix_df = pd.read_parquet(DATA / "042026-unit-mix-enriched-12060.parquet")
rent_df = pd.read_parquet(DATA / "042026-rent-12060.parquet")
hist_df = pd.read_parquet(DATA / "042026-hist-rent-12060-12060.parquet")

for name, df in (
    ("property-enriched", property_df),
    ("unit-mix-enriched", unit_mix_df),
    ("rent", rent_df),
    ("hist-rent", hist_df),
):
    print(f"{name:>20s}: {df.shape}")

print("\nProperty submarket distribution:")
print(property_df["sub_market"].value_counts().head(10))

print("\nRent distribution at training snapshot:")
print(rent_df["rent"].describe())
```

---

## What you should not do

A few traps to avoid.

1. **Do not read `artifacts/metrics*.json`, `docs/results.md`, or `docs/case_study_avm.md` in week 1.** They will short-circuit your intuition-building. The whole point of phase 1 is to develop your own hypotheses about what should matter, then verify or revise them with data — not adopt ours. Read these documents in week 2 or later, after you've done the EDA.
2. **Do not implement features before you can articulate why they should work.** "Walk score might help" is a vibe. "Properties in walkable submarkets command rent premia that distance-to-CBD doesn't capture; here's how I'd test it on the data" is a testable hypothesis. Aim for the second.
3. **Do not over-rely on plots in week 1.** Compute summary statistics, look at numbers, sort properties by attributes. The plots come from understanding what to plot — not the other way around.
4. **Do not skip the historical rent panel.** It is by far the most data and (in our experience) by far the most consequential. If you can't explain in one sentence why the panel matters and where the leakage risks are by end of week 1, you haven't gone deep enough.

---

## How to ask for help

When you get stuck — and you will — frame your question like:

> I'm trying to [GOAL]. I've already [WHAT YOU TRIED]. My current hypothesis is [HYPOTHESIS]. What I don't understand is [SPECIFIC GAP].

Not:

> The data doesn't make sense.

The first form gets you a useful answer in one round; the second gets you 20 minutes of triage. Both are fine for the first week. By week 3 we'll expect the first form.

Welcome aboard.
