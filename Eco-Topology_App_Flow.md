# App Flow Document
*Eco-Topology — End-to-End User & System Flows*

| | |
|---|---|
| **Team** | Synorix |
| **Document Owner** | Samiksha Bhardwaj |
| **Version** | 1.0 |
| **Status** | Draft — for hackathon submission (SheBuilds) |
| **Companion Docs** | PRD, TRD, UI/UX Brief |

## 1. Purpose

This document traces every major path a user or the system itself takes through Eco-Topology, from a scheduled GEDI data pull through to a ranger reading an alert in the field. It complements the UI/UX Brief (which defines what each screen looks like) by defining how a person or process moves between them.

## 2. Actors

| Actor | Type | Role in the flows below |
|---|---|---|
| Pipeline Scheduler | System | Triggers batch runs when new GEDI granules become available |
| NGO Analyst | Human | Reviews and triages the ranked alert list across a region |
| Field Ranger | Human | Consumes a single alert in the field, acts on it |
| Forest Dept. Officer | Human | Pulls evidence for enforcement / reporting |
| Research Collaborator | Human | Inspects model internals and raw features |
| LLM Service | System | Converts structured alert data into a plain-language brief |

## 3. High-Level System Flow (Batch Pipeline)

This is the backbone flow that produces every alert a human ever sees. It runs on a schedule or on-demand re-run, not per user action.

`New GEDI granule available`  →  `Ingest + Hansen label`  →  `Tessellate`  →  `TDA features`  →  `Risk score`  →  `LLM brief`  →  `Alert published`

**1. Trigger**
Pipeline Scheduler detects a new GEDI L2A granule for a monitored region (or an analyst manually triggers a re-run from Settings).

**2. Ingest & Label**
data_loader loads the GEDI point cloud and samples Hansen lossyear labels at each footprint, for cells where ground truth is available.

**3. Tessellate & Extract Features**
pipeline.build_feature_table grids the region into cells and computes persistent-homology features per cell above the minimum point-density threshold.

**4. Classify**
train_model scores every cell, producing a risk_score and alert flag per cell via generate_alerts.

**5. Generate Brief**
For cells above the alert threshold only, the LLM Service is called with the structured risk score + top features and returns a plain-language brief.

**6. Publish**
The ranked alert table (with attached briefs) is written to the alert store, becoming visible on the dashboard and eligible for notifications.

> **Failure isolation**
>
> If the LLM Service call fails or times out, the pipeline still publishes the structured alert (risk score + features) without a narrative, flagged “brief pending” rather than blocking the whole run.

## 4. Flow: NGO Analyst — Alert Triage

`Log in`  →  `Map / Alert Overview`  →  `Sort by risk`  →  `Open Alert Detail`  →  `Review evidence`  →  `Mark reviewed`

1. Analyst logs in and lands on the Map / Alert Overview screen for their default or last-viewed region.
2. Map renders alert markers colored/sized by risk_score; the side panel lists the same alerts ranked highest-first.
3. Analyst clicks a marker or list row → Alert Detail opens in-place (map context preserved).
4. Analyst reads the LLM brief; if unconvinced, expands the Evidence drawer to see risk score breakdown and top contributing features.
5. Analyst marks the alert Reviewed, optionally adding a note, which updates its status in the alert lifecycle (see Section 8).
6. Analyst repeats for the next-highest alert, or filters the list by status/region and continues triage.

## 5. Flow: Field Ranger — Single Alert Action

`Receive digest`  →  `Open Alert Detail`  →  `Read brief`  →  `Export PDF`  →  `Navigate in field`

1. Ranger receives a notification (email/SMS digest) for a new high-risk alert in their patrol area.
2. Opens Alert Detail directly from the notification link — no need to navigate the full dashboard.
3. Reads the 2–4 sentence plain-language brief and the location; jargon-free by design (see UI/UX Brief §6.1).
4. Exports the Alert Detail as a PDF for offline reference, since field connectivity may be limited.
5. Uses the “open in external map app” action to navigate to the flagged coordinates.
6. On return, optionally logs a field observation against the alert (future enhancement — not in current MVP scope).

## 6. Flow: Forest Department Officer — Evidence for Enforcement

`Open Region Summary`  →  `Filter high-confidence alerts`  →  `Open Alert Detail`  →  `Export evidence report`

1. Officer opens Region Summary to see aggregate risk trend for their jurisdiction.
2. Filters the cell table to alerts with both high risk_score and high n_points (confidence), to prioritize defensible cases.
3. Opens Alert Detail for a candidate case; reviews the model's validated precision/recall for that region (from the last Hansen-validated run) to support defensibility.
4. Exports a report combining the LLM brief, evidence drawer contents, and validation metrics for use in an enforcement or policy document.

## 7. Flow: Research Collaborator — Model Inspection

`Open Model & Data Panel`  →  `Review metrics`  →  `Inspect feature importances`  →  `Export raw features`

1. Collaborator opens the Model & Data Panel (role-gated, denser/technical view).
2. Reviews confusion matrix, precision/recall/F1/ROC-AUC from the latest Hansen-validated run.
3. Inspects the feature-importance chart to see which topological statistics are currently driving classifications.
4. From a specific Alert Detail, opens the Advanced view to see the raw persistence diagram (birth–death scatter) behind that cell's score.
5. Exports the raw feature table (X / y / meta) as CSV for offline analysis or reproducibility checks.

## 8. Alert Lifecycle (State Flow)

Every alert moves through a small set of states from the moment it's published to its resolution. This lifecycle is what the status filters in Region Summary and Alert Detail operate on.

| State | Entered when… | Available actions |
|---|---|---|
| New | Pipeline publishes a cell with risk_score ≥ threshold for the first time. | Review, Dismiss as false positive |
| Under Review | An analyst opens and engages with the Alert Detail (evidence viewed, note added). | Mark reviewed, Escalate, Dismiss |
| Escalated | An officer flags the alert for enforcement / field follow-up. | Attach field report, Resolve |
| Persisting | A subsequent pipeline run re-flags the same cell at or above threshold. | Same as Under Review; risk trend shown in Region Summary |
| Resolved / False Positive | A human confirms the outcome (either action taken, or determined not to be genuine loss). | Archive |

## 9. Edge Cases & Alternate Flows

| Scenario | System behavior |
|---|---|
| Grid cell has too few GEDI footprints (below min_points_per_cell) | Cell is excluded from scoring entirely rather than shown as “healthy” — avoids false reassurance from insufficient data. |
| LLM Service unavailable at publish time | Alert publishes with structured data only, status “brief pending”; brief is generated and attached on next successful retry. |
| No alerts above threshold in a region | Map / Alert Overview shows a reassuring empty state, not a blank/broken-looking screen (see UI/UX Brief §5.1). |
| Hansen tile has no overlap with the current GEDI granule's cells | validate_against_hansen raises an explicit error surfaced to the analyst, rather than silently returning empty validation metrics. |
| User role has no access to Model & Data Panel | Navigation entry is hidden rather than shown-then-blocked, to avoid confusing dead ends. |
| Same cell flagged across multiple consecutive runs | Alert transitions New → Persisting rather than creating a duplicate alert, and its risk-trend sparkline updates in Region Summary. |
