# Predicting the Big West Track & Field Championships from Season Bests

DSC 180 (Quarter 1) project. We test whether a simple **season-best performance list**, the method used by meet calculators such as MeetScorer, can predict Big West Conference Outdoor Track & Field Championship results well enough to help **value recruits**.

For every championship from 2019 to 2026 (none in 2020), we:

1. Take each athlete's best **outdoor** regular-season mark in every Big West event, using only marks from before the championship.
2. Rank the marks from best to worst within each event.
3. Give places 1–8 the championship points (10, 8, 6, 5, 4, 3, 2, 1).
4. Add up predicted points by team and compare them with the actual championship results.

## Key results

Averaged over 14 championships (7 seasons × men/women). Source: `notebooks/out/summary.csv`.

| Metric | Entrants field | Everyone field | Last year's totals (baseline) |
|---|---|---|---|
| Team total MAE (points) | 10.3 | 11.4 | 26.2 |
| Spearman (team order) | 0.92 | 0.92 | 0.79 |
| Winner correct | 10 / 14 | 10 / 14 | 6 / 12 |
| Team-event MAE | 2.04 | 2.04 | n/a |
| Scorer precision | 0.79 | 0.74 | n/a |
| Scorer recall | 0.78 | 0.74 | n/a |
| Exact place | 0.28 | 0.29 | n/a |
| Place within one | 0.63 | 0.62 | n/a |

- **Entrants field:** only athletes who actually competed in that event at that year's championship. This is the best case for the method, because it uses information not known before the meet.
- **Everyone field:** every athlete with an outdoor mark in the event from a school at that year's championship. This is the list a coach could build ahead of time.

**Findings**

- Team totals are predicted to within about 10–11 points, against an average team total of about 85. That is less than half the error of using last year's totals, and the season-best list does better in every season.
- Individual places are weak. Only about 28% of predicted scorers finish exactly where predicted, and almost half of athletes predicted to finish 8th don't score.
- Season bests are optimistic. The median entrant finishes about 1% below their outdoor season best, and only about 30% beat it. The gap is largest in throws, vertical jumps and distance events.
- Each year, entrants with no outdoor mark in their event score 31–103 points that the method cannot predict.
- The results are robust. Removing wind-aided marks or adding indoor marks changes team-total error by less than half a point.

**For recruiting:** the method is reliable enough to estimate a recruit's contribution at the team level. A recruit's points in a single event should be read as approximate, especially near 8th place.

## Repository structure

```
.
├── data/
│   ├── raw/
│   │   ├── championship_raw.csv          # scraped Big West Championship results (2019–2026)
│   │   └── regular_season_raw.csv        # scraped regular-season marks for Big West athletes
│   └── processed/
│       ├── bigwest_2019_2026_results.csv # cleaned championship results (one final place per athlete-event)
│       ├── bigwest_roster.csv            # athlete_id, year, school, gender
│       ├── regular_season.csv            # cleaned regular-season marks (2019+, Jan–May, pre-championship)
│       └── meet_venues.csv               # hand-labeled indoor/outdoor venue per meet
└── notebooks/
    ├── data_scraping/
    │   ├── big_west_scraping.ipynb       # cleans championship results -> bigwest_2019_2026_results.csv
    │   └── athete_profiles.ipynb         # cleans regular-season marks -> regular_season.csv
    ├── season_best_replication.ipynb     # main analysis: season-best model + evaluation
    ├── out/                              # CSV outputs of the main analysis
    └── figures/                          # PNG figures of the main analysis
```

### Data dictionary (main processed files)

**`bigwest_2019_2026_results.csv`**: one row per athlete per event per championship.
`athlete_id, athlete_name, team, meet_name, meet_year, gender, event, mark, mark_unit, place, wind, event_type, points`

**`regular_season.csv`**: one row per regular-season performance.
`athlete_id, athlete_name, team, meet_name, meet_date, year, event, mark, mark_unit, place, is_track, wind, event_type, original_team, meet_start_date, gender`

**`meet_venues.csv`**: `meet_name, meet_date, venue (indoor/outdoor), note`. The main notebook stops with an error if any regular-season meet is missing from this file.

## Pipeline

### 1. Championship results: `notebooks/data_scraping/big_west_scraping.ipynb`

- Standardizes team names, for example "Cal State Northridge" and "Cal St. Northridge" become "CSUN".
- Collapses prelims and finals into **one final place per athlete-event**:
  - Finalists keep their finals place.
  - Top-8 prelim qualifiers with no finals result are flagged as suspected DNS and get no place.
  - Non-qualifiers are placed 9th and below.
- Writes `data/processed/bigwest_2019_2026_results.csv`.

### 2. Regular-season marks: `notebooks/data_scraping/athete_profiles.ipynb`

- Parses marks into a single unit per event: seconds for running events, meters for field events, points for multis. Wind readings and status codes (DNS/DNF/DQ/…) are extracted.
- Replaces each athlete's team with their **Big West school for that year**, using the roster and then the championship results. This handles transfers.
- Keeps 2019 and later, January–May meets that start on or before that year's championship end date, and excludes Big West Championship meets (the Big West Challenge Cup is kept).
- Attaches gender from the roster or championship file.
- Flags outlier marks that are far from the athlete's own median or the event median, then drops a hand-verified list of bad marks (placeholder values, wrong event labels, impossible times).
- Writes `data/processed/regular_season.csv`.

### 3. Season-best model and evaluation: `notebooks/season_best_replication.ipynb`

1. **Cleaning:** maps event names to codes, unifies athlete IDs and team names, treats each relay as one "athlete" per team, maps status codes, and puts every mark on a performance scale `p` (`log(mark)` for field and multi events, `-log(time)` for running) so that higher is always better.
2. **Seasons and indoor/outdoor:** multi-day meets use their last day. August–December meets count toward the next spring. Indoor/outdoor labels come from `meet_venues.csv`.
3. **EDA:** data overview, coverage of championship entrants, field size, season best vs. championship mark, and scoring standards.
4. **Model:** builds the season-best lists, ranks them, and scores the top 8 for the entrants and everyone fields.
5. **Evaluation:** team total MAE, Spearman, winner accuracy, team-event MAE, scorer precision and recall, and place accuracy, compared against the last-year's-totals baseline.
6. **Robustness:** legal-wind-only and indoor-included variants, plus a UC San Diego case study.

**Outputs** written to `notebooks/out/` and `notebooks/figures/`:

| File | Contents |
|---|---|
| `sb_athletes_{entrants,everyone}.csv` | Per-athlete predicted vs. actual place and points |
| `sb_team_totals_{entrants,everyone}.csv` | Per-team predicted vs. actual totals and ranks |
| `sb_metrics_{entrants,everyone}.csv` | Metrics per season × gender |
| `summary.csv` | Summary table (above) |
| `scoring_standards.csv` | Median mark for 1st, 3rd and 8th per event |
| `fig_sb_vs_champ.png` | Championship mark vs. season best by event group |
| `fig_team_totals.png` | Predicted vs. actual team totals |
| `fig_mae_by_season.png` | Team-total MAE by season vs. baseline |
| `fig_place_confusion.png` | Predicted vs. actual place |
| `fig_recall_by_group.png` | Scorer recall by event group |

## How to replicate

### Requirements

- Python 3.10+ (the notebooks were last run on Python 3.14)
- `pandas`, `numpy`, `scipy`, `matplotlib`, `jupyter`

```bash
git clone https://github.com/ChidiebereAgunanne/DSC180_Quarter-1.git
cd DSC180_Quarter-1
python -m venv .venv && source .venv/bin/activate
pip install pandas numpy scipy matplotlib jupyter
```

### Reproduce the main results (recommended)

The processed data is already committed, so the analysis can be rerun directly:

```bash
cd notebooks
jupyter notebook season_best_replication.ipynb   # Run All
```

Run it from inside `notebooks/`, because paths are relative (`../data/processed/...`). It regenerates everything in `notebooks/out/` and `notebooks/figures/`.

### Rebuild the processed data (optional)

1. Run `notebooks/data_scraping/big_west_scraping.ipynb` to produce `bigwest_2019_2026_results.csv`.
2. Run `notebooks/data_scraping/athete_profiles.ipynb` to produce `regular_season.csv`. This depends on step 1 and on `bigwest_roster.csv`.
3. Rerun `season_best_replication.ipynb`.

> **Note:** see *Known issues* below. The scraping notebooks currently need some path and file fixes before they run end-to-end.

## Known issues / TODO

- **Missing raw input names:** `big_west_scraping.ipynb` reads `data/raw/bigwest_2021_2026_audit.csv`, and `athete_profiles.ipynb` reads `data/raw/bigwest_regular_season_partial.csv`. Neither file is in the repo. They probably correspond to `championship_raw.csv` and `regular_season_raw.csv`.
- **Missing module:** `athete_profiles.ipynb` imports `bigwest_model` (`bm.parse_mark`), but only a compiled `notebooks/__pycache__/bigwest_model.cpython-314.pyc` is committed, not the `.py` source. `parse_mark` is also defined in `season_best_replication.ipynb`.
- **Relative paths:** the notebooks in `notebooks/data_scraping/` use `../data/...`, which resolves to `notebooks/data/`. It should be `../../data/...`.
- **Typo:** `season_best_replication.ipynb` reads `"../data./processed/meet_venues.csv"`, which should be `"../data/processed/meet_venues.csv"`.
- **No scraping code:** the scraping notebooks only clean data. The code that originally scraped results from the web is not included.
- **No requirements file:** consider adding `requirements.txt`, and a `.gitignore` for `.DS_Store` and `__pycache__/`.
