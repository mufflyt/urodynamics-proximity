# Onboarding: Geographic Distribution of Urodynamics Testing Centers in the United States

**Audience:** Chetan Giduturi (chetan.giduturi@cuanschutz.edu)
**Lead:** Tyler Muffly, MD — Department of Obstetrics & Gynecology, Denver Health
**Repo:** https://github.com/(owner)/urodynamics-proximity (local: `/Users/tylermuffly/urodynamics-proximity`)
**Status:** Analysis pipeline is complete and reproducible. Figures and summary statistics are generated. Manuscript writing has not yet begun — that is your task.
**Your role:** Take the existing abstract + analysis output and develop a full manuscript suitable for submission (target journal TBD; see "Target journals" below).

---

## 1. The 30-second pitch

Urodynamic testing (UDS) is the gold-standard diagnostic for lower urinary tract dysfunction — urinary incontinence, voiding dysfunction, and neurogenic bladder. Despite these conditions affecting tens of millions of US women, **access to urodynamics testing is geographically uneven, with several states having no Medicare-billing UDS testing centers at all and millions of women living more than 100 miles from the nearest center.** This paper quantifies that disparity using 2022 Medicare provider data, geocoded testing-center locations, and census-tract-level female population data, then visualizes it as a national choropleth.

The work parallels (and re-uses code from) a previously-published cardiac PET proximity analysis. This is the urodynamics adaptation.

---

## 2. The abstract (current draft you've already read)

> **Background.** Urodynamic testing (CPT 51726, 51727, 51728, 51729, 51741) is the diagnostic standard for lower urinary tract dysfunction in women, yet little is known about its geographic availability in the United States.
>
> **Objective.** To map the location of Medicare-billing urodynamics testing centers and characterize the distance women in the US must travel to reach one.
>
> **Methods.** We extracted all clinicians who billed Medicare Part B in 2022 for the five urodynamics CPT codes from the CMS Medicare Physician & Other Practitioners Public Use File. Provider street addresses were geocoded via ArcGIS. We computed the great-circle distance from the centroid of every US census tract (2021 TIGER/Line) to the nearest urodynamics center, and weighted distances by the 2023 ACS 5-year female population estimate (variable B01001_026) at the tract level. We classified centers as metropolitan if their geocoded point fell inside a 2021 Core-Based Statistical Area (CBSA) of type "M1" (Metropolitan Statistical Area). State-level density was computed per million women.
>
> **Results.** [Pull final numbers from `run_analysis.R` output — current draft values are summarized in §6 below.]
>
> **Conclusions.** Access to urodynamic testing is geographically concentrated in metropolitan areas. A substantial population of US women — disproportionately in rural states and the Mountain West — must travel long distances to reach a Medicare-billing UDS center, with implications for delayed diagnosis of pelvic floor disorders.

When you write the full paper, the abstract above is the *starting* point — feel free to revise wording, but lock the numbers from the latest `run_analysis.R` output before submission.

---

## 3. Background you'll need for the introduction

**Clinical motivation.** Urinary incontinence affects roughly 50% of community-dwelling women at some point in their lives; overactive bladder, voiding dysfunction, and neurogenic bladder add to that burden. For complex cases — failed empiric therapy, mixed symptoms, planning for incontinence surgery, neurologic disease — urodynamics is the diagnostic test of choice. AUA, SUFU, and ACOG/AUGS all have guidelines on when urodynamics should be performed.

**Workforce/access background.** Urodynamics is performed by urologists, urogynecologists (Female Pelvic Medicine & Reconstructive Surgery / Urogynecology and Reconstructive Pelvic Surgery), and to a lesser extent by general gynecologists and trained nurse practitioners. Equipment is capital-intensive (~$30–60K), which discourages adoption outside larger practices. Reimbursement for the CPT codes has declined over the past decade.

**Geographic-access literature to anchor against.** Cite parallel proximity analyses in: cardiac PET (the methodological parent of this paper), bariatric surgery, comprehensive stroke centers, abortion services, and gynecologic oncology. The methodological hook is the same: provider-level Medicare data → geocode → census-tract distance → weighted population statistics.

**Why Medicare data.** It is the most complete public-use dataset of practicing US clinicians who bill for a given CPT. Limitations: misses Medicaid-only, VA, and cash-pay providers; misses providers below the CMS suppression threshold (≤10 beneficiaries per HCPCS in a year). Address those in the Discussion.

---

## 4. The five urodynamics CPT codes (Methods section)

| CPT | Description |
|---|---|
| 51726 | Complex cystometrogram (CMG) |
| 51727 | CMG with voiding pressure studies |
| 51728 | Complex CMG with voiding pressure studies and urethral pressure profile (UPP) |
| 51729 | Complex CMG with voiding pressure studies, UPP, and EMG |
| 51741 | Complex uroflowmetry (electronic equipment) |

A clinician was counted as a "urodynamics testing provider" if they billed ≥1 of these codes in CY2022. A *location* was defined as a unique geocoded street address (a single clinician with multiple offices contributes to each).

---

## 5. Methodology in detail (read this before writing Methods)

The full pipeline is in `run_analysis.R` (≈400 lines, runs end-to-end). The Quarto notebook `Urodynamics Geography Quarto.qmd` is a more annotated version of the same logic that can be rendered to HTML. Steps:

1. **Load** `Medicare_Urodynamics_Data.csv` — pre-filtered CMS Physician & Other Practitioners PUF (CY2022), restricted to the five CPT codes.
2. **Clean addresses.** Standardize abbreviations (Ave → Avenue, Blvd → Boulevard, etc.), strip suite numbers and punctuation. See lines 105–134 of `run_analysis.R`.
3. **Coalesce by address.** Multiple providers/codes at the same street address collapse to one *location*. Sum `tot_benes`, `tot_srvcs`; average `avg_mdcr_pymt_amt`. Done twice — once on raw addresses, again after the cleaning step — to maximize de-duplication.
4. **Geocode** each unique address through ArcGIS via the `tidygeocoder` R package. Results cached in `geocoded_urodynamics_locations.csv` so reruns are free.
5. **State-level counts and density.**
   - Count of centers per state.
   - Female population per state from the 2023 ACS 5-year (variable `B01001_026`) via `tidycensus::get_acs()`.
   - Density = centers per million women.
6. **Metro vs. non-metro classification.** Intersect geocoded points with 2021 TIGER/Line CBSA polygons (`RU Data/tl_2021_us_cbsa.shp`), filtered to `lsad == "M1"` (Metropolitan Statistical Areas, not Micropolitan).
7. **Census-tract distance.**
   - Load 2021 TIGER/Line census tract polygons from `tlgdb_2021_a_us_substategeo.gdb` (layer "Census_Tract").
   - For each tract centroid, compute distance to the nearest urodynamics center using `sf::st_nearest_feature()` + `sf::st_distance()` in EPSG:5070 (Albers equal-area, meters) — the modernized version. (The original Quarto used `geosphere::distHaversine` on a full N×M matrix; both give comparable results and we kept the new approach because the Haversine matrix gets memory-heavy at ~85K tracts × ~2.6K centers.)
   - Convert meters → miles using `0.000621371`.
8. **Pull tract-level female population** (ACS 5-year 2023, variable `B01001_026`) for every state + DC + PR, cached in `ACS 2023 5-year Female Tract Population.csv`. Merge on `GEOID`.
9. **Weighted statistics.** Use `Hmisc::wtd.quantile()` with `pop` as the weight to compute population-weighted median (and IQR) distance to the nearest center, nationally and by state.
10. **Figures.**
    - `Boxplot.png` — distribution of total beneficiaries per center (log scale).
    - `N per state.png` — choropleth of raw center count per state with geocoded points overlaid.
    - `Figure 1A.png` — choropleth of *centers per million women* (density) — the paper's headline state-level figure.
    - `Figure 1B.png` — choropleth at the **census-tract** level, colored by distance-to-nearest-center in 8 bins (<5, 5–10, 10–25, 25–50, 50–100, 100–200, 200–500, ≥500 mi). Inset panels for Alaska, Hawaii, and Puerto Rico.

**Coordinate systems.** WGS84 for raw geocodes → NAD83 for the CBSA intersection (the TIGER CBSA file uses NAD83) → EPSG:5070 Albers for distance math (equal-area, units in meters). If you describe this in Methods, that's the level of detail readers/reviewers will expect.

**Why female population only.** Urodynamics is performed in both sexes, but the paper is framed around women because (a) prevalence of urinary incontinence and pelvic floor disorders is markedly higher in women, (b) urogynecology workforce is the most relevant access pipeline, and (c) the parent cardiac PET paper used the analogous "adult population" denominator. If you want to add a sensitivity analysis using total adult or female + male populations, the code is trivial to adapt — change `female_population_var` at the top of `run_analysis.R`.

---

## 6. Headline results (preliminary — re-pull final values from the most recent run)

From the most recent run of `run_analysis.R`:

- **~2,600 unique urodynamics testing centers** (Medicare-billing) nationwide.
- **States with zero centers:** check the "States with 0 urodynamics centers" output block — historically includes very small states/territories.
- **Highest density (centers per million women):** typically dense Northeast and Mid-Atlantic states.
- **Lowest density:** Mountain West and Great Plains states (Wyoming, Montana, the Dakotas, Alaska).
- **Population-weighted median distance for women** to the nearest center: report the 25th/50th/75th percentile triple from `wtd.quantile(dist, pop, probs = c(0.25, 0.5, 0.75))`.
- **Women > 100 miles from a UDS center:** report N (in millions) and percentage.
- **Metropolitan Statistical Areas containing a UDS center:** report the count and percentage of all M1 CBSAs.

**Action item for you:** before submission, re-run `Rscript run_analysis.R` once with the latest cached data and paste the console "Summary Statistics" block directly into the Results section.

---

## 7. Repo file map

```
urodynamics-proximity/
├── README.md                                       Reproduction-focused readme
├── ONBOARDING.md                                   This file
├── run_analysis.R                                  One-shot script that produces every figure + stat
├── Urodynamics Geography Quarto.qmd                Annotated Quarto version of the analysis
├── Medicare_Urodynamics_Data.csv                   Source: CMS Medicare PUF, CY2022, filtered to 5 CPTs
├── geocoded_urodynamics_locations.csv              ArcGIS geocoding cache (~2.6K rows)
├── US Census Tract 2020 Data.csv                   Decennial 2020 P1_001N — legacy, not used in current pipeline
├── US Census Tract 2020 Female Data.csv            Decennial 2020 female pop — legacy
├── ACS 2023 5-year Female Tract Population.csv     ACS 2023 5-yr B01001_026 by tract — the *active* denominator
├── tlgdb_2021_a_us_substategeo.gdb/                2021 TIGER/Line census tract polygons (geodatabase)
├── RU Data/
│   └── tl_2021_us_cbsa.*                           2021 TIGER/Line CBSA shapefile (Metro/Micro areas)
├── Boxplot.png                                     Output: beneficiary distribution
├── N per state.png                                 Output: raw count map
├── Figure 1A.png                                   Output: density-per-million-women map
└── Figure 1B.png                                   Output: census-tract distance map (the headline figure)
```

The `.gdb/` is a folder, not a single file — don't try to open it as a binary. R reads it via `sf::st_read(dsn = "tlgdb_…", layer = "Census_Tract")`.

---

## 8. Reproducing the analysis

**Requirements:**
- R ≥ 4.2
- Packages: `dplyr`, `data.table`, `janitor`, `stringr`, `Hmisc`, `tidygeocoder`, `tidycensus`, `ggplot2`, `ggpubr`, `ggthemes`, `ggbeeswarm`, `geosphere`, `sf`, `usmap`, `patchwork`
- A Census API key for `tidycensus` (free — register at https://api.census.gov/data/key_signup.html, then `tidycensus::census_api_key("YOUR_KEY", install = TRUE)`)
- Internet connection (ArcGIS geocoder + Census API are called on first run; everything is cached afterward)

**Run:**
```bash
cd urodynamics-proximity
Rscript run_analysis.R
```

The first run takes 30–60 minutes (mostly geocoding + ACS pulls). Subsequent runs hit the local caches and finish in 2–5 minutes. To force re-geocoding, set `URODYNAMICS_REFRESH_GEOCODES=true`.

**Source data that must be re-downloaded if you start from scratch** (already in the repo):
1. CMS Medicare PUF (filter to HCPCS 51726/51727/51728/51729/51741): https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service/data
2. TIGER/Line 2021 census tract geodatabase: https://www2.census.gov/geo/tiger/TGRGDB21/
3. TIGER/Line 2021 CBSA shapefile: https://catalog.data.gov/dataset/tiger-line-shapefile-2021-nation-u-s-core-based-statistical-areas

---

## 9. Suggested manuscript structure

**Title** (working): "Geographic Disparities in Access to Urodynamic Testing for Women in the United States: A National Medicare-Based Analysis"

**Sections:**

1. **Abstract** — see §2 above; structured (Background / Objectives / Methods / Results / Conclusions).
2. **Introduction** (~500–700 words) — burden of LUTS in women, the role of UDS, prior workforce/access literature, and the gap this paper fills. End with a one-sentence objective.
3. **Methods** (~700–900 words) —
   - Data sources (CMS PUF, TIGER/Line, ACS).
   - Inclusion criteria (the 5 CPT codes, CY2022).
   - Geocoding & deduplication.
   - Metro classification (CBSA M1).
   - Distance computation (nearest-neighbor, EPSG:5070, miles).
   - Population weighting (ACS B01001_026, 2023 5-yr).
   - Statistical analysis (descriptive, weighted quantiles — no inferential testing is needed for the headline analysis).
   - IRB statement (public, de-identified data → not human subjects research).
4. **Results** (~600–900 words + 1 figure) —
   - Count and characteristics of centers (table?).
   - State-level density (Figure 1A).
   - Distance distribution (Figure 1B + numeric summary).
   - Metro vs. non-metro split.
   - States with zero centers.
5. **Discussion** (~800–1100 words) —
   - Primary finding restated.
   - Comparison to cardiac PET / other proximity analyses.
   - Policy implications (telehealth pre-visit triage, traveling clinics, FPMRS workforce, Medicare reimbursement).
   - **Limitations:** Medicare-only (misses Medicaid/cash/VA); 10-beneficiary suppression threshold; CPT-based identification will miss centers that bill incorrectly or under "other" codes; ArcGIS geocoding accuracy; tract centroid as patient-origin proxy is coarse; no travel-time (only great-circle distance); no insurance/cost barriers captured.
   - Future directions.
6. **Conclusion** — 2–3 sentences.
7. **References** — start with the parent cardiac PET paper and the geographic-access literature listed in §3.
8. **Figures** —
   - **Figure 1A** — state-level density choropleth (centers per million women).
   - **Figure 1B** — tract-level distance choropleth (the strongest single visual).
   - Boxplot of beneficiaries per center is optional supplement.

**Tables (suggested):**
- Table 1: characteristics of the 2,600 centers (state count, metro %, specialty mix if you want — the `Rndrng_Prvdr_Type` field in the raw CSV gives Urology vs. Ob/Gyn vs. other).
- Table 2: distance metrics by state (or region), population-weighted median and % > 100 mi.

---

## 10. Target journals (suggestion order)

1. *Urology* or *The Journal of Urology* — direct topical fit.
2. *Female Pelvic Medicine & Reconstructive Surgery* / *Urogynecology* (the AUGS journal) — best audience fit.
3. *Obstetrics & Gynecology ("Green Journal")* — broader gyn audience.
4. *American Journal of Obstetrics & Gynecology* — if you want a higher-impact swing.
5. *Health Affairs* / *JAMA Network Open* — if you reframe as a health-access piece.

---

## 11. Open questions / things to decide before submission

1. **CY year of the Medicare data** — `Medicare_Urodynamics_Data.csv` should be confirmed as CY2022 (latest fully-released PUF as of writing). If CMS has released CY2023 since the analysis was run, consider re-running.
2. **Specialty stratification** — do we want to break down centers by `Rndrng_Prvdr_Type` (Urology vs. Ob/Gyn vs. FPMRS vs. other)? The data supports it.
3. **Beneficiary-volume threshold** — should we exclude very-low-volume centers (e.g., ≤5 cases/yr)? Current analysis includes everyone, which is probably right but worth a sensitivity analysis.
4. **Travel-time vs. distance** — great-circle distance is conservative but ignores roads/terrain. Mountain West states look worse with travel time. Probably a v2 paper, not v1.
5. **Dual-sex denominator** — currently female-only. Could add a panel for both sexes; flagged in §5 above.
6. **IRB language** — confirm Denver Health and CU Anschutz IRB language for "non-human subjects research using public, de-identified data."

---

## 12. Contact

- **Tyler Muffly, MD** — tyler.muffly@dhha.org — PI, Denver Health Ob/Gyn
- **Chetan Giduturi** — chetan.giduturi@cuanschutz.edu — 916-765-7261 — manuscript lead

Ping Tyler with any methodological questions; if you change anything in `run_analysis.R`, please open a branch rather than committing straight to `main`.
