# 🐾 Seattle Pet Licenses: Cloud-Scale ETL & Analytics Pipeline

## 📌 Project Overview

This repository implements a **production-style end-to-end ETL pipeline** for processing Seattle’s public pet licensing data using a **modern cloud data stack**. The solution ingests raw open data, applies robust data quality handling, and delivers **analytics-ready fact and dimension tables** in Snowflake using a **Medallion Architecture (Bronze → Silver → Gold)**.

The project demonstrates how real-world, imperfect civic data can be transformed into a **reliable analytical model** suitable for reporting, trend analysis, and downstream BI tools.

Key focus areas include:

* Handling **missing and inconsistent values**
* Implementing **surrogate keys and slowly evolving dimensions**
* Designing **scalable cloud-native pipelines**
* Bridging **Azure Data Factory + Snowflake** in an enterprise-style workflow

---

## 🏗️ Architecture Overview

<img width="1196" height="868" alt="image" src="https://github.com/user-attachments/assets/92ae50bf-8714-45dd-b7a7-3d01aa982826" />

**End-to-End Flow**

```
Seattle Open Data
        ↓
Azure Storage (Raw Landing Zone)
        ↓
ADF Mapping Data Flows
        ↓
Bronze Layer (Raw / Parquet)
        ↓
Silver Layer (Cleansed & Standardized)
        ↓
Gold Layer (Fact & Dimension Tables)
        ↓
Snowflake Analytics & BI Consumption
```

**Architectural Principles**

* Schema-on-read → schema-on-write progression
* Separation of concerns using medallion layers
* Idempotent and restartable transformations
* Warehouse-optimized dimensional modeling

---

## 🧰 Technology Stack

| Layer              | Tools                                 |
| ------------------ | ------------------------------------- |
| Ingestion          | Azure Data Factory                    |
| Storage            | Azure Blob Storage                    |
| Transformation     | ADF Mapping Data Flows, Snowflake SQL |
| Warehouse          | Snowflake                             |
| Modeling           | Star Schema (Fact & Dimensions)       |
| File Formats       | CSV → Parquet                         |
| Query & Validation | Snowflake Worksheets, DBeaver         |

---

## 📊 Dataset Description

**Source:** [Seattle Open Data Portal](https://data.seattle.gov/City-Administration/Seattle-Pet-Licenses/jguv-t9rb/data_preview)

**Domain:** Civic pet licensing records

Each record represents a licensed pet and includes:

* Animal name
* Species
* Primary and secondary breed
* License number
* License issue date
* ZIP code

⚠️ Like most public datasets, the data contains:

* Null and missing attributes
* Inconsistent species and breed values
* String-based dates requiring normalization

---

## 🧱 Data Modeling Strategy

The analytical layer follows a **star schema** optimized for reporting and aggregation.

<img width="1890" height="1025" alt="dataModel" src="https://github.com/user-attachments/assets/49218163-d1b9-484e-8e8f-0039082a14ba" />

### Fact Table

**`PET_LICENSE_FACT`**

* One row per pet license event
* Linked to dimensions via surrogate keys
* Stores time and location attributes for trend analysis

### Dimension Tables

* **`SPECIES_DIM`** – Manages species with surrogate keys
* **`BREED_DIM`** – Normalized breed information
* **`DATE_DIM`** – Calendar-based analysis
* **`ZIP_DIM`** – Geographic rollups

Surrogate keys ensure:

* Stable joins even when source values change
* Clean handling of missing or unknown attributes
* Warehouse-level performance optimization

---

## 📋 Dataset Schema & Column Details

**Dataset:** Seattle Pet Licenses
**Source:** City of Seattle Open Data Portal
**Granularity:** One record per licensed pet

| Column Name          | Data Type (Source) | Data Type (Post-ETL) | Description                                                       | Example Values                  | Data Quality Notes                                           |
| -------------------- | ------------------ | -------------------- | ----------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------ |
| `LICENSE_NUMBER`     | String             | VARCHAR              | Unique identifier assigned to each pet license issued by the city | `519748`, `722695`              | Some duplicates exist due to renewals; handled at fact grain |
| `ANIMALS_NAME`       | String             | VARCHAR              | Name of the licensed pet                                          | `Zen`, `Misty`, `Bella`         | Optional field; can be NULL                                  |
| `SPECIES`            | String             | VARCHAR → FK         | Species of the animal                                             | `Cat`, `Dog`, `Goat`, `Pig`     | Missing values handled using default dimension member        |
| `PRIMARY_BREED`      | String             | VARCHAR → FK         | Primary breed of the animal                                       | `Domestic Longhair`, `Siberian` | Inconsistent casing and spelling normalized                  |
| `SECONDARY_BREED`    | String             | VARCHAR → FK         | Secondary breed if applicable                                     | `Mix`, `NULL`                   | Frequently NULL; retained for completeness                   |
| `LICENSE_ISSUE_DATE` | String             | DATE                 | Date the license was issued                                       | `2015-12-18`, `2020-06-14`      | Originally stored as text; parsed and validated              |
| `ZIP_CODE`           | String             | VARCHAR → FK         | ZIP code of the pet owner                                         | `98117`, `98115`                | Some invalid ZIPs filtered or defaulted                      |
| `CREATED_AT`         | Timestamp          | TIMESTAMP            | Record ingestion timestamp (ETL generated)                        | `2025-03-13 18:41:22`           | Added for lineage and auditing                               |
| `UPDATED_AT`         | Timestamp          | TIMESTAMP            | Last update timestamp                                             | `2025-03-13 18:41:22`           | Used for incremental processing                              |

---

## 🧱 Dimensional Modeling Breakdown

### Fact Table: `PET_LICENSE_FACT`

| Column             | Description                  |
| ------------------ | ---------------------------- |
| `PET_LICENSE_SK`   | Surrogate key for fact table |
| `LICENSE_NUMBER`   | Business identifier          |
| `SPECIES_SK`       | Foreign key to SPECIES_DIM   |
| `PRIMARY_BREED_SK` | Foreign key to BREED_DIM     |
| `DATE_SK`          | Foreign key to DATE_DIM      |
| `ZIP_SK`           | Foreign key to ZIP_DIM       |

**Fact Grain:**
➡️ One row per **pet license issuance event**

---

### Dimension Table: `SPECIES_DIM`

| Column         | Description                  |
| -------------- | ---------------------------- |
| `SPECIES_SK`   | Auto-generated surrogate key |
| `SPECIES_NAME` | Normalized species name      |
| `UPDATED_BY`   | ETL process identifier       |

**Special Handling:**

* Missing species mapped to a default `"Missing"` record
* Supports late-arriving values via MERGE logic

---

### Dimension Table: `BREED_DIM`

| Column       | Description           |
| ------------ | --------------------- |
| `BREED_SK`   | Surrogate key         |
| `BREED_NAME` | Normalized breed name |
| `BREED_TYPE` | Primary / Secondary   |

---

### Dimension Table: `DATE_DIM`

| Column       | Description   |
| ------------ | ------------- |
| `DATE_SK`    | Surrogate key |
| `DATE_VALUE` | Calendar date |
| `YEAR`       | Year          |
| `MONTH`      | Month         |
| `QUARTER`    | Quarter       |

---

### Dimension Table: `ZIP_DIM`

| Column     | Description   |
| ---------- | ------------- |
| `ZIP_SK`   | Surrogate key |
| `ZIP_CODE` | ZIP code      |
| `CITY`     | City name     |
| `STATE`    | State         |

---

## ⚠️ Key Data Quality Challenges Addressed

| Issue                          | Resolution Strategy               |
| ------------------------------ | --------------------------------- |
| Missing species values         | Default surrogate key (`Missing`) |
| String-based dates             | Parsed into DATE using Snowflake  |
| Duplicate licenses             | Fact grain carefully defined      |
| Inconsistent casing            | Uppercasing and trimming          |
| Late-arriving dimension values | MERGE-based inserts               |


---

## 🔄 Medallion Architecture Implementation

### 🥉 Bronze Layer — Raw Ingestion

* Raw pet license data landed exactly as received
* Stored in Parquet format
* Minimal transformations
* Preserves original source fidelity

### 🥈 Silver Layer — Data Cleansing & Standardization

* Data type corrections (e.g., string → DATE)
* Null handling using default and inferred values
* Standardization of categorical fields
* Preparation for dimensional modeling

### 🥇 Gold Layer — Analytics-Ready Models

* Fact and dimension tables
* Surrogate key generation
* Deduplicated, validated records
* Optimized for BI and reporting tools

---

## 🔧 Key Transformations & Logic

### ✅ Missing Value Handling

* Explicit handling of null species and breeds
* Default “Missing” members added to dimensions
* Prevents broken joins and orphan fact records

### 🔑 Surrogate Key Strategy

* Auto-generated keys using Snowflake sequences
* Ensures immutability and consistency
* Supports late-arriving and previously unseen values

### 🔄 Incremental Dimension Loading

* New dimension values inserted automatically
* Existing dimension records updated safely
* Logic mirrors enterprise Slowly Changing Dimension (SCD) patterns

### 📅 Date Normalization

* License issue dates converted to proper DATE types
* Enables time-series analysis and calendar rollups

---

## ⚙️ Pipeline Orchestration

### Azure Data Factory

<img width="2230" height="1133" alt="dataFlowBreedDim" src="https://github.com/user-attachments/assets/a72411a3-4f48-44dd-9c5e-4e33bb679e53" />

* Mapping Data Flows for transformations
* Modular flows for dimensions and facts
* Debug-enabled development and validation

### Snowflake Processing

<img width="2239" height="1000" alt="snowflakeData" src="https://github.com/user-attachments/assets/338d8f7c-fa15-4423-9b90-f47dce513516" />

* External stages for Parquet ingestion
* File formats defined for schema inference
* SQL-based MERGE logic for dimension management

This dual-engine approach reflects **real enterprise pipelines**, where transformations span both orchestration and warehouse layers.

---

## 🔍 Data Validation & Quality Checks

* Record counts validated across layers
* Null checks on critical business keys
* Referential integrity enforced via surrogate keys
* Manual and SQL-based audits during development

---

## 📈 Analytical Use Cases Enabled

Once loaded, the data supports:

* Pet ownership trends by ZIP code
* Species distribution analysis
* Time-based license issuance trends
* City-level pet population insights
* BI dashboards in Power BI or Tableau

---

## 🚀 Future Enhancements

Planned extensions include:

* **Event-driven ingestion** using Azure Event Grid
* **Incremental refresh** instead of full reloads
* **Automated data quality checks**
* **Interactive dashboards** for civic insights
* **Cost optimization** using Snowflake clustering and pruning

---
