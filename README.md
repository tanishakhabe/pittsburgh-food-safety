# pittsburgh-food-safety
Data Engineering Project Group 7 (95-815, Fall 2026)

End-to-end pipeline for Allegheny County restaurant inspection data, linked to
City of Pittsburgh neighborhoods. **The whole pipeline is in one notebook:
[`food_safety_pipeline.ipynb`](food_safety_pipeline.ipynb).**

## Data sources

| Dataset | Provider | Format | Rows | Stored in |
|---|---|---|---|---|
| [Food facility inspections 2014–2025](https://data.wprdc.org/dataset/allegheny-county-restaurant-food-facility-inspection-violations/resource/4ea730d8-2bf9-4783-b0a5-b41ed687097e) | Allegheny County Health Dept (WPRDC) | CSV | 124,674 | Postgres `raw.inspections_2014_2025` |
| [Inspection violations 2014–2025](https://data.wprdc.org/dataset/allegheny-county-restaurant-food-facility-inspection-violations/resource/1a1329e2-418c-4bd3-af2c-cc334e7559af) | Allegheny County Health Dept (WPRDC) | CSV | 405,474 | Postgres `raw.violations_2014_2025` |
| [Geocoded food facilities](https://data.wprdc.org/dataset/allegheny-county-restaurant-food-facility-inspection-violations/resource/112a3821-334d-4f3f-ab40-4de1220b1a0a) | Allegheny County Health Dept (WPRDC) | CSV | 32,245 | Postgres `raw.geocoded_facilities` |
| [Food facility inspections 2025–present](https://data.wprdc.org/dataset/allegheny-county-restaurant-food-facility-inspection-violations/resource/1c9cf8d8-f3db-4364-b925-d7078c2fb02f) | Allegheny County Health Dept (WPRDC) | CSV | 9,501 | Postgres `raw.inspections_2025_present` |
| [Neighborhood boundaries](https://data.wprdc.org/dataset/neighborhoods2/resource/4af8e160-57e9-4ebf-a501-76ca1b42fc99) | City of Pittsburgh (WPRDC) | GeoJSON | 90 features | MongoDB `group7.neighborhoods_geojson` |

How they join:

- `inspections.encounter = violations.encounter`: one inspection, many violations
- `inspections.id = violations.id = geocoded_facilities.id`: the facility
- facility `x`/`y` point inside a neighborhood polygon (PostGIS `ST_Contains`): the neighborhood

## Where things live

| What | Where |
|---|---|
| Pipeline code | `food_safety_pipeline.ipynb` in this repo |
| Tabular data | AWS RDS PostgreSQL, a single instance with these schemas: `raw` → `staging` → `serving`, plus `meta` |
| GeoJSON | MongoDB Atlas, database `group7` |
| Credentials | `.env` on your own machine (git-ignored) |
| Downloaded files | `data/` on your own machine (git-ignored; the violations CSV is too big for GitHub) |

Postgres schemas:

- `raw`: source files exactly as downloaded. All columns are `TEXT` with no constraints. **Never edit by hand.**
- `staging`: typed, cleaned, de-duplicated tables with primary keys, foreign keys and checks.
- `serving`: analysis-ready tables (Milestone 3).
- `meta`: `load_log` (every raw load), `dq_log` (every data-quality check and the action taken), and safe-cast helper functions.

## Setup (once per person)

```bash
git clone https://github.com/tanishakhabe/pittsburgh-food-safety.git
cd pittsburgh-food-safety
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env               # then fill in the host, username and password
```

Open `food_safety_pipeline.ipynb` in VS Code or Jupyter, choose the `.venv` kernel, and **Run All**.

## Re-running

The notebook can be run any number of times:

- `raw` tables are dropped and reloaded from the files, so you never get duplicate rows.
- `staging` tables are dropped and rebuilt from `raw`, so cleaning is never applied twice.
- MongoDB documents are upserted by id.
- Only `meta.load_log` and `meta.dq_log` grow. Each run is tagged with a `run_id`.

Because each run rebuilds shared tables, **post in the group chat before running it**, so two people don't run it at the same time.

## Working together

1. `git pull` before you start.
2. Work on a branch, e.g. `git checkout -b clean-violations`.
3. Notebooks merge badly. Agree who is editing which section, and **clear all outputs before committing** (VS Code: "Clear All Outputs") unless you're committing a final run on purpose.
4. Push and open a pull request.

## Known source quirks

- WPRDC CSV dumps end the header line with CRLF but data rows with LF. The loader reads the header itself so `COPY` doesn't reject the file.
- The legacy (2014–Jul 2025) and new (Aug 2025+) inspection systems use different IDs. The new-system file can only be linked by facility name and address.
