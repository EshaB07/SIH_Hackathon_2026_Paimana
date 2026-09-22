# PAIMANA Early-Warning Risk System

**Smart India Hackathon 2026 — Problem Statement SIH26103**
Use case on a web-based integrated project-monitoring platform
Theme: Smart Automation | Category: Software | Team: Tantastic

## Overview

Infrastructure projects monitored under MoSPI's PAIMANA portal are currently tracked on a *reactive* basis — cost and schedule overruns are reported only after they have already occurred. This project introduces a predictive early-warning layer on top of the existing PAIMANA data: rather than reporting current status, it looks at a project's recent history and forecasts whether it is likely to deteriorate — through growing cost overrun or growing delay — over the following three months, while there is still time to intervene.

The system outputs a monthly-updating, explainable risk score per project, feeding directly into a decision-support dashboard for policymakers, project administrators, implementing agencies, and monitoring officials.

## Problem Statement

- **Problem Statement ID:** SIH26103
- **Title:** Use case on web-based integrated project-monitoring platform
- **Theme:** Smart Automation
- **Category:** Software

## Key Features

- **Two parallel predictive models** — one for cost escalation, one for schedule/time escalation — trained separately because the two behave very differently (cost overruns are rare and volatile; delays are common and more predictable).
- **Three-month-ahead prediction window**, built from a rolling four-month history of each project.
- **26-variable feature array**, engineered entirely from fields PAIMANA already collects, with an external-variable roadmap for future enrichment (no new mandatory data collection required for the prototype).
- **Explainable output** — every risk score ships with its top drivers via SHAP, so a reviewer sees not just that a project is flagged, but why.
- **Monthly re-scoring pipeline** producing a 0–100 risk score and a Low / Medium / High band per project.
- **Leakage-safe validation** methodology, split by project rather than by row.
- **Fully open-source technology stack.**

## Feature Groups

The 26-variable feature array is organized into four thematic groups derived from PAIMANA's Common Update Format (CUF) fields:

| Group | Features | What It Captures |
|---|---|---|
| Momentum | `progress_velocity`, `expenditure_velocity`, `stagnant_flag` | Progress and spending per month, and whether the project has stalled |
| History of Slipping | `delay_change`, `cost_overrun_change`, `n_date_revisions`, `n_cost_revisions` | Whether delay or overrun has been growing recently, and how often dates or costs were already revised |
| Deadline Pressure | `months_to_commission`, `months_since_sanction`, `pct_time_elapsed`, `required_velocity`, `velocity_gap` | Months remaining, share of time used, the pace needed to finish on schedule, and how far the current pace falls short |
| Money vs. Work | `expenditure_ratio`, `spending_progress_gap` | Share of budget already spent, and whether spending is outpacing physical progress |

## Technical Approach

### 1. Data Ingestion and Processing
- Monthly data sources: PAIMANA/OCMS historical updates, project metadata, project financials, and timelines.
- Data preparation in Python and Pandas: missing-value handling, data and business-rule validation, and construction of monthly project histories.

### 2. Feature Engineering
- Snapshot creation across monthly project lifecycles.
- Engineered feature groups as described above.
- Final feature array: 26 variables, including the original CUF fields.

### 3. Advanced Modeling
- Parallel XGBoost models — one for time overrun, one for cost overrun — each re-scored monthly.
- Planned external-variable roadmap: cement and steel WPI (added), equipment WPI (power, telecom, and mining only), state wage indices, a monsoon-exposure calendar, GDP and construction growth, and retrospective COVID/disaster controls.

### 4. Validation, Explainability, and Outputs
- Temporal holdout validation, plus repeated cross-validation split by project to prevent leakage.
- SHAP-based explainability layer for every prediction.
- Final risk classification supporting: identification of high-risk projects, resource prioritization, anticipation of cost and schedule jumps, validation of interventions via SHAP drivers, and KPI tracking.

### Technology Stack

- **Data processing:** pandas
- **Modeling:** XGBoost, scikit-learn
- **Explainability:** SHAP
- **Storage:** PostgreSQL
- **Frontend:** React

## Model Details

### Approach
The system predicts, for each project, whether cost overrun or schedule delay will meaningfully worsen over the next three months, based on the preceding four months of monitoring data. Two independent XGBoost models are used — one per target — because cost and time overruns follow substantially different distributions in the underlying data.

### Benchmarking
XGBoost was benchmarked against a plain logistic regression baseline on the same CUF fields. Adding the derived features and moving to XGBoost lifted time-overrun prediction accuracy from approximately 80% to approximately 85%, confirming that the added model complexity yields a real improvement rather than being complexity for its own sake.

### Validation Methodology
Validation used 5-fold cross-validation repeated three times, with folds split at the project level (not the row level), so that no model is ever evaluated on a project it was partially trained on. A temporal holdout was additionally used to check performance under realistic, forward-looking deployment conditions.

### Results

| Model | Metric | Result |
|---|---|---|
| Time-delay model | ROC-AUC | 0.925 |
| Time-delay model | PR-AUC | 0.88 |
| Cost-overrun model | Lift over random | ~5–7x |

The time-delay model performs strongly and is reliable enough to prioritize which projects warrant review. The cost-overrun model shows real but modest signal — useful as a supplementary flag rather than a standalone verdict — constrained primarily by the small number of historical cost-overrun examples in the available data (13 months of history across 5 states at prototype stage).

### Explainability
Every risk score is accompanied by its top drivers, computed via SHAP, so that a reviewer sees an actionable reason for each flag — for example, that spending is outpacing physical progress — rather than an opaque numeric score.

### Output
A monthly-updating risk score per project, comprising:
- Cost risk score
- Time risk score
- Combined risk band: Low / Medium / High

These feed directly into a dashboard for downstream decision-making.

## Data Quality Notes

Leakage-safe validation surfaced 62 duplicate national-project listings, which were cleaned from an initial 917 projects down to 855. Cleaning mules, duplicate grouping, and anomaly flags are part of the ongoing data-quality mitigation strategy; removal of certain entry anomalies was found to shift results by less than one point.

## Feasibility and Viability

| Dimension | Prototype (Built) | Full Project (Additional Variables + Deployment) |
|---|---|---|
| Technical Feasibility | Open-source stack (XGBoost, scikit-learn, SHAP); runs in about a minute on a laptop | Extra data from free public sources (WPI, state wage notifications, IMD, NSO); merged by month, state, and sector via standard pandas joins |
| Data Feasibility | Uses only fields MoSPI already collects; 14 derived variables need no new collection | Layer 3 needs a monthly fetch job; proposed additional CUF fields (contract type, land acquisition, clearances) would add small reporting effort |
| Economic Feasibility | Zero licensing cost | Same — all sources are free and public |
| Operational Feasibility | Outputs a score, a band, and drivers readable by non-technical officers | Monthly re-scoring and retraining runs automatically as new snapshots arrive |
| Scalability | Adding states or months needs no redesign | Built to run on the full portfolio (1,981 projects, 22 sectors) and the OCMS history |

### Risks and Mitigations

| Risk | Evidence | Mitigation Strategy |
|---|---|---|
| Short data history | 13 months, 5 states | Retrain monthly; validate against the two-decade OCMS history |
| Data quality | 62 duplicate national-project listings (917 → 855); 46 projects with entry anomalies | Cleaning mules, duplicate grouping, anomaly flags |
| Low cost-model precision | Cost predictions have low precision | Treat output as a relative ranking; supplement with cost-cause data |
| National indices don't vary by project | One index value per month | Use sector-conditional indices plus state-level wages and monsoon data |
| Leakage from publication lag | Prices are published after the fact | Use only lagged values available at prediction time |
| Limited validation range for additional variables | At most 13 distinct values per national series | Run ablation studies against the multi-year OCMS history |
| Marginal-value additional variables | Small number of events | Retain a variable only if it improves out-of-fold results |
| Deployment integration, security, and drift | Government data, changing patterns | Expose scores through an API layer on PAIMANA's existing role-based access; support open-source, on-premise hosting; monitor drift and retrain |
| Over-reliance on the score | Users may over-trust the output | Present drivers and confidence alongside the score; keep humans in the loop |

## Impact and Benefits

### Governance Benefits
Shifts project monitoring from reactive to proactive, enabling policymakers to prioritize interventions before cost or time overruns materialize.

### Economic Benefits
Early intervention on flagged high-risk projects can reduce the scale of cost overruns across a portfolio that PAIMANA's own published figures place at over ₹37 lakh crore.

### Transparency Benefits
SHAP-based driver analysis gives administrators an auditable, explainable reason for every risk flag, rather than an opaque score.

### Technological Benefits
A validated, open-source machine learning pipeline that is replicable across any government project-monitoring dataset, not specific to PAIMANA.

### Target Audience

- **Policymakers / MoSPI leaders** — portfolio-wide risk visibility and prioritization support
- **Project administrators** — early flags with explainable drivers, before overruns are locked in
- **Implementing agencies** — objective, data-driven performance benchmarking against peers
- **Monitoring officials** — reduced manual review burden via automated risk triage

## Research and References

### Dataset References
- PAIMANA Project Portal — https://paimana-proj.mospi.gov.in
- Press Information Bureau, Government of India — https://www.pib.gov.in

### Core Predictive Engine
- Chen, T., & Guestrin, C. (2016). *XGBoost: A Scalable Tree Boosting System.* KDD.
- Lundberg, S. M., & Lee, S.-I. (2017). *A Unified Approach to Interpreting Model Predictions.* NeurIPS.
- Pedregosa, F., et al. (2011). *Scikit-learn: Machine Learning in Python.*
- Flyvbjerg, B. (2003). *Megaprojects and Risk.*
- Roadmap data sources: Wholesale Price Index (DPIIT), Labour Bureau, India Meteorological Department (IMD)
