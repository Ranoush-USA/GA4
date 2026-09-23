# GA4 Analytics Pipeline

A Declarative Automation Bundle that ingests **Google Analytics 4 (GA4)** web traffic data from Google BigQuery and processes it through a **Bronze → Silver → Gold** medallion architecture on Databricks.

## Architecture

```
Google BigQuery (GA4 events)
        │
        ▼  Lakeflow Ingestion (incremental)
  ┌───────────┐
  │  Bronze    │  Raw, unmodified events — immutable landing zone
  │ (bronze)   │  Nested structs/arrays preserved as-is
  └─────┬─────┘
        │
        ▼  Flatten, filter, deduplicate, merge
  ┌───────────┐
  │  Silver    │  Analytics-ready flattened table with geo, device,
  │ (silver)   │  traffic source & event params as columns
  └─────┬─────┘
        │
        ▼  Aggregated views for reporting
  ┌───────────┐
  │  Gold      │  Pre-built views: daily metrics by location,
  │ (gold)     │  event name, and page performance
  └───────────┘
```

## Project Structure

```
GA4/
├── databricks.yml              # Bundle configuration (variables, targets, includes)
├── README.md
├── .gitignore
└── resources/
    ├── jobs/
    │   └── GA4.yml             # Job definition: Silver → Gold (sequential tasks)
    └── notebooks/
        ├── parms_config.ipynb   # Shared widget-based parameter config
        ├── GA4_Bronze.md        # Bronze layer documentation
        ├── GA4-Silver.ipynb     # Bronze → Silver ETL (SQL + PySpark merge)
        ├── GA4_Silver_py        # Silver layer as SDP pipeline (DLT / streaming)
        └── GA4_Gold.ipynb       # Silver → Gold analytical views
```
## Data Layers

### Bronze

Raw GA4 events ingested from **Google BigQuery** via **Lakeflow Ingestion**. No transformation is applied — data lands as-is with nested structs and arrays preserved. Serves as an immutable landing zone for reprocessing if needed.

### Silver

Transforms bronze events into a clean, analytics-ready Delta table:

* Flattens nested structures (`event_params`, `geo`, `device`, `traffic_source`) into individual columns
* Filters out invalid rows (null `event_date` or `event_name`) into a quarantine set
* Deduplicates via `dropDuplicates()`
* Upserts (MERGE) into the silver table keyed on `_databricks_row_hash` + `event_timestamp` + `event_name` for rerun-safe incremental processing

The [GA4_Silver_py](#notebook-460393360706387) notebook contains an alternative Spark Declarative Pipeline implementation with streaming tables, expectations, and separate external/internal traffic handling.

### Gold

Creates pre-built analytical views on top of the silver table:

| View | Granularity | Metrics |
|---|---|---|
| `GA4_daily_metrics_location` | Date × Country × Region × City | Total events |
| `GA4_daily_metrics_event_name` | Event name × Timestamp | Total events |
| `GA4_daily_metrics_pages` | Date × Page | Total page views |

## Bundle Configuration

### Variables

| Variable | Default | Description |
|---|---|---|
| `catalog` | `workspace` | Unity Catalog catalog |
| `bronze_schema` | `bronze` | Bronze schema name |
| `silver_schema` | `silver` | Silver schema name |
| `gold_schema` | `gold` | Gold schema name |

### Targets

* **`dev`** (default) — Development target on the current workspace.
* **`prod`** — Production target deployed under `/Users/rana@ghazzi.com/.bundle/GA4/prod`.

### Job: `GA4_Job`

Two sequential notebook tasks with widget parameters passed from bundle variables:

1. **GA4-Silver** — runs `GA4-Silver.ipynb` with `catalog`, `bronze_schema`, `silver_schema`
2. **GA4-Gold** (depends on Silver) — runs `GA4_Gold.ipynb` with `catalog`, `silver_schema`, `gold_schema`

## Getting Started

### 1. Deployment

- Click the **deployment rocket** 🚀 in the left sidebar to open the **Deployments** panel, then click **Deploy**.

### 2. Running Jobs & Pipelines

- To run a deployed job or pipeline, hover over the resource in the **Deployments** panel and click the **Run** button.

### 3. Managing Resources

- Use the **Add** dropdown to add resources to the bundle.
- Click **Schedule** on a notebook within the bundle to create a **job definition** that schedules the notebook.

## Documentation

- [Declarative Automation Bundles in the workspace](https://docs.databricks.com/aws/en/dev-tools/bundles/workspace-bundles)
- [Declarative Automation Bundles Configuration reference](https://docs.databricks.com/aws/en/dev-tools/bundles/reference)
