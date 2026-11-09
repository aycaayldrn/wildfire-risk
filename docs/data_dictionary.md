# Data Dictionary

Definitions, units, and valid ranges for every data artifact in the pipeline.
Paths are relative to the repo root; stage layout: `data/raw/` (exact API
dumps, read-only) → `data/interim/` (regenerable intermediates) →
`data/features/`, `data/labels/`, `data/splits/` (model-ready).

## 1. `data/raw/firms/*.csv` — FIRMS VIIRS SNPP SP detections (as downloaded)

| Column | Type | Unit / values | Notes |
|---|---|---|---|
| latitude, longitude | float | degrees, WGS84 | center of 375 m detection pixel |
| bright_ti4, bright_ti5 | float | Kelvin | brightness temperature, bands I-4 / I-5 |
| frp | float | MW | fire radiative power |
| scan, track | float | km | pixel footprint |
| acq_date | date | YYYY-MM-DD | |
| acq_time | int | HHMM UTC | |
| satellite | str | N (Suomi NPP) | |
| confidence | str | l / n / h | we keep **h** only |
| type | int | 0–3 | we keep **0** = presumed vegetation fire (1 volcano, 2 static land source, 3 offshore) |
| daynight | str | D / N | both kept |
| version | str | | SP (standard processing), not NRT |

## 2. `data/interim/events/events.parquet` — clustered fire events

One row per fire event (ST-DBSCAN cluster of detections).

| Column | Type | Unit | Notes |
|---|---|---|---|
| event_id | int | | primary key |
| lat, lon | float | degrees, WGS84 | detection centroid = "detected fire origin" |
| x_5070, y_5070 | float | m, EPSG:5070 | centroid in analysis CRS |
| first_detect | date | | anchor; fire week = ISO week containing it |
| last_detect | date | | |
| n_detections | int | | cluster size |
| frp_max, frp_sum | float | MW | |
| mtbs_id | str/null | | joined MTBS perimeter where available |
| ecoregion | str | | EPA Level III code (used by negative sampling) |

## 3. `data/labels/samples.parquet` — training samples (positives + negatives)

One row per sample = one (patch, target week).

| Column | Type | Unit | Notes |
|---|---|---|---|
| sample_id | str | | primary key, e.g. `evt000123` / `negS000456` / `negT000789` |
| event_id | int/null | | null for spatial negatives |
| lat, lon | float | degrees | patch center |
| target_week_start | date | Monday | the week being predicted |
| label | int | 0/1 | 1 = fire in patch during target week |
| sample_kind | str | fire / neg_spatial / neg_temporal | D09 |
| ecoregion | str | | |
| year | int | | drives temporal split |

## 4. `data/interim/patches/{sample_id}.tif` — per-sample multi-band COG

256 × 256 px, 20 m, Sentinel-2 UTM CRS of the covering tile, georeferenced
(inspectable in QGIS). Band order = window-major, W1 = oldest (D−28..D−22) …
W4 = newest (D−7..D−1):

| Bands | Content | Unit / range | Native res |
|---|---|---|---|
| per window: B02, B03, B04, B08, B11, B12 | SCL-masked median composite, surface reflectance | 0–10000 (scale 1e-4) | 10–20 m |
| per window: valid_mask | fraction of acquisitions valid per pixel | 0–1 | — |
| 4 × 6 band windows + 4 masks = 28 bands total | | | |

Indices are computed on the fly from stored bands (not stored):
NDVI = (B08−B04)/(B08+B04), NDMI = (B08−B11)/(B08+B11),
NBR = (B08−B12)/(B08+B12); all in [−1, 1].

Coarse per-window side inputs stored next to the patch in
`{sample_id}_coarse.npz`:

| Key | Shape | Unit | Notes |
|---|---|---|---|
| lst_anom | (4,) or (4, h, w) | K | MOD11A1 day LST minus 2003–2022 day-of-year climatology |
| tmmx | (4,) | K | gridMET window max of daily max temperature |
| rmin | (4,) | % | window min of daily min relative humidity |
| vs | (4,) | m/s | window mean wind speed |
| pr | (4,) | mm | window total precipitation |
| erc | (4,) | index | energy release component, window max |
| bi | (4,) | index | burning index, window max |
| elevation / slope / aspect | scalar or (h, w) | m / degrees / degrees | Copernicus GLO-30 |
| landcover | scalar | ESA WorldCover class code | majority class in patch |
| missing_windows | (4,) bool | | true where optical window failed D07 thresholds |

## 5. `data/features/features_agg.parquet` — aggregated features (XGBoost)

One row per sample_id. For each window w ∈ {1..4} and each of
{ndvi, ndmi, nbr}: `{index}_med_w{w}`, `{index}_iqr_w{w}` (median / IQR over
valid patch pixels). Plus `lst_anom_w{w}`, the six weather columns per
window, `elevation`, `slope_med`, `aspect_mode`, `landcover`, and the label
columns copied from §3. Trend features: `{index}_slope` = least-squares slope
of the window medians over the four windows (per 28 days).

## 6. `data/features/norm_stats.json` — frozen normalization statistics

μ and σ per band/variable, computed on **training years 2017–2022 only**
(D06). Never recomputed on test data; reused verbatim for any future
cross-region test.

## 7. `data/splits/*.csv` — committed split definitions

`train_ids.csv`, `valid_ids.csv` (spatial block folds: `sample_id, fold`),
`test_ids.csv` (all 2023–2024 samples). Splits are per *event*: all samples
deriving from one event are on one side (D10).

## 8. SCL classes kept as valid (D07)

4 = vegetation, 5 = not vegetated, 6 = water. All other classes (cloud
probabilities, shadow, snow, dark/defective, no-data) are masked out.

## 9. Checkpoint & provenance

`data/interim/extraction_done.parquet`: `sample_id, finished_at, s2_scenes
(list of product IDs used), status`. Lets extraction resume and records
exactly which upstream scenes produced each sample.
