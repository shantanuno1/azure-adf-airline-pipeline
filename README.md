# Airline Data Pipeline — Azure Data Factory (Medallion Architecture)

An end-to-end ELT pipeline built in Azure Data Factory that ingests airline and flight data from multiple source systems, processes it through a Bronze → Silver → Gold medallion architecture, applies an incremental load pattern, and creates stakeholder-ready business views in Delta format on Azure Data Lake Storage Gen2.

## Architecture

This project follows a medallion architecture:

- **Bronze**: raw data ingestion from source systems
- **Silver**: cleaned and transformed datasets
- **Gold**: curated business-ready views for stakeholders

### Source Systems
- GitHub (JSON over HTTP)
- On-Prem SQL Server via Self-Hosted Integration Runtime
- Azure SQL Database using incremental watermark logic

### End-to-End Flow
GitHub / HTTP + On-Prem SQL Server + Azure SQL Database → Bronze → Silver → Gold → ADLS Gen2 (Delta)

The full workflow is orchestrated through a parent pipeline with failure alert handling.

## Business Questions Solved

### Business View 1
**Which are the top 5 airlines generating the highest total sales?**

### Business View 2
**How can flights be categorized into Short, Medium, and Long journeys based on journey duration?**

## What This Project Does

### 1. Multi-Source Data Ingestion
The project ingests data from three different source systems:

- **GitHub / HTTP source**: JSON data is accessed through Web Activity and landed in Bronze using Copy Activity.
- **On-Prem SQL Server**: multiple datasets are migrated using Self-Hosted Integration Runtime, For Each Activity, and parameterized Copy Activity.
- **Azure SQL Database**: incremental ingestion is implemented using a watermark-based pattern.

### 2. Parent Pipeline Orchestration
A parent pipeline orchestrates the ingestion workflows and controls pipeline execution across API, On-Prem, and SQL sources.

### 3. Failure Handling and Alert Notification
The project includes failure handling using ADF Web Activity and Azure Logic App to capture pipeline name, run ID, status, and error details and send alert notifications.

### 4. Silver Layer Transformations
The Silver layer transforms and curates raw Bronze data into structured datasets.

Silver transformations include:
- column selection and renaming
- derived columns
- full-name splitting
- filtering
- type casting
- structured dataset curation

The Silver layer processes datasets such as:
- `DimAirline`
- `DimAirport`
- `DimFlight`
- `DimFlightTransform`
- `DimPassenger`

### 5. Gold Layer Business Views
The Gold layer creates stakeholder-ready outputs through joins, aggregation, ranking, filtering, and derived metrics.

#### Business View 1 — Top 5 Airlines by Total Sales
This view identifies the top 5 airlines generating the highest total sales through:
- join between airline and transformed flight data
- aggregation of total sales
- ranking logic
- top 5 filtering

#### Business View 2 — Flight Categorization by Journey Duration
This view calculates journey duration and classifies flights into:
- Short
- Medium
- Long

It includes explicit handling of overnight flights where arrival time is earlier than departure time.

## Tools and Services Used

- Azure Data Factory
- Azure Data Lake Storage Gen2
- Azure SQL Database
- GitHub / HTTP source ingestion
- Self-Hosted Integration Runtime
- Mapping Data Flows
- Azure Logic App
- Execute Pipeline
- Web Activity
- Copy Activity
- For Each Activity
- Lookup Activity
- Delta format output

## Pipelines Used

| Pipeline | Purpose |
|---|---|
| `API_Injection` | Ingests JSON data from GitHub/HTTP source into the Bronze layer |
| `On_Prem_Injection` | Migrates on-prem source datasets using Self-Hosted IR and dynamic looping |
| `SqlToDatabase` | Performs incremental load from Azure SQL Database using watermark logic |
| `SilverLayer` | Runs Silver layer transformations and creates curated datasets |
| `GoldLayer` | Runs Gold layer transformations and builds business views |
| `parent_pipeline` | Orchestrates child pipelines and handles end-to-end execution |

## Data Sources

### On-Prem Data
- `DimAirline.csv`
- `DimFlight.csv`
- `DimPassenger.csv`

### GitHub Data
- `DimAirport.json`

### Azure SQL Data
- incremental source data loaded using watermark logic

## Medallion Layer Output

### Bronze Layer
Stores raw ingested files from all source systems.

### Silver Layer
Stores transformed datasets:
- `DimAirline`
- `DimAirport`
- `DimFlight`
- `DimFlightTransform`
- `DimPassenger`

### Gold Layer
Stores final stakeholder-facing business outputs:
- **Business View 1**: Top 5 Airlines by Total Sales
- **Business View 2**: Flight Categorization by Journey Duration

## Incremental Load Pattern

The Azure SQL ingestion pipeline uses a watermark-based incremental load approach:
- `Lookup_Last_load`
- `Lookup_Latest_load`
- Copy new or changed data only
- update watermark after successful execution

This avoids full reloads and improves efficiency.

## Screenshots

This repository includes 14 project screenshots covering:
- project overview
- ingestion pipelines
- parent pipeline orchestration
- Silver layer data flow
- Gold layer data flow
- business view inputs and outputs
- failure alert workflow
- Logic App integration
- monitoring and execution logs

## Repository Structure

```text
datasets/
screenshots/
README.md
```

## Future Improvements

- add trigger-based automation
- move secrets to Azure Key Vault
- implement CI/CD integration
- add additional data quality validations
- strengthen production-ready monitoring

## What I Learned

- designing end-to-end ELT pipelines in Azure Data Factory
- implementing Bronze, Silver, and Gold architecture
- ingesting data from multiple source systems
- using Self-Hosted Integration Runtime for on-prem connectivity
- implementing incremental load using watermark logic
- building stakeholder-facing views using Mapping Data Flows
- handling pipeline failures using Web Activity and Azure Logic App
- orchestrating pipelines with Execute Pipeline

## Conclusion

This project helped me strengthen my understanding of real-world Azure Data Engineering workflows, including ingestion, transformation, orchestration, incremental loading, and stakeholder-facing data serving.
