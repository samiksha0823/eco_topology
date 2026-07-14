# Technical Requirement Document
*Eco-Topology — System Design, Data Contracts & Implementation Specification*

| | |
|---|---|
| **Team** | Synorix |
| **Document Owner** |  |
| **Version** | 1.0 |
| **Status** |  companion to the Product Requirement Document |
| **Reference Implementation** | data_loader.py, pipeline.py, topology_features.py, train_model.py, main.py |

## 1. System Overview

Eco-Topology is a five-layer pipeline that turns raw GEDI LiDAR point clouds into field-ready deforestation alerts. Each layer is independently testable and communicates with the next via a well-defined tabular or object interface.

| Layer | Responsibility | Key Output |
|---|---|---|
| 1. Data Layer | Ingest GEDI L2A canopy-height point clouds and Hansen GFC lossyear labels. | Point-cloud DataFrame (lat, lon, value, label) |
| 2. TDA Engine | Tessellate into grid cells; build a Vietoris–Rips filtration per cell with GUDHI. | Persistence diagram (H0, H1) per cell |
| 3. Feature & ML Layer | Vectorize diagrams into summary statistics; train / run Random Forest or XGBoost. | Risk score per cell |
| 4. LLM Reporting Layer | Convert risk score + topological features into a plain-language brief. | Natural-language alert text |
| 5. Output Layer | Rank alerts, attach geolocation, expose for dashboard / export. | Ranked alert table |

## 2. Technology Stack

| Component | Technology | Purpose |
|---|---|---|
| Language | Python 3 | Core language for the entire analysis engine |
| Topology | GUDHI (RipsComplex, SimplexTree) | Vietoris–Rips filtration and persistent homology computation |
| Geospatial I/O | h5py, rasterio | Reading GEDI HDF5 granules and Hansen GeoTIFF tiles |
| Data handling | pandas, numpy | Point-cloud structuring, tessellation, feature tables |
| Machine learning | scikit-learn, XGBoost | Random Forest / XGBoost risk classification, evaluation metrics |
| LLM integration | Anthropic Claude API (or equivalent LLM API) | Natural-language early-warning report generation |
| Data sources | NASA GEDI L2A (LP DAAC), Hansen Global Forest Change (GFC) | Canopy structure and ground-truth loss labels |

## 3. Data Requirements

### 3.1 GEDI L2A (Canopy Height)

- Format: HDF5 (.h5), organized by beam (BEAM0000–BEAM1011).
- Fields used: lat_lowestmode, lon_lowestmode, rh (relative height metrics; rh100 used as canopy top height), quality_flag.
- Access: NASA Earthdata login required; recommended retrieval via the earthaccess Python library, filtered by bounding box and date range.
- Preprocessing: drop footprints where quality_flag == 0; replace GEDI fill values (-9999) with NaN and drop.

### 3.2 Hansen Global Forest Change (Validation Labels)

- Format: GeoTIFF, one *_lossyear_*.tif tile per 10×10 degree cell.
- Semantics: pixel value 0 = no loss detected; 1–23 = year of loss (1 → 2001 … 23 → 2023).
- Usage: sampled at each GEDI footprint's lat/lon via rasterio; binarized to a 0/1 label (loss_year > 0).

> **Temporal integrity requirement**
>
> For a genuine early-warning framing (not concurrent-loss detection), feature-source GEDI acquisitions must predate the Hansen loss year used as the label. Mixing same-year or post-loss GEDI data as “features” for that year's label leaks the outcome into the input.

## 4. Component Specifications

### 4.1 Data Ingestion (data_loader.py)

- load_gedi_h5(filepath, beam_names=None, quality_flag=True) → DataFrame[lat, lon, value, beam]
- load_hansen_labels(lossyear_tif_path, points_df, buffer_deg=0.0005) → points_df + [loss_year, label]
- generate_synthetic_forest(...) → synthetic healthy/degraded canopy point clouds for pipeline testing without external downloads.

### 4.2 Spatial Tessellation & Feature Extraction (pipeline.py, topology_features.py)

- tessellate(points_df, cell_size, lat_col, lon_col) assigns each point a cell_id on a regular lat/lon grid.
- compute_persistence_diagram(points, max_edge_length=None, max_dimension=2) builds the Vietoris–Rips complex and returns the raw persistence diagram.
- diagram_to_features(diagram, dims=(0,1)) vectorizes H0/H1 into: count, max_persistence, sum_persistence, mean_persistence, std_persistence, entropy.
- build_feature_table(points_df, cell_size, min_points_per_cell=10) → (X, y, meta): the full per-cell feature matrix, labels, and centroid metadata.

Tuning note: cell_size (degrees) directly trades off spatial resolution against point density per cell. min_points_per_cell is a hard floor below which a cell's topology is not considered reliable and is excluded rather than guessed at.

### 4.3 Risk Classification (train_model.py)

- train_and_evaluate(X, y, model_type='xgboost'|'random_forest', test_size=0.25) → trained model + held-out metrics (precision, recall, F1, ROC-AUC, confusion matrix).
- XGBoost default hyperparameters: 300 estimators, max_depth 4, learning_rate 0.05, subsample/colsample 0.8 — tuned for a moderate-sized tabular feature set, not raw pixels.
- feature_importance(model, X, top_n) surfaces which topological statistics drove the classifier, supporting the explainability requirement.
- generate_alerts(model, X, meta, threshold=0.5) → ranked DataFrame of cell_id, centroid_lat/lon, risk_score, alert flag.
- validate_against_hansen(alerts_df, hansen_labels) → precision/recall/F1 of the alert system against ground truth, independent of the classifier's own train/test split.

### 4.4 LLM Reporting Layer

The LLM layer is strictly a translation step: it receives already-computed, structured outputs and re-expresses them in natural language. It never independently computes a risk score or invents figures.

| Field | Type | Description |
|---|---|---|
| cell_id | string | Identifier of the flagged grid cell |
| centroid_lat / centroid_lon | float | Geolocation for map placement |
| risk_score | float [0–1] | Classifier output probability |
| h1_max_persistence | float | Longest-lived structural void — the primary early-warning signal |
| h1_entropy | float | How concentrated vs. diffuse the void signal is |
| top_features | list | Top contributing features from feature_importance() |
| n_points | int | GEDI footprint count in the cell, for confidence context |

Example prompt contract (input → output):

> **Prompt design principle**
>
> System prompt instructs the LLM to: (1) restate the risk score and its meaning in plain language, (2) explain that a persistent H1 void indicates a structural gap distinct from normal canopy variation, (3) avoid technical jargon (no “persistence diagram,” “Vietoris–Rips,” etc. in the output), and (4) never state a fact not present in the structured input.

- Model: Claude API (or equivalent), called only for cells above the alert threshold to control cost.
- Latency budget: LLM call is asynchronous / batchable — not on the critical path of classification.
- Failure mode: if the LLM call fails, the system must still surface the structured alert (risk score + features) without blocking on narrative generation.

### 4.5 Orchestration (main.py)

- CLI entry point: python main.py --model {xgboost|random_forest} --cell-size <deg> --threshold <float> --seed <int>.
- Default run mode uses generate_synthetic_forest() so the full pipeline is verifiable without external downloads before real data is wired in.
- Swapping to real data requires only replacing the data-loading block with load_gedi_h5() + load_hansen_labels(); all downstream calls (pipeline, train_model) are unchanged.

## 5. Data Contracts Between Layers

| Interface | Producer | Consumer | Shape |
|---|---|---|---|
| Point-cloud DataFrame | data_loader | pipeline.tessellate | columns: lat, lon, value, [label] |
| Feature table (X, y, meta) | pipeline.build_feature_table | train_model | X: one row per cell, ~14 numeric columns; y: binary; meta: cell_id + centroid |
| Alert table | train_model.generate_alerts | LLM layer / dashboard | cell_id, centroid_lat, centroid_lon, risk_score, alert (bool) |
| Alert brief | LLM layer | Output layer / dashboard | cell_id, narrative text, source figures echoed for traceability |

## 6. Non-Functional Technical Requirements

| Category | Requirement |
|---|---|
| Compute | Runs on a single standard machine (no GPU required); GUDHI Rips complex construction is the dominant cost per cell. |
| Scalability | Per-cell computation is embarrassingly parallel; cell-level processing can be distributed if region size grows. |
| Reproducibility | All stochastic steps (train/test split, RF/XGBoost) accept a seed parameter for reproducible runs. |
| Extensibility | Feature extraction (topology_features.py) is decoupled from the classifier, so the model can be swapped without touching TDA code. |
| Data provenance | Every feature row and alert must be traceable to its source GEDI granule and Hansen tile. |
| Package management | Python dependencies pinned in requirements.txt (gudhi, scikit-learn, xgboost, h5py, rasterio, numpy, pandas). |

## 7. Error Handling & Edge Cases

- Cells with fewer than min_points_per_cell footprints are excluded from the feature table, not silently scored — avoids fabricated confidence from sparse data.
- Infinite-death persistence features (points that never merge within the filtration range) are capped at the max finite death value before statistics are computed, avoiding inf/NaN propagation.
- Hansen tile / GEDI granule coordinate mismatches (no overlapping cell_ids) raise an explicit error in validate_against_hansen rather than returning a misleading empty result.
- LLM API failures degrade gracefully: the structured alert is still emitted; only the narrative field is marked unavailable.

## 8. Testing Strategy

| Level | Approach |
|---|---|
| Unit | Test topology_features functions against small, hand-computable point clouds with known persistence diagrams. |
| Integration | Run the full main.py pipeline against generate_synthetic_forest() output and assert healthy vs. degraded tiles separate cleanly. |
| Validation | Run train_model.validate_against_hansen on a real GEDI + Hansen tile pair; report precision/recall, not just accuracy. |
| LLM output QA | Spot-check that generated briefs contain no invented figures and match the structured input's risk score and top features. |

## 9. Deployment Considerations

- Current implementation is a batch pipeline (CLI-driven); suitable for scheduled re-runs rather than real-time streaming.
- Recommended near-term deployment: scheduled job (e.g. cron / workflow scheduler) triggered on new GEDI granule availability for a monitored region.
- Dashboard/export layer is decoupled from the modeling pipeline via the alert table contract, so it can be built independently (see UI/UX Brief).
- Secrets (NASA Earthdata credentials, LLM API keys) must be stored outside source control (environment variables / secret manager).

## 10. Appendix

#### A. requirements.txt

gudhi · scikit-learn · xgboost · h5py · rasterio · numpy · pandas

#### B. Reference documentation

- NASA GEDI data portal: https://gedi.umd.edu/
- Hansen Global Forest Change download: https://earthenginepartners.appspot.com/science-2013-global-forest/download_v1.7.html
- NASA Earthdata Search: https://search.earthdata.nasa.gov/
