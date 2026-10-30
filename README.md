# Stats Under the Stars 9 — Forecasting Operational Cases on the ANAS Road Network

Solution developed for the **Stats Under the Stars 9 (SUS9)** hackathon. The task was to
forecast the number of monthly operational cases (requests, reports and operational events)
for each road of the Italian **ANAS** national network across the test period, using
historical event-level records from 2016–2025.

Submissions were ranked by **Root Mean Squared Logarithmic Error (RMSLE)**, which rewards
accurate forecasts while limiting the influence of very high-volume roads.

## Key idea

We considered two substantially different formulations of the task:

1. **Direct monthly forecasting** — predict every future road–month count directly from
   history, temporal variables, road identifiers and metadata. The model has to estimate
   *both* the annual volume of a road and its distribution across the twelve months.

2. **Annual allocation** — after discovering that the metadata contained annual case counts
   (disaggregated by incident type) for 2024 and 2025, the annual total for each road no
   longer needed to be forecast. The task collapses to estimating **monthly proportions**
   `p_{r,y,m}` (summing to 1 per road–year) and multiplying them by the known annual total.

The second formulation was decisive: it removes all uncertainty about the annual magnitude
and lets the model focus only on the temporal distribution of a known total.

## Results

| Approach                               | Local RMSLE (2023) | Public RMSLE |
| -------------------------------------- | :----------------: | :----------: |
| Direct monthly forecasting             |       0.712        |    0.69483   |
| Annual-calibrated CatBoost             |         –          |    0.485     |
| CatBoost with incident-type composition|       0.497        |    0.48319   |
| Historical incident-type seasonality   |       0.499        |      –       |
| **Final 75/25 allocation ensemble**    |     **0.496**      |  **0.47927** |

The final ensemble reduced the public RMSLE from **0.69483** to **0.47927**, an improvement
of roughly **31%**. The gain came from *reformulating the statistical problem*, not from
swapping one algorithm for another.

## Repository structure

```
sus9-anas-forecasting/
├── README.md
├── LICENSE
├── requirements.txt
├── .gitignore
├── notebooks/
│   └── three_catboost_submissions.ipynb   # reproduces the 3 allocation submissions
└── report/
    └── report.pdf                          # full write-up of the methodology
```

## Method summary

The pipeline treats three types of road differently, depending on how much history is
available for each test road:

- **Roads with a direct historical match** — road-level historical statistics (mean, median,
  max, observed months) are used as features.
- **Unseen roads with a known road family** — family-level statistics are used, where the
  *family* is extracted from the initial component of the road code (e.g. `SS 106 RACC.` →
  `SS106`).
- **Unseen roads with no history at all** — predictions rely on calendar effects, global
  statistics, target-year incident-type composition and physical road metadata.

Two allocation models are combined:

- **Model B — CatBoost with incident-type composition.** A gradient-boosted regressor
  predicts a non-negative monthly score; the twelve scores are normalised within each
  road–year to obtain allocation shares. Missing numeric metadata is left missing and handled
  natively by CatBoost.
- **Model C — historical seasonality by incident type.** For each incident type, a monthly
  profile is learned from 2016–2022 (with add-one smoothing) and applied to the known annual
  composition of each road–year.

The two are blended as `0.75·B + 0.25·C` (weight chosen on the 2023 validation year), the
shares are renormalised to sum to 1, and continuous predictions are converted to non-negative
integers with the **largest-remainder method** so that every road–year sums *exactly* to its
known annual total.

## Data (not included)

The competition dataset (event-level ANAS records, road metadata and accident data) is
**not redistributed** in this repository — it belongs to the hackathon organisers and is not
ours to publish. To reproduce the results you need the original files:

- `train.csv`, `test.csv` — event-level history and the test grid
- `road_metadata.csv` — road metadata **including** the `2024_tipologia_*` and
  `2025_tipologia_*` annual incident-type columns
- `incidenti.xlsx` — accident data (optional enrichment)

## Team

Denise Brancaleone · Matías Cerda · Gianluca Pasquinelli · Marco Restuccia
