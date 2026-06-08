# aqueduct-dagster-poc

Aqueduct POC B — Dagster + dlt + GCS + FROST SensorThings

Two independent source pipelines, each running on its own schedule:

```
HydroVu API  → dlt → GCS (parquet) → HydroVuAdapter → CanonicalBundle → frost_load_hydrovu → FROST
CABQ API     → dlt → GCS (parquet) → CabqAdapter    → CanonicalBundle → frost_load_cabq    → FROST
```

Orchestrated by Dagster. Each pipeline has three assets:

| Asset | HydroVu | CABQ |
|-------|---------|------|
| Ingest (dlt → GCS) | `raw_hydrovu_readings` | `raw_cabq_readings` |
| Transform (GCS → CanonicalBundles) | `canonical_bundles_hydrovu` | `canonical_bundles_cabq` |
| Load (CanonicalBundles → FROST) | `frost_load_hydrovu` | `frost_load_cabq` |

---

## Setup

### Prerequisites
- [uv](https://docs.astral.sh/uv/) installed
- Docker + Docker Compose (for FROST)
- GCS bucket created
- HydroVu API credentials (client ID + secret)
- CABQ API credentials (if required)

### 1. Clone and install

```bash
git clone <repo>
cd aqueduct-dagster-poc-v2
uv sync
```

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env — fill in:
#   GCS_BUCKET_NAME
#   HYDROVU_CLIENT_ID, HYDROVU_CLIENT_SECRET
#   FROST_SERVICE_ROOT_URL (default: http://localhost:8080/FROST-Server)
```

### 3. Configure dlt

```bash
cp .dlt/secrets.toml.example .dlt/secrets.toml
# Edit .dlt/secrets.toml — fill in HydroVu credentials and GCS service account key
# Edit .dlt/config.toml  — set bucket_url to your GCS bucket under [destination.filesystem]
#                         — update [hydrovu] and [cabq] API URLs and start dates
```

### 4. Start FROST locally

```bash
docker compose up -d
# FROST available at http://localhost:8080/FROST-Server/v1.1
```

### 5. Launch Dagster UI

```bash
uv run dagster dev
# Open http://localhost:3000
```

---

## Running the pipelines

Both pipelines are independent — run either one without touching the other.

**From the Dagster UI:**
- Open http://localhost:3000
- Go to **Jobs** → select `hydrovu_pipeline` or `cabq_pipeline` → **Materialize all**

**From the CLI:**

```bash
# Full HydroVu pipeline
uv run dagster job execute -m aqueduct_dagster.defs.definitions -j hydrovu_pipeline

# Full CABQ pipeline
uv run dagster job execute -m aqueduct_dagster.defs.definitions -j cabq_pipeline

# Run a single asset
uv run dagster asset materialize -m aqueduct_dagster.defs.definitions --select raw_hydrovu_readings
```

---

## Project structure

```
aqueduct-dagster-poc-v2/
├── docker-compose.yml              # FROST + PostGIS
├── .env.example                    # env var template — copy to .env
├── .dlt/
│   ├── config.toml                 # dlt non-secret config (bucket URL, API URLs, start dates)
│   └── secrets.toml.example        # dlt secrets template — copy to secrets.toml
├── src/aqueduct_dagster/
│   ├── canonical/                  # shared data model — adapters and loader both import from here
│   │   ├── canonical_model.py      # dataclasses: CanonicalBundle, Thing, Location, Datastream, etc.
│   │   ├── canonical_constants.py  # shared units, sensors, observed properties, key helpers
│   │   └── base_adapter.py         # abstract BaseAdapter — all source adapters inherit from this
│   ├── adapters/
│   │   ├── hydrovu_adapter.py      # HydroVu → CanonicalBundle mapping
│   │   └── cabq_adapter.py         # CABQ → CanonicalBundle mapping
│   ├── pipeline/
│   │   ├── hydrovu_dlt_pipeline.py # dlt source + resource + pipeline factory for HydroVu
│   │   └── cabq_dlt_pipeline.py    # dlt source + resource + pipeline factory for CABQ
│   ├── defs/
│   │   ├── assets/
│   │   │   ├── ingest_hydrovu.py   # Dagster asset: raw_hydrovu_readings
│   │   │   ├── ingest_cabq.py      # Dagster asset: raw_cabq_readings
│   │   │   ├── transform_hydrovu.py# Dagster asset: canonical_bundles_hydrovu
│   │   │   ├── transform_cabq.py   # Dagster asset: canonical_bundles_cabq
│   │   │   └── load.py             # Dagster assets: frost_load_hydrovu, frost_load_cabq
│   │   └── definitions.py          # Dagster entry point — jobs, schedules, asset registry
│   └── loader/
│       ├── frost_loader.py         # FrostLoader (abstract) + FrostStaClientLoader (concrete)
│       └── watermark_store.py      # FrostWatermarkStore — per-run dedup via Dagster context
└── tests/
    ├── test_hydrovu_adapter.py
    └── test_cabq_adapter.py
```

---

## Architecture notes

**Canonical model as the contract**
Adapters produce `CanonicalBundle` objects. The FROST loader consumes them. Neither knows about the other's internals — the canonical model is the only shared interface.

**Incremental loading**
dlt tracks a cursor (`timestamp` field) per source. On first run it fetches from `initial_start_date`. On subsequent runs it fetches only records newer than the last cursor value. Cursor state is persisted to GCS alongside the parquet files.

**Watermark deduplication**
`FrostWatermarkStore` tracks the last observation timestamp successfully loaded into FROST per datastream. Each run skips any observation at or before the watermark — FROST has no built-in deduplication.

**Independent pipelines**
`hydrovu_pipeline` and `cabq_pipeline` are completely independent Dagster jobs. Each has its own schedule and its own terminal load asset (`frost_load_hydrovu` / `frost_load_cabq`). Running one never triggers or blocks the other.

---

## TODOs before first real run

**HydroVu**
- [ ] Implement `hydrovu_dlt_pipeline.py` — auth, location fetch, readings fetch, incremental cursor
- [ ] Implement `HydroVuAdapter` — `to_thing()`, `to_observations()`, `_build_datastreams()`
- [ ] Confirm `DEPTH_TO_WATER_PARAM_ID` (query HydroVu `/parameters` endpoint)
- [ ] Confirm HydroVu field names: `timestamp`, `gps.latitude`, `gps.longitude`
- [ ] Set HydroVu cron schedule in `definitions.py`

**CABQ**
- [ ] Implement `cabq_dlt_pipeline.py` — auth, fetch, incremental cursor
- [ ] Implement `CabqAdapter` — `to_thing()`, `to_observations()`, `_build_datastreams()`
- [ ] Confirm CABQ API endpoint and add to `.dlt/config.toml` `[cabq]` block
- [ ] Add CABQ credentials to `.dlt/secrets.toml`
- [ ] Set CABQ cron schedule in `definitions.py`

**FROST loader**
- [ ] Implement `frost_load()` in `load.py` — wire up service client, watermarks, loader
- [ ] Fill `FrostStaClientLoader` TODOs — link Thing/Sensor/ObsProp by ID ref (not re-nesting)
- [ ] Implement `FrostWatermarkStore.get/set` persistence (Dagster metadata, Postgres, or GCS)
- [ ] Implement `_max_phenomenon_time()` to recover watermark on fresh runs

**Canonical model**
- [ ] Replace `NO_DEFINITION` placeholders with real ODM2/QUDT URIs
- [ ] Resolve open question: store observations in feet or preserve source unit?
