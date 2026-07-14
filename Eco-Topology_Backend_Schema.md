# Backend Schema Document
*Eco-Topology — Data Model, API Surface & Storage Architecture*

| | |
|---|---|
| **Team** | Synorix |
| **Document Owner** |   |
| **Version** | 1.0 |
| **Status** | Draft — companion to the Technical Requirement Document |

## 1. Storage Architecture Overview

Eco-Topology's backend separates three kinds of data with different access patterns: large immutable source files (GEDI granules, Hansen tiles), structured tabular/relational records (cells, features, predictions, alerts), and geospatial query workloads (map rendering, region lookups). A single relational database with geospatial extensions covers all three needs at hackathon-to-early-production scale.

| Layer | Recommended technology | Rationale |
|---|---|---|
| Relational + geospatial store | PostgreSQL + PostGIS | Native geometry types and spatial indexing for cell/alert map queries; strong relational integrity for the model-run lineage below |
| Raw source file storage | Object storage (e.g. S3-compatible bucket) | GEDI HDF5 granules and Hansen GeoTIFF tiles are large, immutable, and cheaper to keep as files than to load wholesale into the database |
| Feature / prediction cache | Same PostgreSQL instance (dedicated schema) | Feature tables and predictions are moderate in size and benefit from being queryable alongside alerts; a separate feature store is unnecessary at this scale |

## 2. Entity Overview

Twelve core entities cover the pipeline end-to-end, from source data through to a human-facing alert.

| Entity | Purpose |
|---|---|
| regions | A named, bounded area under monitoring (bounding box or polygon). |
| gedi_granules | Metadata for each ingested GEDI L2A HDF5 file. |
| hansen_tiles | Metadata for each ingested Hansen GFC lossyear GeoTIFF tile. |
| footprints | Individual GEDI LiDAR shots (points) after quality filtering. |
| grid_cells | Tessellated spatial cells within a region, the unit of analysis. |
| topological_features | Persistence-diagram summary statistics for one cell, one pipeline run. |
| model_runs | One execution of the training/scoring pipeline, with its hyperparameters and metrics. |
| predictions | A risk score for one cell from one model_run. |
| alerts | A prediction that crossed the alert threshold, with lifecycle status. |
| llm_reports | The generated plain-language brief for one alert. |
| users | Human accounts (rangers, analysts, officers, researchers). |
| audit_log | Append-only record of status changes and exports for traceability. |

## 3. Table Schemas

### 3.1 regions

| Column | Type | Notes |
|---|---|---|
| region_id | UUID, PK |  |
| name | TEXT | Human-readable region name |
| boundary | GEOMETRY(POLYGON, 4326) | PostGIS polygon; drives map viewport and cell tessellation extent |
| cell_size_deg | NUMERIC | Grid cell size used for tessellation in this region |
| alert_threshold | NUMERIC | Region-specific override of the default alert threshold |
| created_at | TIMESTAMPTZ |  |

### 3.2 gedi_granules

| Column | Type | Notes |
|---|---|---|
| granule_id | UUID, PK |  |
| region_id | UUID, FK → regions |  |
| source_filename | TEXT | Original GEDI04/GEDI02 .h5 filename, for provenance |
| product | TEXT | e.g. 'GEDI02_A' (canopy height) or 'GEDI04_A' (biomass) |
| acquisition_date | DATE | Used to enforce the pre-loss temporal-integrity rule (see TRD §3.2) |
| storage_path | TEXT | Object storage key for the raw .h5 file |
| ingested_at | TIMESTAMPTZ |  |

### 3.3 hansen_tiles

| Column | Type | Notes |
|---|---|---|
| tile_id | UUID, PK |  |
| tile_name | TEXT | e.g. Hansen_GFC-2023-v1.11_lossyear_10N_070W |
| storage_path | TEXT | Object storage key for the GeoTIFF |
| coverage | GEOMETRY(POLYGON, 4326) | Tile's spatial extent |
| gfc_version | TEXT | e.g. v1.11, for reproducibility |
| ingested_at | TIMESTAMPTZ |  |

### 3.4 footprints

| Column | Type | Notes |
|---|---|---|
| footprint_id | BIGSERIAL, PK |  |
| granule_id | UUID, FK → gedi_granules |  |
| cell_id | UUID, FK → grid_cells | Assigned at tessellation time |
| location | GEOMETRY(POINT, 4326) | PostGIS point (lon, lat) |
| canopy_value | NUMERIC | rh100 (canopy height) or agbd (biomass), per product |
| quality_flag | SMALLINT | Retained post-filter for auditability |
| loss_year | SMALLINT | Sampled from hansen_tiles; 0 = no loss, 1–23 = year |

Indexing: a spatial (GiST) index on footprints.location and a b-tree index on (granule_id, cell_id) support both map queries and per-cell feature extraction efficiently.

### 3.5 grid_cells

| Column | Type | Notes |
|---|---|---|
| cell_id | UUID, PK |  |
| region_id | UUID, FK → regions |  |
| cell_x | INTEGER | Grid column index (matches pipeline.tessellate) |
| cell_y | INTEGER | Grid row index |
| centroid | GEOMETRY(POINT, 4326) | For map marker placement |
| footprint_count | INTEGER | Denormalized count; cells below min_points_per_cell are excluded from scoring |

### 3.6 topological_features

| Column | Type | Notes |
|---|---|---|
| feature_id | BIGSERIAL, PK |  |
| cell_id | UUID, FK → grid_cells |  |
| model_run_id | UUID, FK → model_runs | Features are versioned per run so historical scores remain reproducible |
| h0_count, h0_max_persistence, h0_sum_persistence, h0_mean_persistence, h0_std_persistence, h0_entropy | NUMERIC (x6) | Matches topology_features.diagram_to_features output for H0 |
| h1_count, h1_max_persistence, h1_sum_persistence, h1_mean_persistence, h1_std_persistence, h1_entropy | NUMERIC (x6) | Matches diagram_to_features output for H1 — h1_max_persistence is the primary early-warning signal |
| canopy_mean, canopy_std | NUMERIC | Non-topological radiometric features, if included |

### 3.7 model_runs

| Column | Type | Notes |
|---|---|---|
| model_run_id | UUID, PK |  |
| region_id | UUID, FK → regions |  |
| model_type | TEXT | 'xgboost' or 'random_forest' |
| hyperparameters | JSONB | Full parameter set used, for reproducibility |
| cell_size_deg, min_points_per_cell, alert_threshold | NUMERIC / INTEGER / NUMERIC | Run-time pipeline configuration |
| precision, recall, f1, roc_auc | NUMERIC | Held-out evaluation metrics |
| hansen_validation_precision, hansen_validation_recall | NUMERIC | From validate_against_hansen, independent of the train/test split |
| started_at, completed_at, status | TIMESTAMPTZ / TIMESTAMPTZ / TEXT | 'running' \| 'completed' \| 'failed' |

### 3.8 predictions

| Column | Type | Notes |
|---|---|---|
| prediction_id | BIGSERIAL, PK |  |
| cell_id | UUID, FK → grid_cells |  |
| model_run_id | UUID, FK → model_runs |  |
| risk_score | NUMERIC [0–1] | Classifier output probability |
| top_features | JSONB | Top contributing features for this cell, from feature_importance() |
| created_at | TIMESTAMPTZ |  |

### 3.9 alerts

| Column | Type | Notes |
|---|---|---|
| alert_id | UUID, PK |  |
| cell_id | UUID, FK → grid_cells |  |
| prediction_id | BIGSERIAL, FK → predictions | The prediction that triggered this alert |
| status | TEXT | new \| under_review \| escalated \| persisting \| resolved \| false_positive |
| assigned_to | UUID, FK → users | Nullable |
| reviewer_note | TEXT | Nullable |
| first_flagged_at, last_updated_at | TIMESTAMPTZ (x2) | first_flagged_at is preserved across Persisting transitions |

### 3.10 llm_reports

| Column | Type | Notes |
|---|---|---|
| report_id | UUID, PK |  |
| alert_id | UUID, FK → alerts |  |
| brief_text | TEXT | The 2–4 sentence plain-language narrative |
| prompt_input_snapshot | JSONB | Exact structured payload sent to the LLM, for auditability (TRD §4.4 contract) |
| llm_model | TEXT | e.g. 'claude-sonnet-5' |
| status | TEXT | pending \| generated \| failed |
| generated_at | TIMESTAMPTZ |  |

### 3.11 users

| Column | Type | Notes |
|---|---|---|
| user_id | UUID, PK |  |
| name, email | TEXT (x2) |  |
| role | TEXT | ranger \| analyst \| officer \| researcher \| admin |
| region_ids | UUID[] | Regions this user monitors; drives default dashboard scope |
| notification_prefs | JSONB | Digest frequency, channel (email/SMS) |

### 3.12 audit_log

| Column | Type | Notes |
|---|---|---|
| log_id | BIGSERIAL, PK |  |
| entity_type, entity_id | TEXT / UUID | e.g. 'alert', alert_id |
| action | TEXT | e.g. 'status_changed', 'report_exported' |
| actor_user_id | UUID, FK → users | Nullable for system-initiated actions |
| details | JSONB | Before/after values where applicable |
| created_at | TIMESTAMPTZ |  |

## 4. Key Relationships

- regions 1—N grid_cells: a region is tessellated into many cells at ingestion/setup time.
- grid_cells 1—N footprints: many GEDI shots fall within one cell.
- grid_cells 1—N topological_features (one per model_run): features are recomputed and versioned each time the pipeline runs, never overwritten in place.
- model_runs 1—N predictions: one run scores every eligible cell in its region.
- predictions 0—1 alerts: an alert exists only for predictions at or above threshold.
- alerts 1—1 llm_reports: each alert has at most one current brief (regenerated reports create a new row and supersede the prior one, not an update-in-place, to preserve history).

> **Why features and predictions are versioned by model_run, not overwritten**
>
> Region Summary's risk-trend view and the audit trail both depend on being able to see how a cell's score evolved across runs. Overwriting rows in place would silently destroy that history.

## 5. API Surface (REST)

A thin REST layer over the schema above, mapping directly onto the pipeline's existing Python functions (see TRD §4) and the App Flow screens (see App Flow Document).

| Endpoint | Method | Purpose |
|---|---|---|
| /regions | GET / POST | List monitored regions; create a new one (triggers initial tessellation) |
| /regions/{id}/runs | POST | Trigger a new pipeline run for a region (ingest → features → score → alerts) |
| /regions/{id}/runs/{run_id} | GET | Poll run status and evaluation metrics |
| /regions/{id}/cells | GET | Grid cells with latest risk score, for map rendering (bbox-filterable) |
| /alerts | GET | Ranked alert list; filterable by region, status, risk range |
| /alerts/{id} | GET | Full Alert Detail payload: prediction, top features, llm_report |
| /alerts/{id}/status | PATCH | Update lifecycle status (reviewed / escalated / resolved / false_positive) |
| /alerts/{id}/report | GET | Export Alert Detail as PDF |
| /alerts/{id}/report/regenerate | POST | Re-request the LLM brief if status is 'failed' |
| /regions/{id}/summary | GET | Aggregate KPIs and risk trend for Region Summary |
| /model-runs/{id}/features | GET | Raw feature table export (CSV) for research use |
| /users/me | GET | Current user profile, role, and default region scope |

## 6. Data Retention & Versioning

- Raw GEDI/Hansen source files are retained indefinitely in object storage (small marginal cost, high provenance value).
- topological_features and predictions are retained per model_run to support historical risk-trend queries; older runs may be summarized/archived after a retention window rather than deleted outright.
- alerts and llm_reports are retained indefinitely while status is not resolved; resolved alerts are archived (not deleted) for audit purposes.
- audit_log is append-only and never modified or deleted, by design.

## 7. Security & Access Control

- Role-based access control (RBAC) via users.role gates the Model & Data Panel (researcher/admin only) and Settings (analyst/admin only), per the UI/UX Brief's role-based visibility guidance.
- region_ids scoping on the users table ensures analysts and rangers only see alerts for regions they are assigned to.
- NASA Earthdata credentials and the LLM API key are stored in a secrets manager, never in application config or source control.
- All status-changing API calls (PATCH /alerts/{id}/status, report exports) write to audit_log with the acting user, satisfying the PRD's defensible-evidence requirement.
