# Azure Data Factory — Adventure Works Ingestion Project

## Overview
This project demonstrates a **dynamic, parameter-driven data ingestion pipeline** built using **Azure Data Factory (ADF)**. It ingests raw CSV files from GitHub into an **Azure Data Lake Storage Gen2 (ADLS Gen2)** bronze layer, following the **Medallion Architecture** pattern.

The project is designed as a foundation for a full-scale ETL solution. Future expansions will include data transformation using **Azure Databricks** and serving via **Azure Synapse Analytics**.

---

## Architecture

```
GitHub (Raw CSVs)
      ↓
Azure Data Factory
  └── Lookup Activity (reads config JSON from ADLS)
        ↓
  └── ForEach Activity (iterates over each file)
        ↓
  └── Copy Activity (ingests CSV → csv)
        ↓ success              ↓ failure
  Success Log (SQL)       Error Log (SQL)
        ↓
ADLS Gen2 — Bronze Layer
```

---

## Dataset
This project uses the **Adventure Works** dataset, which includes the following CSV files:

| File | Description |
|------|-------------|
| AdventureWorks_Calendar.csv | Date/calendar dimension |
| AdventureWorks_Customers.csv | Customer dimension |
| AdventureWorks_Products.csv | Product dimension |
| AdventureWorks_Product_Categories.csv | Product category dimension |
| AdventureWorks_Product_Subcategories.csv | Product subcategory dimension |
| AdventureWorks_Sales_2015.csv | Sales fact table - 2015 |
| AdventureWorks_Sales_2016.csv | Sales fact table - 2016 |
| AdventureWorks_Sales_2017.csv | Sales fact table - 2017 |
| AdventureWorks_Returns.csv | Returns fact table |
| AdventureWorks_Territories.csv | Territory dimension |

---

## Pipeline Design

### Dynamic Parameter-Driven Approach
Instead of hardcoding file paths, the pipeline reads a **JSON config file** (`adventure-params.json`) stored in ADLS. This file contains the source URL, destination folder name, and output file name for each dataset.

```json
[
    {
        "p_file_url"     : "github_path/AdventureWorks_Customers.csv",
        "p_folder_name"  : "Customers",
        "p_file_name"    : "Customers.csv"
    }
]
```

This makes the pipeline **scalable** — adding a new data source only requires adding a new entry to the JSON config file, with no changes to the pipeline itself.

---

### Activities

#### 1. Lookup Activity — `parameters-lookup`
- Reads the JSON config file from ADLS `config/` folder
- Outputs an array of file metadata to the ForEach activity

#### 2. ForEach Activity — `ForEach1`
- Iterates over each item in the Lookup output

#### 3. Copy Activity — `copy_github_data`
- **Source:** GitHub linked service (Anonymous authentication) with dynamic relative URL `@dataset().p_file_url`
- **Sink:** ADLS Gen2 linked service with dynamic path `bronze/@dataset().p_folder_name/@dataset().p_file_name`
- Retains csv format on landing

#### 4. Stored Procedure Activity — `success-log`
- Triggered on **successful** copy
- Logs pipeline name, file name, folder name, status and timestamp to Azure SQL

#### 5. Stored Procedure Activity — `error-log`
- Triggered on **failed** copy
- Logs pipeline name, file name, folder name, error message and timestamp to Azure SQL

---

## ADLS Folder Structure

```
storage-account/
├── bronze/
│   ├── Calendar/
│   │   └── Calendar.csv
│   ├── Customers/
│   │   └── Customers.csv
│   ├── Products/
│   │   └── Products.csv
│   └── ... (one folder per dataset)
├── config/
│   └── adventure-params.json
└── logs/
    └── incompatible_rows/
```

---

## Logging & Auditing

All pipeline runs are logged to an **Azure SQL Database** table for auditing and monitoring.

```sql
CREATE TABLE pipeline_run_log (
    log_id          INT IDENTITY(1,1) PRIMARY KEY,
    pipeline_name   NVARCHAR(200),
    activity_name   NVARCHAR(200),
    file_name       NVARCHAR(200),
    folder_name     NVARCHAR(200),
    error_message   NVARCHAR(MAX),
    status          NVARCHAR(50),
    logged_at       DATETIME DEFAULT GETDATE()
)
```

---

## Security

| Component | Authentication Method |
|-----------|----------------------|
| ADF → ADLS Gen2 | System Assigned Managed Identity |
| ADF → Azure SQL | SQL Authentication |
| SQL Password | Stored in Azure Key Vault, referenced via ADF Key Vault Linked Service |
| ADF → GitHub | Anonymous (public repository) |

---

## Azure Resources Used

- **Azure Data Factory** — Orchestration
- **Azure Data Lake Storage Gen2** — Raw data storage (Bronze layer)
- **Azure SQL Database** — Pipeline run logging
- **Azure Key Vault** — Secret management

---

## Git Integration
This ADF instance is integrated with **GitHub** for source control. All pipeline JSON definitions are version-controlled in this repository under the `adf/` directory.

---

## Future Enhancements
This project is intentionally scoped to the ingestion layer. Planned future additions include:

- **Silver Layer** — Data cleansing and transformation using Azure Databricks
- **Gold Layer** — Aggregated, business-ready data using Azure Synapse Analytics
- **Incremental Loading** — Watermark-based incremental ingestion pattern
- **CI/CD Pipeline** — GitHub Actions to automate deployment to a production ADF instance

---

## Enterprise Considerations (Notes)
While this is a personal project, the following enterprise best practices have been applied:

- Parameter-driven pipelines (no hardcoding)
- Managed Identity for ADLS authentication
- Secrets managed via Azure Key Vault
- Medallion Architecture folder structure
- Audit logging for every pipeline run
- Git-based source control for all ADF artifacts

In a full enterprise setup, this would also include:
- Separate Dev and Prod ADF instances
- Feature branch per developer workflow with PR approvals
- UAT environment with a dedicated branch
- Azure DevOps or GitHub Actions for CI/CD deployment
