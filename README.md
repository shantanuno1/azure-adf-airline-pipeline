# Airline Data Pipeline — Azure Data Factory (Medallion Architecture)

An end-to-end ELT pipeline built in Azure Data Factory that ingests airline and flight data from multiple heterogeneous sources, processes it through a Bronze → Silver → Gold layered architecture with an incremental load pattern, and serves curated, top-N analytical output in Delta format on Azure Data Lake Storage Gen2.

## Architecture

```
On-Prem SQL Server ──(Self-Hosted IR, For Each loop)──┐
                                                        ├──► Bronze ──► Silver (5 parallel cleansing branches) ──► Gold (join + aggregate + rank) ──► ADLS Gen2 (Delta)
JSON file via HTTP (Web + Copy Activity) ─────────────┘
                                                        
Azure SQL Database ──(Incremental/Watermark Load)──► SqlToDatabase ──► feeds Gold layer dimension
```

**Orchestration:** a parent pipeline triggers the ingestion and transformation pipelines in sequence, so the full flow can be run end-to-end from a single execution.

## What this project does

**Multi-source ingestion**
- Pulls flight/airline/passenger/airport reference data from an on-premises SQL Server via a **Self-Hosted Integration Runtime**, looping dynamically over multiple source tables/files using a **For Each Activity** wrapped around a parameterized Copy Activity (not a hardcoded single copy).
- Pulls a JSON reference dataset (DimAirport) over HTTP using a Web Activity, then lands it via Copy Activity.

**Incremental loading (watermark pattern)**
- The `SqlToDatabase` pipeline implements a real incremental load: `Lookup_Last_load` and `Lookup_Latest_load` activities read/compare watermark values, and only new or changed data is copied, with the watermark persisted as a JSON file in the bronze layer. This avoids reloading the full dataset on every run.

**Silver layer — parallel cleansing (5 branches)**
- A single Data Flow (`DataTransformation`) processes five dimension/fact sources in parallel: DimAirline, DimFlight, DimPassenger, DimAirport, and DimFlightTransform.
- Includes column renames, derived columns (e.g. country, gender flag), full-name splitting into first/last name, row filtering (e.g. age > 25), type casting (e.g. ticket cost), and **Upsert** alter-row logic before sinking — rather than blind appends.

**Gold layer — join, aggregate, and rank**
- A second Data Flow (`Data Serving`) performs a **left outer join** between `DimFlightTransformation` and `DimAirline` on `airline_id`.
- Computes a derived `Duration_Hour` column from departure/arrival timestamps, including explicit handling of **overnight flights** (where arrival time is numerically earlier than departure time, requiring a +24 hour adjustment).
- Aggregates data by airline, ranks it using a window function, and filters to the **top 5** results before writing to the final sink — a genuine analytical output, not a pass-through copy.
- Final curated output lands at `gold/BusinessView-2` in **Delta format** on ADLS Gen2.

## Tools & Services Used

- Azure Data Factory (pipelines, mapping data flows, linked services, parameterized datasets)
- Self-Hosted Integration Runtime (on-prem SQL Server connectivity)
- Azure Data Lake Storage Gen2 (Delta format, medallion layout: bronze/silver/gold)
- Azure SQL Database
- Mapping Data Flows (joins, derived columns, conditional logic, aggregate, rank, filter, alter row/upsert)
- Incremental load / watermark pattern for change-driven ingestion

## Pipelines

| Pipeline | Purpose |
|---|---|
| `On_Prem_Injection` | Loops over on-prem SQL Server sources via Self-Hosted IR using a For Each Activity |
| `API_Injection` | Pulls a JSON reference file over HTTP (Web Activity) and lands it via Copy Activity |
| `SqlToDatabase` | Incremental/watermark-based load into Azure SQL Database |
| `SilverLayer` | Runs the `DataTransformation` data flow — 5-branch parallel cleansing of bronze data |
| `GoldLayer` | Runs the `Data Serving` data flow — join, aggregation, ranking, top-5 filter, Delta sink |
| `parent_pipeline` | Orchestrates the full pipeline run end-to-end |

## Known limitations / Future improvements

- The ADLS Gen2 linked service currently authenticates using a storage account key. The next iteration should move this credential into **Azure Key Vault** and reference it via a Key Vault-backed linked service, which is the recommended security practice for production pipelines.
- Pipelines are currently triggered manually; adding a **schedule or event-based trigger** would make ingestion fully automated.
- No retry/error-handling branches are configured yet; adding failure paths (e.g. alerting on activity failure) would make this production-ready.

## Screenshots

See the `/screenshots` folder for: the API ingestion flow, the on-prem For Each loop, the incremental watermark load pattern, the full 5-branch Silver layer data flow, the Gold layer join/aggregate/rank logic, and the overnight-flight duration expression.
