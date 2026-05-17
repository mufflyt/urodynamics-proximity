# Urodynamics-Proximity

Code and data for mapping geographic proximity to urodynamics testing centers across the United States.

Urodynamic testing (CPT codes 51726, 51727, 51728, 51729, 51741) is the diagnostic standard for evaluating lower urinary tract dysfunction — urinary incontinence, voiding dysfunction, and neurogenic bladder. This analysis examines geographic disparities in access to urodynamics testing using Medicare provider data, geocoded testing-center locations, 2021 TIGER/Line census tract polygons, and 2023 ACS 5-year female population estimates.

> **Writing the paper?** See [`ONBOARDING.md`](./ONBOARDING.md) for a manuscript-oriented walkthrough (background, methods, results, file map, journal targets).

---

## What this analysis produces

| Output | Description |
|---|---|
| `Boxplot.png` | Distribution of total Medicare beneficiaries per testing center (log scale) |
| `N per state.png` | Choropleth of raw center counts per state with geocoded points overlaid |
| `Figure 1A.png` | Choropleth of centers per million women (state-level density) |
| `Figure 1B.png` | Census-tract-level choropleth of distance to the nearest urodynamics center, with inset panels for AK / HI / PR |
| Console summary | National and per-state population-weighted distance quantiles; metro vs. non-metro split; states with zero centers |

---

## The five CPT codes

| CPT | Description |
|---|---|
| 51726 | Complex cystometrogram (CMG) |
| 51727 | CMG with voiding pressure studies |
| 51728 | Complex CMG with voiding pressure studies and urethral pressure profile (UPP) |
| 51729 | Complex CMG with voiding pressure studies, UPP, and EMG |
| 51741 | Complex uroflowmetry (electronic equipment) |

A clinician was counted if they billed ≥1 of these codes in CY2022. A *location* is a unique geocoded street address.

---

## Pipeline overview

1. Load the CMS Medicare Physician & Other Practitioners PUF, filtered to the five urodynamics CPTs.
2. Standardize street-address strings and coalesce records to unique locations.
3. Geocode locations via ArcGIS (`tidygeocoder`); cache results in `geocoded_urodynamics_locations.csv`.
4. Pull state- and tract-level female population from the 2023 ACS 5-year (`B01001_026`) via `tidycensus`.
5. Compute centers per million women by state.
6. Classify each center as metropolitan vs. non-metropolitan by intersecting its point with 2021 TIGER/Line CBSA polygons (LSAD `M1`).
7. For every 2021 census tract centroid, compute distance to the nearest center using `sf::st_nearest_feature()` in EPSG:5070 (Albers equal-area, meters → miles).
8. Population-weight the distance distribution with `Hmisc::wtd.quantile()`.
9. Render `Boxplot.png`, `N per state.png`, `Figure 1A.png`, and `Figure 1B.png`.

The full pipeline is in [`run_analysis.R`](./run_analysis.R) (≈400 lines, runs end-to-end). [`Urodynamics Geography Quarto.qmd`](./Urodynamics%20Geography%20Quarto.qmd) is the annotated Quarto version.

---

## Reproducing the analysis

**Requirements**
- R ≥ 4.2
- R packages: `dplyr`, `data.table`, `janitor`, `stringr`, `Hmisc`, `tidygeocoder`, `tidycensus`, `ggplot2`, `ggpubr`, `ggthemes`, `ggbeeswarm`, `geosphere`, `sf`, `usmap`, `patchwork`
- A free Census API key — register at https://api.census.gov/data/key_signup.html, then in R: `tidycensus::census_api_key("YOUR_KEY", install = TRUE)`
- Internet connection on first run (ArcGIS geocoder + Census API). All results cached afterward.

**Run**
```bash
cd urodynamics-proximity
Rscript run_analysis.R
```

First run: 30–60 min (geocoding + ACS pulls). Cached reruns: 2–5 min. Set `URODYNAMICS_REFRESH_GEOCODES=true` to force re-geocoding.

---

## Source data already in the repo

| File / folder | Source |
|---|---|
| `Medicare_Urodynamics_Data.csv` | CMS Medicare Physician & Other Practitioners PUF (CY2022), filtered to HCPCS 51726/51727/51728/51729/51741: https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service/data |
| `tlgdb_2021_a_us_substategeo.gdb/` | 2021 TIGER/Line sub-state geography geodatabase (Census Tract layer): https://www2.census.gov/geo/tiger/TGRGDB21/ |
| `RU Data/tl_2021_us_cbsa.*` | 2021 TIGER/Line Core-Based Statistical Areas: https://catalog.data.gov/dataset/tiger-line-shapefile-2021-nation-u-s-core-based-statistical-areas |
| `ACS 2023 5-year Female Tract Population.csv` | Cache of `tidycensus::get_acs(variable = "B01001_026", year = 2023, survey = "acs5", geography = "tract")` |
| `US Census Tract 2020 Data.csv`, `US Census Tract 2020 Female Data.csv` | Legacy Decennial 2020 P1 files; **not used** by the current pipeline (kept for archival reference) |
| `geocoded_urodynamics_locations.csv` | Cache of ArcGIS geocodes for the ~2,600 unique testing-center addresses |

---

## File map

```
urodynamics-proximity/
├── README.md
├── ONBOARDING.md                                   Manuscript-writing guide
├── run_analysis.R                                  Reproducible end-to-end pipeline
├── Urodynamics Geography Quarto.qmd                Annotated Quarto notebook (alt format)
├── Medicare_Urodynamics_Data.csv                   Source: CMS PUF
├── geocoded_urodynamics_locations.csv              Cache: ArcGIS geocodes
├── ACS 2023 5-year Female Tract Population.csv     Cache: tract-level female pop (ACS 5-yr 2023)
├── US Census Tract 2020 Data.csv                   Legacy (not used in current pipeline)
├── US Census Tract 2020 Female Data.csv            Legacy (not used in current pipeline)
├── tlgdb_2021_a_us_substategeo.gdb/                Census tract polygons
├── RU Data/
│   └── tl_2021_us_cbsa.*                           Metro/Micro CBSA shapefile
├── Boxplot.png                                     Output
├── N per state.png                                 Output
├── Figure 1A.png                                   Output (headline state-level figure)
└── Figure 1B.png                                   Output (headline tract-level figure)
```

---

## Methodology notes worth knowing

- **Coordinate systems.** Raw geocodes are WGS84. The CBSA intersection runs in NAD83 (TIGER's native CRS). The distance calculation runs in EPSG:5070 Albers equal-area (units in meters) so nearest-neighbor distance is metric-accurate; the result is converted to miles using `miles_per_meter = 0.000621371`.
- **Distance algorithm.** Current code uses `sf::st_nearest_feature()` + `sf::st_distance()` for memory efficiency at ~85K tracts × ~2.6K centers. The original Quarto used the full `geosphere::distm` Haversine matrix; both give comparable results.
- **Population weighting.** All headline distance statistics are population-weighted using `Hmisc::wtd.quantile()` with female tract population as the weight. Unweighted summaries would overrepresent low-population rural tracts.
- **Metro classification.** A center is "metropolitan" if its geocoded point falls within a CBSA whose LSAD is `M1` (Metropolitan Statistical Area). Micropolitan (`M2`) areas are explicitly excluded.
- **Why female-only denominator.** Pelvic floor disorders and urinary incontinence disproportionately affect women, and urogynecology is the primary access pipeline. The denominator is trivial to swap — change `female_population_var` at the top of `run_analysis.R`.

---

## Project

- **Lead:** Tyler Muffly, MD — Department of Obstetrics & Gynecology, Denver Health (tyler.muffly@dhha.org)
- **Manuscript lead:** Chetan Giduturi — chetan.giduturi@cuanschutz.edu
- **Status:** Analysis pipeline is complete and reproducible; manuscript drafting in progress (see `ONBOARDING.md`).
