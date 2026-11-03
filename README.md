# Seattle Transit Frequency Dashboard

A work-in-progress tool for showing how often King County Metro (and later Sound Transit and other Puget Sound agencies) routes run at different times of day. It uses the public GTFS schedule feed. The end goal is an interactive map dashboard where you can filter routes by frequency, such as "every route running every 15 minutes or better on weekday evenings."

> **Status:** early development. The ETL and frequency calculations work inside Jupyter notebooks. The mapping is a prototype and the Streamlit dashboard is a stub. See [Roadmap](#roadmap).

## What it does today

1. **Downloads** the consolidated Puget Sound GTFS feed from [OneBusAway](https://gtfs.sound.obaweb.org/prod/gtfs_puget_sound_consolidated.zip).
2. **Cleans and merges** `trips`, `stop_times`, `stops`, and `routes`, then filters to King County Metro (`agency_id == 1`).
3. **Classifies service IDs** by day type (`weekday`, `saturday`, `sunday`, `special`) using `calendar.txt`.
4. **Calculates headways** (the gap between departures) per route, grouped by route, service ID, direction, and stop. Results are saved for each day type and time window.
5. **Maps route shapes** with Folium on CARTO basemap tiles, including a test map of only the RapidRide lines.

## Repository layout

```
├── Notebooks/
│   ├── kcmetro_etl.ipynb     # Download, clean, merge GTFS; compute frequencies
│   ├── mapping.ipynb         # Build GeoJSON from shapes; Folium maps
│   └── streamlit_test.py     # Dashboard placeholder
├── data/
│   ├── KCM_data/             # King County Metro GTFS text files
│   ├── pgc_gtfs/             # Consolidated Puget Sound GTFS (downloaded; git-ignored)
│   ├── freq_data/            # Frequency outputs: freq_<day>_<start>_<end>.csv
│   ├── serv_dow.csv          # service_id → day type lookup
│   ├── route_shape_ids.csv   # route_id → most common shape_id
│   ├── shapes.geojson        # Route shapes as LineStrings
│   ├── geometry.csv          # Route geometries (WKT)
│   ├── kcm_service_area.html # Folium map of all KCM shapes
│   └── rr_routes.html        # Folium map of RapidRide lines only
├── Project-Outline.pdf/.docx # Project goals and planned work
└── README.md
```

## Getting started

### Requirements

Python 3.13 (the version the project was developed with) and these packages:

```
pandas numpy requests geopandas shapely pyproj geojson folium mapclassify
contextily matplotlib seaborn plotly streamlit streamlit-folium
```

There is no `requirements.txt` yet. Until there is, install the packages manually:

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS/Linux
source .venv/bin/activate

pip install pandas numpy requests geopandas shapely pyproj geojson folium mapclassify contextily matplotlib seaborn plotly streamlit streamlit-folium jupyter
```

### Basemap API key

The maps use CARTO raster tiles. Set your key as an environment variable named `carto-api` before you start Jupyter. Hyphenated variable names can be awkward in some shells, so you may prefer to rename it in the notebook.

```powershell
${env:carto-api} = "your-key"   # PowerShell
```

### Running the pipeline

Run the notebooks from inside `Notebooks/`, because all paths are relative (`../data/...`).

1. **`kcmetro_etl.ipynb`**: run the cells in order through the frequency loop. This downloads the GTFS feed into `data/pgc_gtfs/` and writes `data/serv_dow.csv` and every file in `data/freq_data/`. The cells under *"Scripts: UNDER CONSTRUCTION"* are still being refactored and will not run yet.
2. **`mapping.ipynb`**: builds `data/shapes.geojson` and `data/route_shape_ids.csv`, then saves the HTML maps to `data/`. Open those HTML files in a browser to view them.
3. **Dashboard (stub)**:
   ```bash
   streamlit run Notebooks/streamlit_test.py
   ```

## Frequency output format

Each file in `data/freq_data/` covers one day type and one hour window. Hours go past 24 because GTFS times continue past midnight, so `20_30` means 8 PM to 6 AM the next morning.

| Windows | `0_30` (all day), `6_9` (AM peak), `9_15` (midday), `15_19` (PM peak), `19_24` (evening), `20_30` (night/owl) |
|---|---|
| Day types | `weekday`, `saturday`, `sunday` |

Columns:

| Column | Meaning |
|---|---|
| `route_short_name` | Public route name (e.g. `1`, `E Line`) |
| `mean_gap` | Average headway between departures at the same stop and direction |
| `median_gap` | Median headway. Less skewed by early or late outlier trips |
| `first` / `last` | Earliest and latest departure within the window |
| `trip_count` | Number of unique trips in the window |

Gaps are stored as pandas `Timedelta` strings (e.g. `0 days 00:15:00`). Read them back with `pd.to_timedelta`.

## Roadmap

From [Project-Outline.pdf](Project-Outline.pdf):

- [ ] Move the notebook logic into scripts (`gap_calc.py`, `display.py`, `info.py`) and add a `requirements.txt`
- [ ] Add Sound Transit and City of Seattle (agency IDs `40` and `23`)
- [ ] Add more gap metrics: min, max, and 25th/75th percentiles
- [ ] Streamlit + Folium dashboard that filters routes by frequency tier, day, and time window
- [ ] Summary charts (most and least frequent routes, span of service)
- [ ] Walkshed bubbles and travel time to key destinations
- [ ] Real-time data to compare scheduled and actual headways

## Data source and license

Schedule data comes from the [Puget Sound consolidated GTFS feed](https://gtfs.sound.obaweb.org/prod/gtfs_puget_sound_consolidated.zip) and King County Metro's GTFS. It is subject to each agency's terms of use. Basemap tiles are © [OpenStreetMap](https://www.openstreetmap.org/copyright) contributors and © [CARTO](https://carto.com/).
