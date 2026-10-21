# Industrial Data Platform

An end-to-end data platform for manufacturing analytics: ingest shop-floor sensor and
production data, model it into a dimensional warehouse, compute OEE / yield / downtime KPIs,
and place a failure-prediction layer on top.

**Status:** W1 — industrial data map in progress

**Stack:** Python · dbt · Airflow · DuckDB / PostgreSQL · Azure (Blob, ADF, Synapse, IoT Hub) · Power BI

---

## Why this repository exists

Industrial data roles (Bosch, Siemens, Schneider Electric, Honeywell, ABB, Continental, ZF, Valeo)
ask for three things that a general data-engineering portfolio rarely shows:

1. **A real pipeline**, not a notebook — ingestion, layered storage, tests, scheduling, cost.
2. **Manufacturing metrics** — OEE, yield, downtime, MTBF/MTTR — computed from actual data shape.
3. **Honesty about data gaps** — knowing what a public dataset cannot tell you, and documenting it.

This repository is built around those three, on a weekly budget of ~10 hours.

## Scope

**In scope**

- Sensor and production data ingestion into a layered warehouse (ODS → DWD → DWS → ADS).
- dbt models with tests, an Airflow DAG with retries and backfill.
- A cloud path on Azure (Blob → ADF → Synapse) with a cost table.
- A one-page OEE / SPC dashboard.
- Binary failure classification on sensor data, reported with recall and false-alarm rate.

**Out of scope** (deliberately)

- Deep learning, NLP, streaming at scale, Kubernetes, Terraform.
- Any claim of production readiness — this is a portfolio project with documented assumptions.

## Architecture

```
  shop floor                edge                 cloud (Azure)
 ───────────           ───────────        ──────────────────────────
 sensor / PLC   ──►    OPC UA · MQTT  ──►  IoT Hub ──► Blob (raw)
                                                         │
                                                         ▼
                                            ┌────────────────────────┐
                                            │  ADF  ·  dbt           │
                                            └───────────┬────────────┘
                                                        │
        ODS  ──►  DWD  ──►  DWS  ──►  ADS            (local: DuckDB/Postgres)
         │        │         │        │
      raw load  cleaned   subject   Power BI
                & typed    marts     dashboard
                                                        │
                                                        ▼
                                          predictive maintenance notebook
```

| Layer | Responsibility | Implementation |
|---|---|---|
| raw | Immutable landing zone | Blob container / `data/raw` |
| ODS | 1:1 load of source files, no business logic | `ingest.py` |
| DWD | Typed, deduplicated, SCD-aware detail | dbt `staging` |
| DWS | Subject marts: machine, shift, day | dbt `marts` |
| ADS | Serving layer for BI | dbt `marts` + Power BI |

## Data

| Dataset | Origin | Size | Real or synthetic | Role |
|---|---|---|---|---|
| AI4I 2020 Predictive Maintenance | UCI ML Repository | 10,000 rows × 14 cols | **Real** sensor readings + failure labels | Primary source |
| `production_log` | Generated | 5 machines × 30 days × 2 shifts | **Synthetic** | Supplies planned/run time, counts, downtime |
| `failure_events` | Generated | ~1 per machine-month | **Synthetic** | Supplies failure type and repair minutes |
| Olist e-commerce | Kaggle | ~100k orders, 9 tables | Real business data | SQL and modelling practice only |

### Why synthetic tables are necessary

AI4I contains air/process temperature, rotational speed, torque, tool wear and failure labels —
but **no timestamp, no machine ID, no planned production time, no unit counts, and no downtime**.
OEE cannot be derived from it directly.

Rather than pretending otherwise, the missing fields are generated with explicit, documented
assumptions:

- one shift = 480 planned minutes, two shifts per day, 5 machines (M01–M05);
- ideal cycle time per product type (L / M / H);
- downtime, reject counts and repair minutes drawn from stated distributions;
- every generated table carries an `is_synthetic` flag in its data dictionary.

**Rule:** synthetic values are never presented as measured plant data. Interviewers care more
about how the gap is handled than about the numbers themselves.

## Metric catalog

Implemented progressively; business metrics in W2–W5, manufacturing KPIs in W5 and W7.

**Manufacturing KPIs**

| KPI | Formula | Source |
|---|---|---|
| OEE | Availability × Performance × Quality | `production_log` |
| Availability | Run Time / Planned Production Time | `production_log` |
| Performance | (Ideal Cycle Time × Total Count) / Run Time | `production_log` |
| Quality | Good Count / Total Count | `production_log` |
| Yield | Good Count / Total Count | `production_log` |
| Downtime by reason | Sum of downtime minutes per reason | `production_log` |
| MTBF | Run Time / Failure Count | `production_log` + `failure_events` |
| MTTR | Repair Minutes / Failure Count | `failure_events` |
| Cycle Time | Run Time / Total Count | `production_log` |

Six Big Losses map to the three OEE factors: equipment failure and setup/adjustment → Availability;
minor stops and reduced speed → Performance; process defects and reduced yield → Quality.

**Business metrics (Olist)** — GMV, order count, average order value, repurchase rate,
cancellation rate, fulfilment time (median / P90), category contribution, RFM + monthly retention.

## Repository layout

```
docs/         notes, glossary, architecture decisions        (W1–W4)
research/     JD tracker, company notes                      (W1+)
sql/          metric and KPI queries                         (W2–W5)
data/         local datasets — git-ignored, never committed
dbt/          dbt project: staging, marts, tests, snapshots  (W8)
airflow/      DAG definitions and plugins                    (W10)
dashboards/   Power BI report files                          (W14)
notebooks/    EDA, time-series features, modelling           (W6, W19)
```

## Roadmap

| Week | Deliverable | Status |
|---|---|---|
| W1 | Industrial data map: stack one-pager, 20-term glossary, 15 JD scan, repo skeleton | **in progress** |
| W2–W5 | SQL: 60 problems, 8 business metrics, star schema (5 tables), 5 KPI queries | todo |
| W6–W7 | `ingest.py` one-command ODS rebuild, synthetic `production_log`, time-series features | todo |
| W8–W11 | dbt 16 models / 32 tests, 1 incremental + 1 snapshot, Airflow 10-task DAG | todo |
| W12–W14 | Azure Blob / ADF / Synapse path, cost table, PII inventory | todo |
| W14–W15 | Power BI one-pager (OEE, yield, downtime, failures, MTBF/MTTR), SPC control chart | todo |
| W15–W23 | Failure classification, recall / false-alarm trade-off, one-page business conclusion | todo |

## Engineering conventions

- One branch per task (`docs/oee`, `feat/ingest`), merged to `main` after review.
- Commit messages follow `type: summary` — `feat`, `fix`, `docs`, `chore`, `refactor`, `test`.
- Secrets live in `.env`, never in git; `.env.example` documents required keys.
- Datasets and generated artifacts are git-ignored; the pipeline rebuilds them.
- Every model that uses synthetic fields states which fields are synthetic and why.

## Author

**Bi Ronglin** — MSc in Data, Artificial Intelligence & Cloud, Wuhan University of Technology × EFREI.
Target: 6-month industrial data internship, 2027-07 → 2027-12.
GitHub: [@R-olling](https://github.com/R-olling)