# Methodology — Wildfire Susceptibility Prediction from Satellite Data

Self-contained description of the end-to-end method. Design rationale lives in
`docs/decisions.txt` (referenced as D01–D15); all numeric choices live in
`config/params.yml`. This document is the source for the thesis methodology
chapter and the context document for any assistant working on this project.

## 1. Problem statement

We predict **fire susceptibility per patch-week**: given the observable state
of a ~5 km × 5 km patch of land over the four weeks preceding a target week,
output the probability that a vegetation fire ignites in that patch during the
target week (D01).

This is deliberately *not* ignition-point prediction (individual ignitions are
near-random events driven by lightning and humans) and *not* spread prediction
(a separate future model; see §12). The unit of prediction is the
(patch, week) pair.

## 2. Study area and period

**California, 2017–2024** (D02). Training/validation years 2017–2022; years
2023–2024 are a strict temporal holdout used only for final evaluation.
Analysis CRS is EPSG:5070 (NAD83 Conus Albers, meters); raster patches are
read in the native Sentinel-2 UTM CRS (California spans zones 10N/11N).

## 3. Data sources

| Role | Source | Resolution / cadence | Access |
|---|---|---|---|
| Fire labels | FIRMS VIIRS SNPP Standard Processing | 375 m detections, ~2×/day | FIRMS API (key) |
| Burned perimeters (QC + future spread labels) | MTBS | per-fire polygons | public download |
| Optical / vegetation | Sentinel-2 L2A COGs (AWS Open Data) | 10–20 m, ~5-day revisit | anonymous HTTP |
| Thermal | MODIS LST MOD11A1 (daily, 1 km) | 1 km, fixed overpass times | Earthdata login |
| Weather | gridMET | 4 km, daily, CONUS | anonymous HTTP |
| Topography | Copernicus GLO-30 DEM | 30 m, static | anonymous HTTP |
| Land cover | ESA WorldCover | 10 m, static | anonymous HTTP |

ECOSTRESS (70 m thermal) was evaluated and demoted to a stretch experiment:
its ISS orbit yields irregular revisit at varying local times, which confounds
LST with acquisition hour and prevents an honest climatology (D04).

## 4. Fire event construction (labels)

1. Query FIRMS for VIIRS SNPP SP detections over the AOI and period.
2. Filter: `confidence = high`, `type = 0` (presumed vegetation fire).
3. Cluster detections into discrete **events** in space *and* time
   (ST-DBSCAN-style: spatial threshold 1000 m in projected coordinates,
   temporal gap threshold 48 h — tuned empirically, D08). Spatial-only
   clustering is insufficient: it merges nearby separate fires and fragments
   multi-day events. Distances are computed in EPSG:5070 or with a haversine
   metric, never in raw degrees.
4. Each event gets a centroid, first-detection date (the "fire week" anchor),
   detection count, and FRP statistics. Centroids are "detected fire origin",
   not literal ignition points (375 m pixels, detection lag).
5. MTBS perimeters are joined where available as a sanity check on clustering
   and stored for future spread-prediction work.

## 5. Negative sampling

Ratio ~3 negatives per positive (D09), of two kinds:

- **Spatial negatives**: same ecoregion and calendar month as a paired fire
  event, ≥ 20 km from any FIRMS detection that month.
- **Temporal negatives**: the *same locations that burned*, sampled in
  non-fire weeks (≥ 8 weeks away from any detection at that location).
  These prevent the model from merely memorizing fire-prone geography.

## 6. Input construction per sample

Each sample = one (patch, target week). Inputs cover four 7-day windows
ending 28, 21, 14, and 7 days before the target week (D03).

**Patch geometry (D13):** 256 × 256 px at 20 m = 5.12 km. Sentinel-2 10 m
bands are average-resampled to 20 m (SWIR B11/B12 are natively 20 m).

**Optical (per window, D07):** all Sentinel-2 acquisitions intersecting the
window are masked with the SCL layer (keep classes 4/5/6: vegetation,
not-vegetated, water) and merged by **per-pixel median** — robust to
undetected cloud edges and smoke. A window with < 60 % valid pixels is marked
missing; a sample with > 1 missing window is dropped. Stored bands:
B02, B03, B04, B08, B11, B12. Derived indices:
NDVI = (B08−B04)/(B08+B04), NDMI = (B08−B11)/(B08+B11),
NBR = (B08−B12)/(B08+B12).

**Thermal (per window, D04):** MOD11A1 daytime LST, 1 km, averaged over the
window; expressed as an **anomaly** against a per-pixel day-of-year
climatology built from 2003–2022 MODIS data.

**Weather (per window, D05):** gridMET daily values aggregated per window —
max temperature (tmmx), min relative humidity (rmin), wind speed (vs),
precipitation (pr), energy release component (erc), burning index (bi).

**Static:** elevation, slope, aspect (Copernicus DEM); land cover class
(WorldCover).

Coarse layers (1 km LST, 4 km weather) are kept at native resolution and fed
to the CNN through a separate low-resolution branch rather than pretending
they carry 20 m texture.

## 7. Normalization and climate non-stationarity (D06)

- Dynamic variables are expressed as **anomalies** relative to multi-year,
  per-location, week-of-year norms wherever feasible — comparable across
  years despite climate drift.
- Z-score normalization statistics (μ, σ per band/variable) are computed on
  **training years only (2017–2022)**, frozen to `norm_stats.json`, and
  reused unchanged on the test years. Per-year normalization is forbidden
  (future leakage; erases the drift under study).
- The 2023–2024 temporal holdout *measures* forward drift instead of hiding
  it, which is exactly what a "predict next week" claim requires.

## 8. Dataset assembly, storage, splits

- Cloud COG archives are read **once** per sample via HTTP range requests
  (only the ~5 km patch); results are materialized to local disk. Training
  never reads the network (D12). Extraction is embarrassingly parallel
  (shared-nothing workers, thread pool of ~12, per-request retry with
  exponential backoff, idempotent per-sample outputs with atomic
  write-then-rename, resumable via a checkpoint file).
- Per sample: one multi-band GeoTIFF/COG (windows × variables; georeferenced
  so any patch can be inspected in QGIS) **and** one aggregated feature row
  (medians/IQRs per window) in Parquet — the same extraction feeds both the
  tabular baseline and the CNN (D15).
- **Splits (D10):** made per fire *event* (all of an event's data on one
  side); spatial block cross-validation within the training years; strict
  temporal holdout 2023–2024. Split ID lists are committed to `data/splits/`.

## 9. Models (D11)

1. **Baseline — XGBoost** on the aggregated per-patch features. Trained and
   evaluated first; establishes the floor every later model must beat.
2. **CNN** on full patches: ResNet-18-scale encoder initialized with
   pretrained geospatial weights (TorchGeo / SSL4EO Sentinel-2); the four
   temporal windows stacked as channels; coarse inputs (LST anomaly, weather)
   via a small side branch concatenated at the head; weighted BCE or focal
   loss for class imbalance. Mixed precision, batch 16–32 on an 8 GB GPU.

Samples are dictionaries of tensors with optional keys
(`dynamic_satellite`, `static_topography`, `weather`, `anthropogenic=None`),
so future input groups plug in without restructuring.

## 10. Evaluation (D10, frozen before any modeling)

- Metrics: **PR-AUC** (primary), recall at fixed false-alarm rate,
  calibration curve. Accuracy is never a headline metric.
- All models are compared on the identical splits with the identical harness.
- Headline result: CNN vs. XGBoost on the same events.
- Ablations: 1 vs 2 vs 4 input weeks; with/without weather; with/without
  thermal.
- Experiment tracking: MLflow; every reported number traces to a run ID and
  a dataset version.

## 11. Known limitations (state in the thesis)

- FIRMS centroids approximate fire origin (375 m, detection lag).
- SCL does not reliably flag **smoke**; pre-fire windows in fire season may
  carry residual smoke contamination biasing the positive class.
- Susceptibility ≠ ignition certainty: ignition sources (lightning, humans)
  are not modeled in this phase.
- Single-region training (California); cross-region generalization untested
  unless the EU stretch goal is reached.
- 1 km thermal / 4 km weather provide conditions context, not fine texture.

## 12. Future work (out of scope, kept pluggable)

- Fire **spread** prediction after ignition (MTBS perimeters already
  collected as labels).
- Anthropogenic inputs (roads, settlements, land use) via the reserved
  optional sample key.
- ECOSTRESS 70 m thermal experiment; EU generalization test
  (frozen US normalization stats reused).
- Any user-facing application (desktop app etc.) — only after the model works.
