# Utilities Maintenance Data Engineering

An end-to-end Azure Data Engineering project that builds a scalable
lakehouse platform for utilities maintenance data. The project
demonstrates batch ingestion, incremental processing, data
transformation, Delta Lake Change Data Feed (CDF), dimensional modeling,
SQL analytics, orchestration, and Git-based source control.

## Architecture

``` text
Maintenance Source Data
        │
        ▼
Azure Data Factory
        │
        ▼
ADLS Gen2 - Bronze
        │
        ▼
Azure Databricks / PySpark
        │
        ├── Data Cleaning
        ├── Deduplication
        ├── Data Quality
        └── Transformations
        │
        ▼
ADLS Gen2 - Silver / Delta Tables
        │
        │  Delta Change Data Feed
        ▼
Incremental Gold Processing
        │
        ▼
ADLS Gen2 - Gold
        │
        ▼
Gold Star Schema
        │
        ├── Fact Maintenance
        └── Dimension Tables
        │
        ▼
Databricks SQL Warehouse
        │
        ▼
SQL Analytics Views
        │
        ▼
GitHub
```

## Technology Stack

  Layer                    Technology
  ------------------------ ------------------------------
  Data Ingestion           Azure Data Factory
  Data Lake                Azure Data Lake Storage Gen2
  Processing               Azure Databricks
  Language                 PySpark / SQL
  Storage Format           Delta Lake
  Incremental Processing   Delta Change Data Feed
  Governance               Unity Catalog
  Analytics                Databricks SQL Warehouse
  Source Control           GitHub

## Project Objectives

-   Build an end-to-end Azure data engineering pipeline.
-   Ingest maintenance data using Azure Data Factory.
-   Store raw data in an ADLS Gen2 Bronze layer.
-   Clean, standardize, deduplicate, and validate data in the Silver
    layer.
-   Implement incremental processing using Delta Change Data Feed.
-   Build a Gold dimensional model for analytics.
-   Create reusable SQL analytical views.
-   Orchestrate the pipeline from ingestion through Gold processing.
-   Maintain Databricks, ADF, and SQL artifacts in GitHub.

## Lakehouse Layers

### Bronze

The Bronze layer stores the ingested source data in ADLS Gen2.

Responsibilities:

-   Source data ingestion
-   Raw data preservation
-   File-based landing
-   Separation of source ingestion from downstream transformations

### Silver

The Silver layer contains cleaned and standardized Delta tables.

Main tables:

-   `assets`
-   `technicians`
-   `locations`
-   `maintenance`

Silver processing includes:

-   Data type standardization
-   Data cleaning
-   Deduplication
-   Data quality checks
-   Business transformations
-   Structured Delta tables

### Gold

The Gold layer provides an analytics-ready star schema.

#### Fact Table

-   `fact_maintenance`

#### Dimension Tables

-   `dim_asset`
-   `dim_technician`
-   `dim_location`
-   `dim_date`
-   `dim_maintenance_type`
-   `dim_status`

The fact table contains maintenance measures such as:

-   Downtime hours
-   Labor hours
-   Parts used
-   Parts cost
-   Labor cost
-   Total cost

Surrogate keys are used to connect fact records with dimension tables.

## Incremental Processing

The project uses Delta Lake Change Data Feed (CDF) to avoid repeatedly
processing the complete Silver dataset.

CDF is enabled on the Silver source tables:

-   `assets`
-   `technicians`
-   `locations`
-   `maintenance`

The incremental process:

1.  Reads the latest Delta table version.
2.  Reads the previously processed version from a control table.
3.  Checks whether new changes are available.
4.  Reads only the required CDF version range.
5.  Processes inserts and update post-images.
6.  Deduplicates changed business keys.
7.  Merges changes into Gold dimensions and facts.
8.  Updates the control table only after successful processing.
9.  Performs validation after the Gold load.

This provides an incremental architecture suitable for continuously
changing maintenance data.

## Gold Star Schema

``` text
                    dim_asset
                        │
                        │
dim_technician ─── fact_maintenance ─── dim_location
                        │
                        │
               dim_maintenance_type
                        │
                        │
                   dim_status

                   dim_date
```

### `fact_maintenance`

Contains maintenance work-order level measures and foreign keys to the
dimensions.

Key attributes include:

-   `work_order_id`
-   `asset_key`
-   `technician_key`
-   `location_key`
-   `maintenance_type_key`
-   `status_key`
-   `created_date_key`
-   `scheduled_date_key`
-   `completed_date_key`
-   `downtime_hours`
-   `labor_hours`
-   `parts_cost`
-   `labor_cost`
-   `total_cost`

### Dimensions

`dim_asset` contains asset information such as asset type, manufacturer,
model, location, criticality, and status.

`dim_technician` contains technician identity, skill level, team, and
experience.

`dim_location` contains city and region information.

`dim_date` provides calendar attributes including year, quarter, month,
week, day, and weekend indicators.

`dim_maintenance_type` describes maintenance type, category, planning
status, failure information, and priority.

`dim_status` contains work-order status and resolution code.

## SQL Analytics Layer

A separate `utilities_analytics` schema provides business-facing SQL
views on top of the Gold layer.

Analytical views include:

-   `maintenance_cost_summary`
-   `maintenance_by_asset`
-   `maintenance_by_month`
-   `technician_performance`
-   `maintenance_status_summary`

These views provide metrics such as:

-   Maintenance count
-   Total maintenance cost
-   Average maintenance cost
-   Labor cost
-   Parts cost
-   Downtime hours
-   Labor hours
-   Technician workload
-   Maintenance status distribution

The views query the Gold tables directly, so newly processed Gold data
is reflected automatically when the views are queried.

## Orchestration

Azure Data Factory orchestrates the data flow from source ingestion
through the Bronze and Silver layers and triggers the Gold processing.

Databricks job concurrency is controlled so that multiple upstream
triggers do not cause concurrent Gold processing runs.

The Gold job uses:

-   `max_concurrent_runs: 1`
-   Queueing for additional runs

This ensures that Gold processing runs sequentially and avoids competing
incremental loads.

## Data Model

The project uses surrogate keys generated from deterministic hashes for
several dimensions.

Examples:

``` text
asset_key
    = xxhash64(asset_id)

technician_key
    = xxhash64(technician_id)

location_key
    = xxhash64(location_id)

maintenance_type_key
    = xxhash64(
        maintenance_type,
        maintenance_category,
        planned_flag,
        failure_type,
        failure_severity,
        priority
    )

status_key
    = xxhash64(
        work_order_status,
        resolution_code
    )
```

Date keys use the `yyyyMMdd` format.

Example:

``` text
20260828
```

## Repository Structure

``` text
Utilities_DataEngineering/
│
├── ADF/
│   ├── dataset/
│   ├── factory/
│   ├── linkedService/
│   ├── pipeline/
│   ├── trigger/
│   └── publish_config.json
│
├── Bronze/
│
├── Silver/
│
├── Gold/
│
├── SQL/
│   └── Utilities_gold.dbquery.ipynb
│
├── README.md
│
└── .gitignore
```

> ADLS Gen2 data files and storage containers are not committed to the
> repository. The repository contains project code and configuration
> artifacts rather than the actual lake data.

## Data Quality and Validation

The pipeline includes validation checks across the Silver and Gold
layers.

Examples include:

-   Duplicate detection
-   Null validation
-   Referential integrity checks
-   Surrogate-key validation
-   Fact-to-dimension join validation
-   Row-count validation
-   Incremental version tracking

This helps ensure that records arriving in the Gold layer can be
reliably consumed by analytics workloads.

## Security Considerations

The project uses Azure identity and access controls for data access.

For source control:

-   Actual data files are not committed.
-   Secrets, passwords, storage keys, tokens, and client secrets should
    not be committed.
-   ADF linked services should use managed identity or secure credential
    references where possible.
-   Azure RBAC and Unity Catalog permissions control access to storage
    and tables.

## Key Engineering Concepts Demonstrated

This project demonstrates practical Data Engineering concepts including:

-   Azure Data Factory orchestration
-   ADLS Gen2 lakehouse architecture
-   Medallion architecture
-   PySpark transformations
-   Delta Lake
-   Delta MERGE
-   Change Data Feed
-   Incremental data processing
-   Control-table based processing
-   Deduplication
-   Data quality validation
-   Star schema dimensional modeling
-   Fact and dimension design
-   Surrogate keys
-   Unity Catalog
-   Databricks SQL
-   SQL analytical views
-   Job concurrency management
-   GitHub source control

## Project Outcome

The completed platform provides an end-to-end path from raw maintenance
data to analytics-ready datasets:

``` text
Raw Source
   ↓
ADF Ingestion
   ↓
Bronze
   ↓
Silver
   ↓
CDF Incremental Processing
   ↓
Gold Star Schema
   ↓
SQL Analytics Views
   ↓
Databricks SQL Warehouse
```

The architecture is designed to demonstrate production-oriented Azure
Data Engineering practices while remaining suitable as a portfolio
project.

## Future Enhancements

Potential future improvements include:

-   Automated CI/CD for Databricks and ADF
-   Additional data quality frameworks
-   Slowly Changing Dimensions Type 2
-   Pipeline monitoring and alerting
-   Azure Key Vault integration
-   Automated testing
-   Performance optimization and partitioning strategies
-   Additional business-oriented analytical views

## Author

**Hrs S**

Azure Data Engineering Portfolio Project
