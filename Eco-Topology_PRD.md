# Product Requirement Document
*Eco-Topology — Early-Warning Deforestation Detection via Persistent Homology*

| | |
|---|---|
| **Team** | Synorix |
| **Document Owner** |  |
| **Version** | 1.0 |
| **Status** |  |
| **Related SDGs** | SDG 13 — Climate Action, SDG 15 — Life on Land |

## 1. Executive Summary

Eco-Topology is a deforestation early-warning system that analyzes the 3D structure of forest canopy — captured by NASA GEDI spaceborne LiDAR — using Topological Data Analysis (TDA) to detect selective logging and structural degradation before it is visible to conventional pixel-based satellite monitoring. A persistent-homology feature layer feeds a Random Forest / XGBoost risk classifier, and a large language model (LLM) translates model output into plain-language, field-ready alert briefs, closing the gap between rigorous mathematical detection and actionable conservation response.

> **Core insight**
>
> Standard monitoring detects deforestation by color/pixel change, which only registers after canopy is visibly gone. Eco-Topology detects the loss of structural connectivity in the canopy — a topological “void” — which forms well before the change is visible from above.

## 2. Problem Statement

Illegal and selective logging frequently occurs beneath an intact-looking upper canopy. Conventional remote-sensing pipelines (NDVI thresholds, Landsat-based change detection such as Hansen Global Forest Change) rely on spectral or color change and only flag loss after a meaningful area of canopy has already been cleared. This creates a structural blind spot:

- Pixel-based blind spot: standard ML applied to image color/surface-level pixel change cannot see sub-canopy structural loss.
- Damage detected too late: loss typically registers only after severe, visually obvious canopy area is gone.
- Hidden early-stage decay: subtle structural weakening, such as selective logging beneath an intact green canopy, goes undetected entirely.

Conservation agencies and forest departments therefore act reactively, often months after the damage that mattered most has already occurred.

## 3. Goals & Objectives

### 3.1 Product Goals

- Detect structural forest degradation earlier than pixel/color-based monitoring, using the topology of canopy point clouds rather than image color.
- Produce alerts that are directly usable by non-technical field teams, not just data scientists — via LLM-generated plain-language briefs.
- Validate detection quality objectively against an established ground truth (Hansen Global Forest Change).
- Build a pipeline that scales to any region with GEDI coverage (roughly 51.6°N–51.6°S) without per-region custom tuning.

### 3.2 Business / Impact Goals

- Support SDG 13 (Climate Action) by protecting forest carbon-sink capacity.
- Support SDG 15 (Life on Land) by catching biodiversity-relevant degradation at its structural root.
- Provide a low-cost monitoring layer that complements, rather than replaces, existing satellite programs.

## 4. Target Users & Personas

| Persona | Description | Primary Need |
|---|---|---|
| Field Ranger / Forest Guard | On-the-ground personnel patrolling a forest concession or protected area. | A short, plain-language alert telling them where to physically investigate, without needing to read a persistence diagram. |
| Conservation NGO Analyst | Monitors multiple concessions or protected areas remotely; triages alerts. | A ranked, geolocated alert dashboard and enough evidence detail to prioritize response and justify escalation. |
| Forest Department / Policymaker | Government body responsible for enforcement and reporting. | Defensible, mathematically grounded evidence for enforcement action or policy reporting. |
| Research Collaborator | Academic or technical partner (e.g. hydrology / TDA research context). | Access to raw topological features, model performance metrics, and validation results for further study. |

## 5. Scope

### 5.1 In Scope (Hackathon MVP)

- Ingestion of GEDI L2A canopy-height point-cloud data and Hansen GFC loss-year labels.
- Spatial tessellation of point clouds into grid cells sized for persistent-homology analysis.
- Persistent homology feature extraction (H0/H1 statistics) per cell via a Vietoris–Rips filtration.
- Binary risk classification per cell (Random Forest / XGBoost), trained and validated against Hansen labels.
- LLM-generated natural-language early-warning brief for each flagged cell.
- Ranked, geolocated alert output suitable for a map-based dashboard.

### 5.2 Out of Scope (This Phase)

- Real-time streaming ingestion of new GEDI acquisitions (current pipeline is batch/offline).
- Mobile field application for rangers (alerts are currently dashboard/report-based).
- Multi-sensor fusion beyond GEDI + Hansen (e.g. Sentinel radar, drone LiDAR) — noted as future roadmap.
- Automated enforcement workflows or integration with government case-management systems.

## 6. User Stories

| ID | As a… | I want to… | So that… |
|---|---|---|---|
| US-1 | Conservation NGO analyst | see a ranked list of high-risk cells across my region | I can prioritize which areas to investigate first |
| US-2 | Field ranger | read a short plain-language explanation of why a cell was flagged | I know what to look for without interpreting raw model output |
| US-3 | Forest department officer | see the model's precision/recall against Hansen ground truth | I can trust the alert enough to justify enforcement action |
| US-4 | Research collaborator | export the topological feature vectors per cell | I can conduct further analysis or reproduce results |
| US-5 | Analyst | re-run the pipeline on newly available GEDI acquisitions | I can track whether a previously flagged cell's risk is increasing or resolving |

## 7. Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-1 | System shall ingest GEDI L2A HDF5 granules and extract footprint-level canopy-height values (rh100) with quality-flag filtering. | Must |
| FR-2 | System shall ingest Hansen GFC lossyear GeoTIFF tiles and sample loss labels at GEDI footprint coordinates. | Must |
| FR-3 | System shall tessellate point clouds into configurable spatial grid cells. | Must |
| FR-4 | System shall compute a Vietoris–Rips persistence diagram (H0, H1) for each cell with sufficient point density. | Must |
| FR-5 | System shall vectorize each persistence diagram into summary statistics (count, max/sum/mean/std persistence, entropy). | Must |
| FR-6 | System shall train a binary risk classifier (Random Forest or XGBoost) on the feature table against Hansen-derived labels. | Must |
| FR-7 | System shall output a ranked, geolocated list of cells with a risk score and alert flag. | Must |
| FR-8 | System shall generate a plain-language early-warning report per flagged cell via an LLM, given its risk score and topological features. | Must |
| FR-9 | System shall report model evaluation metrics (precision, recall, F1, ROC-AUC, confusion matrix) against held-out Hansen labels. | Should |
| FR-10 | System shall support re-running the full pipeline on a new GEDI acquisition for a previously analyzed region. | Should |
| FR-11 | System shall expose feature importances so analysts can see which topological signals drove a given classification. | Could |

## 8. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Feature extraction and classification for a single region (hundreds of grid cells) should complete within a few minutes on standard hardware. |
| Scalability | Pipeline should scale to any GEDI-covered region without per-region code changes; only the grid-cell size parameter should need tuning. |
| Accuracy | Model should be validated against Hansen ground truth with reported precision/recall, not just accuracy, given class imbalance is expected. |
| Explainability | Every alert must be traceable back to the specific topological features that triggered it — no “black box” flags. |
| Usability | Field-facing outputs must be understandable without TDA or ML background — this is the explicit purpose of the LLM reporting layer. |
| Cost | LLM usage should be scoped to alert generation only (post-classification), not run on every cell, to control API cost at scale. |
| Data Provenance | Every alert must retain a reference to the source GEDI granule and Hansen tile used, for auditability. |

## 9. Success Metrics

- Model precision and recall against Hansen-confirmed loss labels (target: both ≥ 0.75 on held-out validation cells).
- Lead time: median time between an Eco-Topology alert and the corresponding Hansen-confirmed loss year, where measurable.
- Alert comprehensibility: a non-technical reviewer can correctly state why a cell was flagged after reading only the LLM brief.
- Coverage: pipeline successfully runs end-to-end on at least one real GEDI + Hansen tile pair beyond the synthetic demo.

## 10. Assumptions & Constraints

### 10.1 Assumptions

- GEDI footprint density within a chosen grid cell is sufficient (typically 10–15+ points) for a meaningful Vietoris–Rips filtration.
- Hansen lossyear labels are an acceptable, if imperfect, proxy for ground truth during model validation.
- Users have basic internet access to reach a map-based dashboard; no offline-first requirement in this phase.

### 10.2 Constraints

- GEDI coverage is limited to roughly 51.6°N–51.6°S, excluding higher-latitude forests.
- GEDI data latency (per NASA documentation) means the freshest data is not instantaneous — true real-time monitoring is not currently possible.
- LLM-generated reports depend on third-party API availability and are subject to that provider's rate limits and cost model.

## 11. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Sparse GEDI footprints in a region produce unreliable topological features | False negatives / unusable cells | Enforce a minimum-points-per-cell threshold; flag low-density cells as “insufficient data” rather than “healthy” |
| Hansen labels lag real-world loss by up to a year | Noisy training labels | Use pre-loss GEDI acquisitions as features and later-year loss as the label to preserve the early-warning framing |
| LLM produces an inaccurate or overconfident narrative | Loss of trust in alerts | Constrain LLM prompts to only restate structured model output; never let the LLM invent risk scores or numbers |
| False positives erode field-team trust over time | Alert fatigue, reduced adoption | Report precision/recall transparently; allow threshold tuning per deployment |

## 12. Milestones & Roadmap

Phased delivery, aligned with the project's technical workflow:

1. Phase 1 — Data Pipeline: GEDI + Hansen ingestion, spatial tessellation.
2. Phase 2 — TDA Feature Engineering: Vietoris–Rips filtration, persistence statistics.
3. Phase 3 — Model Training & Validation: Random Forest / XGBoost, validated against Hansen ground truth.
4. Phase 4 — LLM Reporting Layer: natural-language early-warning briefs.
5. Phase 5 — Alert Delivery: ranked, geolocated map dashboard.
6. Phase 6 — Continuous Monitoring: scheduled re-runs on new GEDI acquisitions to track risk trends over time.

*Future roadmap (post-hackathon, not in current scope): multi-sensor fusion, a mobile field app for rangers, and automated integration with enforcement case-management systems.*
