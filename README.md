# Airline Data Pipeline — Azure Data Factory (Medallion Architecture)

An end-to-end ELT pipeline built in Azure Data Factory that ingests airline and flight data from three different source systems, processes it through a Bronze → Silver → Gold medallion architecture with an incremental load pattern, and serves two stakeholder-ready business views in Delta format on Azure Data Lake Storage Gen2.

## Architecture

```
GitHub (JSON over HTTP, Web + Copy Activity)        ─┐
On-Prem SQL Server (Self-Hosted IR, For Each loop)  ─┼──► Bronze ──► Silver (5 parallel cleansing branches) ──► Gold (join + aggregate + rank) ──► ADLS Gen2 (Delta)
Azure SQL Database (Incremental/Watermark load)     ─┘

Orchestrated end-to-end by parent_pipeline, with a Web Activity failure alert on error.
```

## What this project does

**Ingestion from three source systems**
- **GitHub (HTTP/JSON):** a Web Activity calls a JSON file hosted on GitHub, then a Copy Activity lands it in the bronze layer (`API_Injection` pipeline).
- **On-premises SQL Server:** a Self-Hosted Integration Runtime connects to an on-prem SQL Server, with a **For Each Activity** looping over a parameterized Copy Activity to pull multiple source tables/files dynamically (`On_Prem_Injection` pipeline).
- **Azure SQL Database (incremental):** the `SqlToDatabase` pipeline implements a true watermark-based incremental load — `Lookup_Last_load` and `Lookup_Latest_load` activities compare watermark values, only new/changed data is copied, and the watermark itself is persisted as a JSON file in the bronze layer's `monitor` folder. A second Copy Activity (`Copy data - Watermark`) updates the watermark after each successful run.

**Orchestration with failure handling**
- A `parent_pipeline` executes On-Prem, API, and SQL ingestion pipelines in sequence, followed by a **Web Activity failure alert** — so a failure anywhere in the chain triggers a notification rather than failing silently.
- Verified via Monitor: 45+ pipeline runs over a 30-day period, including both successful and intentionally-debugged failed runs — this was iterated on and debugged like a real pipeline, not a one-shot tutorial run.

**Silver layer — parallel cleansing (5 branches)**
- A single Data Flow (`DataTransformation`) processes five dimension/fact sources in parallel: DimAirline, DimFlight, DimPassenger, DimAirport, and DimFlightTransform.
- Includes column renames, derived columns (e.g. country, gender flag), full-name splitting into first/last name, row filtering (age > 25), type casting (ticket cost), and **Upsert** alter-row logic before sinking — not blind appends.

**Gold layer — two stakeholder-ready business views**
A second Data Flow (`Data Serving`) builds two separate curated outputs:

- **Business View 1 — Top 5 Airlines by Total Sales:** a left outer join between `DimFlightTransformation` and `DimAirline` on `airline_id`, aggregated by `airline_name` to compute `Total_Sales`, ranked with a window function, and filtered to the top 5 (`Top_Rank`).
- **Business View 2 — Flight Categorization by Journey Duration:** a derived `Duration_Hour` column computed from departure/arrival timestamps — including explicit handling of **overnight flights** (where arrival time is numerically earlier than departure, requiring a +24 hour adjustment) — followed by a `JourneyType` classification into **Short / Medium / Long** duration categories.

Both outputs are written to ADLS Gen2 in **Delta format** (confirmed via the `_delta_log` transaction log folder present in the output containers).

## Tools & Services Used

- Azure Data Factory (pipelines, mapping data flows, linked services, parameterized datasets)
- Self-Hosted Integration Runtime (on-prem SQL Server connectivity)
- Azure Data Lake Storage Gen2 (Delta format, medallion layout: bronze/silver/gold containers)
- Azure SQL Database
- Mapping Data Flows (joins, derived columns, conditional logic, aggregate, rank, filter, alter row/upsert)
- Incremental load / watermark pattern for change-driven ingestion
- Pipeline monitoring and run history for debugging and validation

## Pipelines

| Pipeline | Purpose |
|---|---|
| `API_Injection` | Pulls a JSON file from GitHub over HTTP (Web Activity) and lands it via Copy Activity |
| `On_Prem_Injection` | Loops over on-prem SQL Server sources via Self-Hosted IR using a For Each Activity |
| `SqlToDatabase` | Incremental/watermark-based load from Azure SQL Database into ADLS Gen2 |
| `SilverLayer` | Runs the `DataTransformation` data flow — 5-branch parallel cleansing of bronze data |
| `GoldLayer` | Runs the `Data Serving` data flow — builds Business View 1 and Business View 2 |
| `parent_pipeline` | Orchestrates all ingestion pipelines end-to-end with a failure alert |

## Known limitations / Future improvements

- The ADLS Gen2 linked service currently authenticates using a storage account key. The next iteration should move this credential into **Azure Key Vault** and reference it via a Key Vault-backed linked service, which is the recommended security practice for production pipelines.
- Pipelines are currently triggered manually; adding a **schedule or event-based trigger** would make ingestion fully automated.

## Screenshots

See the `screenshots` folder for the full pipeline walkthrough: source ingestion (API, on-prem, SQL incremental load), parent pipeline orchestration with failure alert, the Silver layer's 5-branch cleansing data flow, the Gold layer's full join/aggregate/rank logic, both business view outputs with real data, and pipeline monitoring/execution logs showing actual run history.
