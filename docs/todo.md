# Thesis To-Do — step-by-step plan to completion

Status as of 2026-11-09: repo skeleton + config + docs exist; **no pipeline
code implemented yet**. Target: done by January (hard fallback: February).
That leaves ~8 working weeks to a complete result — the schedule below is
compressed accordingly. Rules that keep it feasible:

- Stretch goals (ECOSTRESS, EU generalization, spread model, any UI) are
  **dead for the thesis**. They live in `methodology.md` §12 as future work.
- Writing starts in parallel the moment the baseline exists (step 5), not
  after experiments end.
- The XGBoost baseline (end of step 5) is the **safety floor**: a complete,
  defensible thesis exists at that point even if the CNN underperforms.

Check off items as they complete; when a step lands, flip its D-number in
`decisions.txt` to IMPLEMENTED and record the real code path.

---

## Phase 0 — Prerequisites (this week, mostly user actions)

- [ ] **Register NASA Earthdata account** (free; needed for MODIS LST;
      activation can take a day). Put credentials in env vars
      `EARTHDATA_USER` / `EARTHDATA_PASS` (see `config/params.yml`).
- [ ] **FIRMS API key** in env var `FIRMS_API_KEY` (user already has the key).
- [ ] **Buy the SSD** — 1 TB is approved; must be TLC NAND (e.g. SN850X /
      990 EVO Plus / Crucial T500 / Lexar NM790 in 1 TB). Avoid QLC
      (P3/P3 Plus, WD Green). Point `data/` at it once installed.
- [ ] **Create the conda env** from `env/environment.yml`; install PyTorch
      with CUDA wheels + torchgeo separately (see env file footer).
- [ ] Draw the California AOI polygon into `config/aoi.geojson`
      (state boundary or a bounding box; currently an empty file).

## Phase 1 — Labels (target: ~Nov 17–24) → D08

- [ ] `src/io/firms_api.py`: download VIIRS SNPP SP detections for the AOI,
      2017–2024, chunked by year; save exact dumps to `data/raw/firms/`.
      Filter high confidence + type 0 downstream, never in the raw dump.
- [ ] `src/geo/clustering.py`: spatio-temporal clustering (projected CRS,
      spatial 1000 m / temporal 48 h gap; tune in notebook 01). Output
      `data/interim/events/events.parquet` per data dictionary §2.
- [ ] Download MTBS perimeters for CA 2017–2024; join to events; visually
      sanity-check ~20 events in QGIS (clusters vs perimeters).
- [ ] `src/labeling/weekly_labels.py`: build positives + spatial negatives
      (same ecoregion/month, ≥20 km from detections) + temporal negatives
      (burned locations, non-fire weeks). Output `data/labels/samples.parquet`.
- [ ] **Checkpoint:** report event count and samples count. If events < ~1500,
      widen AOI (OR/NV/AZ) via `params.yml` before extraction starts.

## Phase 2 — Extraction (target: ~Nov 24 – Dec 12; the long pole) → D07/D12/D13

- [ ] `src/io/cogs_sentinel.py`: STAC search (earth-search AWS) + windowed
      COG reads for the 6 bands + SCL per acquisition.
- [ ] `src/features/veget.py`: SCL masking, per-window median composite,
      valid-fraction accounting, missing-window policy.
- [ ] `src/io/` MODIS LST reader (earthaccess) + `src/features/thermal.py`:
      day-of-year climatology (2003–2022) and window anomaly.
- [ ] gridMET reader + per-window weather aggregates.
- [ ] `src/features/topo_landcover.py`: DEM slope/aspect/elevation, WorldCover.
- [ ] Extraction orchestrator: shared-nothing thread pool (~12 workers),
      tenacity retries, atomic write-then-rename, resumable checkpoint
      (`data/interim/extraction_done.parquet`). Outputs per data dictionary
      §4 + aggregated rows §5 **in the same pass**.
- [ ] Pilot run on ~100 samples → inspect patches in QGIS, verify band order,
      composites, anomalies. Only then launch the full run (runs for days;
      it is resumable — reboots are fine).
- [ ] QC notebook: distribution plots per feature, missing-window rates,
      drop-rate accounting. Record kept/dropped counts for the thesis.

## Phase 3 — Splits + evaluation harness (parallel to late Phase 2) → D06/D10

- [ ] `src/modeling/datasets.py`: per-event spatial block CV folds within
      2017–2022; temporal test = 2023–2024; commit `data/splits/*.csv`.
- [ ] Freeze normalization: fit μ/σ on train years → `norm_stats.json`.
- [ ] `src/modeling/metrics.py`: PR-AUC, recall @ fixed false-alarm rate,
      calibration curve. **Frozen after this point** — every model uses it.

## Phase 4 — Baseline XGBoost (target: ~Dec 12–19) → D11

- [ ] `src/modeling/train_xgb.py`: train on aggregated features, tune lightly
      (depth, eta, class weight) on CV folds, evaluate once on the temporal
      test. Log everything to MLflow.
- [ ] Feature importance (gain + SHAP if time) — thesis explainability section.
- [ ] **Safety floor reached. Start writing the thesis now**: methodology
      chapter from `methodology.md`, data chapter from `data_dictionary.md`
      + QC numbers, baseline results chapter.

## Phase 5 — CNN (target: ~Dec 19 – Jan 16) → D11

- [ ] PyTorch `Dataset` reading per-sample COGs + coarse `.npz`, applying
      frozen normalization, missing-window masking.
- [ ] Model: TorchGeo/SSL4EO-pretrained ResNet-18 encoder, 4 windows stacked
      as channels (adapt first conv to 24+4 channels), coarse-input side
      branch (small MLP) concatenated at the head, weighted BCE or focal loss.
- [ ] Train with mixed precision (batch 16–32 on the 8 GB GPU); early
      stopping on CV fold PR-AUC; MLflow for every run.
- [ ] Final evaluation on the temporal test set with the frozen harness.
      Headline: CNN vs XGBoost, identical splits.
- [ ] If the CNN stalls: document honestly and lean on the baseline — a
      negative result with analysis is still a thesis result.

## Phase 6 — Ablations + writing (target: Jan 16 – end of Jan) 

- [ ] Ablations (each is one config change + one training run):
      1 vs 2 vs 4 input weeks; with/without weather; with/without thermal.
- [ ] Risk-map visualization over a held-out region/period; inspect top
      false positives in QGIS (georeferenced patches make this trivial).
- [ ] Finish writing: results, discussion, limitations (methodology §11),
      future work (§12). Every number cites an MLflow run ID.
- [ ] **February buffer** exists only for overruns of Phases 5–6 — nothing
      new is scheduled into it.

---

## Standing risks to watch

1. **Extraction overruns** (most likely failure). Mitigation: pilot first,
   start the full run before Dec 1, keep it resumable, write Phases 3–4
   code while it runs.
2. **Cloud/smoke drop rate** eats too many positives → widen AOI or relax
   `min_valid_px_frac` with a documented sensitivity check.
3. **Class imbalance in practice** worse than 1:3 after drops → reweigh loss,
   never resample the test set.
4. **Disk pressure on 1 TB** → delete superseded raw caches (checkpoint file
   says what is safe); keep only the current dataset version.
