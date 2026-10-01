# Loc Events — data pipeline

This folder contains the Python ingestion, Databricks/dbt transformations, Airflow orchestration, and PostgreSQL publishing code behind Loc Events, an event-discovery application for the Netherlands.

This is Mohammed Alfakih's portfolio fork of a HackYourFuture team project. See the [project overview](../README.md) and [my contributions](../docs/my-contributions.md) for the demo, team credit, and evidence of my work.

## How data reaches the application

```text
Ticketmaster Discovery API + optional price enrichment
    → Python ingestion and validation
    → Azure Blob Storage: partitioned raw JSON
    → Databricks/dbt: staging → intermediate → event marts
    → PostgreSQL: analytics.external_events
    → backend API → frontend
```

- **Ingestion:** fetches Dutch events in five 30-day windows, with up to five pages of 200 records per window, and deduplicates source IDs. This bounded collection does not guarantee complete source coverage.
- **Validation and landing:** validates records with Pydantic and reports rejects. The landed file preserves the original fetched records for downstream processing; ingestion fails when none are valid. Files contain one JSON object per line. Re-running a UTC run date replaces that date's file.
- **Price enrichment:** optional Ticketmaster and Universe records are landed separately under `enrichment/prices/`. Configure providers through `PRICE_ENRICHMENT_PROVIDERS`.
- **Transformation:** dbt builds `fct_external_events`, then `fct_external_events_enriched`. The backend-facing mart includes `logical_event_id` and inferred `venue_setting`: `indoor`, `outdoor`, `mixed`, or `unknown`.
- **Publishing:** refreshes `external_events` through staging and a transaction while preserving the published table's views, grants, and indexes. Missing events with Saved or Going references are retained with `is_published=false`. An empty mart is refused.

The current contract is documented in the [base mart YAML](dbt/models/marts/_fct_external_events.yml), [enriched mart YAML](dbt/models/marts/_fct_external_events_enriched.yml), and [dbt tests](dbt/tests/). Older training documents and `.example` files may describe the starter job-board project; use the active event models for current definitions.

## Folder map

| Path | Purpose |
|---|---|
| `src/ingestion/` | Fetch, validate, enrich prices, and land records |
| `dbt/models/staging/` | Read and normalize source files |
| `dbt/models/intermediate/` | Eligibility, daily deduplication, and price transformations |
| `dbt/models/marts/` | Event marts, venue inference, and processing metrics |
| `src/publishing/` | Publish events and optional health metrics |
| `src/common/` | Shared warehouse, Azure job, and build helpers |
| `airflow/dags/` | Scheduled pipeline and failure alerts |
| `tests/` | Python tests |
| `optional/streamlit/` | Optional dashboard code |

## Setup

Use Python **3.11 or 3.12** and `uv`. From the repository's `data/` folder:

```bash
uv sync --all-extras
```

Copy `.env.example` to `.env` and fill in your settings. In PowerShell:

```powershell
Copy-Item .env.example .env
```

Cloud stages require authorized access to Azure storage, Databricks, and PostgreSQL. A fork contains code and templates; it does not provision those services. Keep credentials in the ignored `.env` file.

## Developing locally

### Inspect ingestion without cloud storage

Set `SOURCE_API_URL` and `TICKETMASTER_API_KEY` in `.env`. The template uses the Ticketmaster Discovery endpoint. From `data/`:

```bash
uv run --env-file .env python -m src.ingestion.pipeline --local
```

This calls the live API and writes into `local-landing/` instead of Azure. You can supply a directory after `--local`; `--run-date YYYY-MM-DD` sets the run date. The event file is:

```text
local-landing/<LANDING_PREFIX>/events/ingest_date=YYYY-MM-DD/data.json
```

Set `PRICE_ENRICHMENT_PROVIDERS=` to disable optional price-provider calls during inspection. Cloud dbt models cannot read these local files.

### Run the cloud development stages

Keep these values aligned in `.env`:

| Setting | Personal development value |
|---|---|
| `LANDING_CONTAINER` | `dev` |
| `LANDING_PREFIX` | Your own prefix, such as `mohammed` |
| `LANDING_PATH` | `/Volumes/<catalog>/landing/dev/<prefix>/events` |
| `DBT_SCHEMA` | Your own schema, such as `dev_mohammed` |
| `BACKEND_PG_PUBLISH_SCHEMA` | `analytics_dev` |
| `BACKEND_PG_USER` | The authorized development writer |

Also configure `STORAGE_ACCOUNT`, the Databricks host, SQL warehouse HTTP path, catalog and personal token, and PostgreSQL connection values from `.env.example`.

`LANDING_PATH` points to the **events folder itself**; do not append `/events` again. Price-enrichment paths are derived from it unless overridden. Personal warehouse schemas are separate, but `analytics_dev` is shared: the latest development publish replaces its current data.

After signing into the authorized Azure account, run these commands from `data/`:

```bash
uv run --env-file .env python -m src.ingestion.pipeline
uv run --env-file .env dbt debug --target dev --project-dir dbt --profiles-dir dbt
uv run --env-file .env dbt build --target dev --project-dir dbt --profiles-dir dbt
```

A direct `dbt build` attempts the configured venue inference model and requires its dependencies and permissions. The scheduled DAG handles optional enrichment failures as described below.

After the required models and final mart build successfully, confirm the database destination and publish to the development schema:

```bash
uv run --env-file .env python -m src.publishing.sync --schema analytics_dev
```

The publisher defaults to `fct_external_events_enriched` → `external_events`. Its CLI also accepts `--mart`, `--table`, and `--schema`.

## What runs in Airflow

The main DAG, `final_project_pipeline`, runs daily at **09:00 Europe/Amsterdam**, without catchup and with one active run at a time:

```text
ingest → list_landing_files → dbt_build → publish_to_backend
```

`list_landing_files` is a temporary diagnostic task. In production, ingestion starts an Azure Container Apps job and waits for completion. Settings come from Airflow Variables; secrets are resolved inside tasks from the environment or Azure Key Vault.

The main DAG builds required event data first. If optional venue inference fails, the final mart uses `venue_setting=unknown`. Required model and final-mart failures stop publication. Optional Health Page metrics are handled separately; their failure does not undo successful event publication. Task failures use the configured Slack callback.

Local Astro runs mount the code through `airflow/docker-compose.override.yml` and load `data/.env`. Copy `airflow/airflow_settings.yaml.example` to `airflow/airflow_settings.yaml` before starting Astro. Its `INGEST_MODE=local` runs ingestion in the Airflow worker **and still writes to Azure**; this differs from the ingestion CLI's `--local` option. A personal `DATABRICKS_TOKEN` selects dbt's `dev` target. The team VM also has a separate, initially paused `final_project_pipeline_dev` DAG for integration development.

## Verification

From `data/`:

```bash
uv run pytest -q
```

For a cloud run, inspect the landed file, dbt build/test results, and rows in the configured PostgreSQL destination. Ingestion success alone does not prove that application data is fresh. The [data workflow](../.github/workflows/data-ci-cd.yaml) contains automated checks and deployment steps configured for the team environment.

This guide describes the checked-in code. Live cloud execution depends on the environment and credentials used for a run.
