# Airline Data Pipeline — Azure Data Factory (Medallion Architecture)

An end-to-end ELT pipeline built in Azure Data Factory that ingests airline and flight data from three different source systems, processes it through a Bronze → Silver → Gold medallion architecture with an incremental load pattern, and creates two stakeholder-ready business views in Delta format on Azure Data Lake Storage Gen2.

## Architecture

GitHub (JSON over HTTP, Web + Copy Activity)        ─┐  
On-Prem SQL Server (Self-Hosted IR, For Each loop)  ─┼──► Bronze ──► Silver (5 parallel cleansing branches) ──► Gold (join + aggregate + rank) ──► ADLS Gen2 (Delta)  
Azure SQL Database (Incremental/Watermark load)     ─┘

Orchestrated end-to-end by `parent_pipeline`, with a Web Activity failure alert on error.

## Business Questions Solved

1. Which are the top 5 airlines generating the highest total sales?
2. How can flights be categorized into Short, Medium, and Long journeys based on flight duration?

## What This Project Does

### Ingestion from Three Source Systems

#### 1. GitHub (HTTP/JSON)
A Web Activity accesses a JSON file hosted on GitHub, then a Copy Activity lands it in the Bronze layer through the `API_Injection` pipeline.

#### 2. On-Premises SQL Server
A Self-Hosted Integration Runtime connects to an on-prem SQL Server, with a For Each Activity looping over a parameterized Copy Activity to pull multiple source tables/files dynamically through the `On_Prem_Injection` pipeline.

#### 3. Azure SQL Database (Incremental)
The `SqlToDatabase` pipeline implements a watermark-based incremental load pattern:
- `Lookup_Last_load` and `Lookup_Latest_load` compare watermark values
- only new or changed data is copied
- the watermark is persisted as a JSON file in the Bronze layer monitor folder
- a second Copy Activity (`Copy data - Watermark`) updates the watermark after each successful run

### Orchestration with Failure Handling

A `parent_pipeline` executes On-Prem, API, and SQL ingestion pipelines in sequence, followed by a Web Activity failure alert. This ensures failures trigger notification instead of failing silently.

### Silver Layer — Parallel Cleansing (5 Branches)

A single Data Flow (`DataTransformation`) processes five datasets in parallel:
- `DimAirline`
- `DimFlight`
- `DimPassenger`
- `DimAirport`
- `DimFlightTransform`

Silver layer transformations include:
- column renames
- derived columns
- full-name splitting into first and last name
- row filtering
- type casting
- structured curation of Bronze data into Silver outputs

### Gold Layer — Two Stakeholder-Ready Business Views

A second Data Flow (`Data Serving`) builds two curated outputs:

#### Business View 1 — Top 5 Airlines by Total Sales
A left outer join between `DimFlightTransformation` and `DimAirline` on `airline_id`, aggregated by `airline_name` to compute `Total_Sales`, ranked with a window function, and filtered to the top 5 using `Top_Rank`.

#### Business View 2 — Flight Categorization by Journey Duration
A derived `Duration_Hour` column is computed from departure and arrival timestamps, including explicit handling of overnight flights where arrival time is earlier than departure and requires a `+24 hour` adjustment. Flights are then classified into Short, Medium, and Long duration categories using `JourneyType`.

Both outputs are written to ADLS Gen2 in Delta format.

## Tools & Services Used

- Azure Data Factory
- Azure Data Lake Storage Gen2
- Azure SQL Database
- Self-Hosted Integration Runtime
- HTTP / GitHub source ingestion
- Mapping Data Flows
- Execute Pipeline orchestration
- Incremental load / watermark pattern
- Delta format output in ADLS Gen2

## Pipelines

| Pipeline | Purpose |
|---|---|
| `API_Injection` | Pulls a JSON file from GitHub over HTTP and lands it in Bronze using Web + Copy Activity |
| `On_Prem_Injection` | Uses Self-Hosted IR and For Each to ingest multiple on-prem SQL Server sources dynamically |
| `SqlToDatabase` | Performs incremental watermark-based ingestion from Azure SQL Database into ADLS Gen2 |
| `SilverLayer` | Runs the `DataTransformation` data flow to cleanse and standardize Bronze data |
| `GoldLayer` | Runs the `Data Serving` data flow to build stakeholder-facing business views |
| `parent_pipeline` | Orchestrates ingestion pipelines end-to-end and triggers failure alert logic |

## Data Sources Used

### On-Prem Source Files
- `DimAirline.csv`
- `DimFlight.csv`
- `DimPassenger.csv`

### GitHub Source
- `DimAirport.json`

### Azure SQL Source
- incremental source data loaded through watermark logic

## Medallion Layout

### Bronze
Stores raw ingested data from:
- GitHub/HTTP
- On-Prem SQL Server
- Azure SQL Database

### Silver
Stores transformed and curated datasets such as:
- `DimAirline`
- `DimAirport`
- `DimFlight`
- `DimFlightTransform`
- `DimPassenger`

### Gold
Stores stakeholder-ready business outputs:
- Top 5 Airlines by Total Sales
- Flight Categorization by Journey Duration

## Known Limitations / Future Improvements

- The ADLS Gen2 linked service currently authenticates using a storage account key. A future improvement is to move this credential into Azure Key Vault and reference it through a Key Vault-backed linked service.
- Pipelines are currently triggered manually. Adding a schedule or event-based trigger would make ingestion fully automated.
- Additional business rules and validation checks can be added to strengthen production-readiness.

## Screenshots

See the `screenshots` folder for the full project walkthrough, including:
- source ingestion pipelines
- parent pipeline orchestration
- Silver layer data flow
- Gold layer data flow
- business view outputs
- pipeline monitoring and execution logs

## What I Learned

- Building end-to-end ELT pipelines in Azure Data Factory
- Designing Bronze, Silver, and Gold layer architecture
- Ingesting data from multiple source systems
- Using Self-Hosted Integration Runtime for on-prem connectivity
- Implementing incremental loading with watermark logic
- Creating business-ready views using Mapping Data Flows
- Applying orchestration and failure-handling patterns in ADF
