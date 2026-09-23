# Bronze Layer

## Overview

Raw web traffic data is ingested from **Google BigQuery** into Databricks using **Lakeflow Ingestion**. New data lands in the Bronze table as-is, with **no transformation or processing applied** at this layer.

## Key Characteristics

| Aspect | Detail |
|---|---|
| **Source** | Google BigQuery (web traffic) |
| **Ingestion Method** | Lakeflow Ingestion (incremental) |
| **Processing** | None — data lands raw and unmodified |
| **Nested Fields** | Present — flattening and type casting are deferred to the Silver layer |

## Notes

- The Bronze layer preserves the raw payload exactly as received from BigQuery, serving as an immutable landing zone.
- Many fields are deeply nested (structs/arrays) and must be **flattened** and **type-cast** during transformation in downstream Silver/Gold layers.
- Because no processing occurs here, the Bronze table should be schema-flexible (e.g., `VARIANT` or raw string columns) to accommodate schema drift from the source.
- Retain an ingestion timestamp (`_ingested_at`) and source metadata for lineage tracking and reprocessing if needed. 