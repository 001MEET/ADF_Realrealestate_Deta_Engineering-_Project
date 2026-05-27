# 🏠 Real Estate Data Engineering Project

> **End-to-End ETL Pipeline on Microsoft Azure**
> Azure Data Factory • Azure SQL Database • Star Schema DWH • Power BI Ready

---

## 📊 Project Stats

| Metric | Value |
|--------|-------|
| 🏘️ Listings | 12,000 |
| 👤 Leads | 50,000 |
| 💰 Sales | 8,456 |
| 🏢 Properties | 8,000 |
| 📣 Campaigns | 1,500 |
| 🧑‍💼 Agents | 500 |
| 🔁 ADF Pipelines | 3 |
| ✅ Pipeline Success Rate | 100% |

---

## 🏗️ Architecture Overview

This project implements a **Medallion Architecture** (Bronze → Silver → Gold) to ingest, validate, and transform real estate data into a queryable star-schema Data Warehouse.

```
┌─────────────────────────────────────────────────────────────────┐
│                      AZURE DATA FACTORY                         │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │            PL_Master_RealEstate_ETL  (Orchestrator)      │  │
│  │                                                          │  │
│  │   ┌─────────────────────┐   ┌──────────────────────┐    │  │
│  │   │  pl_IngestRawToStg  │──▶│ Pl_Transforma_Stg    │    │  │
│  │   │  (8 Copy Activities)│   │  ToDwh (13 Steps)    │    │  │
│  │   └─────────────────────┘   └──────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         │                                   │
         ▼                                   ▼
┌─────────────────┐                 ┌─────────────────┐
│  BRONZE LAYER   │                 │   GOLD LAYER    │
│  Azure Blob     │  ──── stg.* ──▶ │  dwh.dim_*      │
│  Storage (CSV)  │  Silver Layer   │  dwh.fact_*     │
└─────────────────┘                 └─────────────────┘
```

### Three Layers

| Layer | Storage | Tables | Purpose |
|-------|---------|--------|---------|
| 🟤 **Bronze** | Azure Blob Storage | CSV Files | Raw source data, no transformation |
| ⚪ **Silver** | Azure SQL — `stg.*` | 8 Staging tables | Validated, `stg_IsValid` flagged |
| 🟡 **Gold** | Azure SQL — `dwh.*` | 5 Dim + 4 Fact | Star-schema, BI-ready |

---

## 🗄️ Data Warehouse Schema (Star Schema)

![ER Diagram](Screenshots/ER_DIagram.png)

### Dimension Tables

| Table | Rows | Key Columns |
|-------|------|-------------|
| `dwh.dim_Agent` | 500 | AgentKey, AgentID, Region, City, ExperienceYears, IsCurrent |
| `dwh.dim_Property` | 8,000 | PropertyKey, PropertyType, SizeCategory, Region, City, IsCurrent |
| `dwh.dim_Listing` | 12,000 | ListingKey, AskingPrice, Status, ListingDays, IsCurrent |
| `dwh.dim_Campaign` | 1,500 | CampaignKey, Channel, Cost |
| `dwh.dim_Date` | auto | DateKey, Year, MonthNum, Quarter |

### Fact Tables

| Table | Rows | Grain |
|-------|------|-------|
| `dwh.fact_Sales` | 8,456 | One row per property sale |
| `dwh.fact_Leads` | 50,000 | One row per lead |
| `dwh.fact_Offers` | 25,000 | One row per offer |
| `dwh.fact_Viewings` | 0* | One row per viewing |

> *`fact_Viewings` — 40,000 rows exist in staging but pipeline investigation pending.

---

## ⚙️ ADF Pipelines

### 1. `PL_Master_RealEstate_ETL` — Master Orchestrator

![Master Pipeline](Screenshots/PL_Master_RealEstate_ETL.png)

Chains the two sub-pipelines sequentially. On failure, triggers `Sp_LogFailure`.

| Activity | Type | Duration | Status |
|----------|------|----------|--------|
| Run_Ingest | Execute Pipeline | 51s | ✅ Succeeded |
| Run_Transorm | Execute Pipeline | 49s | ✅ Succeeded |

---

### 2. `pl_IngestRawToString` — Bronze → Silver

![Ingest Pipeline](Screenshots/pl_IngestRawToString.png)

Reads 8 CSV files from Azure Blob Storage and loads them in parallel into staging tables.

**Copy Activities:**
- `Copy_Agents` — 18s &nbsp;|&nbsp; `Copy_Properties` — 18s
- `Copy_Campaigns` — 22s &nbsp;|&nbsp; `Copy_Offers` — 22s
- `Copy_Transactions` — 22s &nbsp;|&nbsp; `Copy_Listings` — 25s
- `Copy_Viewings` — 28s &nbsp;|&nbsp; `Copy_Leads` — 29s

---

### 3. `Pl_Transforma_StagingTodwh` — Silver → Gold

![Transform Pipeline 1](Screenshots/Pl_Transforma_StagingTodwh_1.png)
![Transform Pipeline 2](Screenshots/Pl_Transforma_StagingTodwh_2.png)

Loads dimensions in parallel → facts sequentially → row count check → log result.

| Step | Stored Procedure | Duration |
|------|-----------------|----------|
| 1 | Set_Start_Time | <1s |
| 2–6 | Sp_LoadDimDate / Agents / Campaigns / Listings / Properties | 16–17s |
| 7 | Sp_LoadFactSales | 6s |
| 8 | Sp_LoadFactoffers | 6s |
| 9 | Sp_LoadFactLeads | 6s |
| 10 | LKP_RowCount (Lookup) | 17s |
| 11 | IF_RowCountCheck | 2s |
| 12–13 | Set_Success → Sp_LogSuccess | 6s |

---

## ✅ Data Quality Checks

### A1 — Staging vs DWH Row Counts
![Row Count Check](Screenshots/Staging_Row_Vs_DWH_Row_Counts.png)

### A2 — Staging Invalid Rows (All Zero)
![Invalid Rows](Screenshots/Staging_invalid_rows_summary.png)

### A3 — NULL Check on Critical Fact Columns (All Zero)
![Null Check](Screenshots/NULL_check_on_critical_DWH_fact_columns.png)

| Check | Fail Count | Result |
|-------|-----------|--------|
| fact_Sales NULL SalePrice | 0 | ✅ PASS |
| fact_Sales NULL ListingKey | 0 | ✅ PASS |
| fact_Sales NULL AgentKey | 0 | ✅ PASS |
| fact_Leads NULL ListingKey | 0 | ✅ PASS |
| fact_Offers NULL OfferPrice | 0 | ✅ PASS |

---

## 📈 Analytical Reports (Power BI Ready)

### B1 — Sales Funnel (Leads → Viewings → Offers → Sales)
![Sales Funnel](Screenshots/Sales_Funnel__Leads___Viewings___Offers___Sales_.png)

### B2 — Sales Performance by Region & Property Type (Monthly)
![Sales Performance](Screenshots/Sales_Performance_by_Region___Property_Type__Monthly_.png)

### B3 — Property Inventory & Pricing Trends
![Property Inventory](Screenshots/Property_Inventory___Pricing_Trends.png)

### B4 — Campaign ROI & Lead Conversion

![Campaign ROI](Screenshots/Campaign_ROI___Lead_Conversion.png)

| Channel | Campaigns | Total Leads | Converted Sales | ROI % |
|---------|-----------|-------------|-----------------|-------|
| 📱 Social Ads | 348 | 1,337 | 245 | **5,753%** |
| 🌐 Portal Featured | 719 | 2,856 | 473 | 4,560% |
| 📋 Offline | 74 | 317 | 53 | 4,333% |
| 📧 Email Blast | 304 | 1,260 | 220 | 4,202% |
| 🔍 Search Ads | 55 | 238 | 39 | 2,952% |

### B5 — Agent Productivity Dashboard
![Agent Dashboard](Screenshots/Agent_Productivity_Dashboard.png)

---

## 📋 Pipeline Log

![Pipeline Log](Screenshots/Pipeline_log___recent_runs.png)

All recent runs completed with **zero errors**:

| Pipeline | Source | Target | Status | Start Time |
|----------|--------|--------|--------|------------|
| Pl_Transforma_StagingTodwh | stg.AllTables | dwh.Facts | ✅ Success | 2026-05-22 08:08 |
| pl_IngestRawToString | Blob_RawFiles | stgAllTables | ✅ Success | 2026-05-22 08:07 |
| Pl_Transforma_StagingTodwh | stg.AllTables | dwh.Facts | ✅ Success | 2026-05-22 05:47 |
| pl_IngestRawToString | Blob_RawFiles | stgAllTables | ✅ Success | 2026-05-21 10:46 |

---

## 📁 Repository Structure

```
📦 ADF_RealEstate_Data_Engineering_Project
 ┣ 📂 pipeline/
 ┃  ┣ 📄 PL_Master_RealEstate_ETL.json
 ┃  ┣ 📄 pl_IngestRawToString.json
 ┃  └ 📄 Pl_Transforma_StagingTodwh.json
 ┣ 📂 StoredProcedures/
 ┃  ┣ 📄 Sp_LoadDimDate.sql
 ┃  ┣ 📄 Sp_LoadDimAgents.sql
 ┃  ┣ 📄 Sp_LoadDimCampaigns.sql
 ┃  ┣ 📄 Sp_LoadDimListings.sql
 ┃  ┣ 📄 Sp_LoadDimProperties.sql
 ┃  ┣ 📄 Sp_LoadFactSales.sql
 ┃  ┣ 📄 Sp_LoadFactoffers.sql
 ┃  ┣ 📄 Sp_LoadFactLeads.sql
 ┃  ┣ 📄 Sp_LogSuccess.sql
 ┃  └ 📄 Sp_LogFailure.sql
 ┣ 📂 Datasets/
 ┃  ┣ 📂 Blob_csv/          ← 8 CSV source datasets
 ┃  └ 📂 SQL_Datasets/      ← 8 SQL staging datasets
 ┣ 📂 Screenshots/          ← Pipeline & query evidence
 ┣ 📄 SQLQuery1.sql         ← All audit + analytical queries
 ┗ 📄 README.md
```

---

## 🛠️ Technology Stack

| Component | Technology |
|-----------|------------|
| Orchestration | Azure Data Factory (ADF) |
| Raw Storage | Azure Blob Storage (CSV) |
| Database | Azure SQL Database |
| Transformation | SQL Stored Procedures |
| Monitoring | `stg.PipelineLog` table |
| Schema | Star Schema (Kimball) |
| BI Layer | Power BI (query-ready) |

---

## 🚀 How to Run

1. **Clone the repo**
   ```bash
   git clone https://github.com/001MEET/ADF_Realrealestate_Deta_Engineering-_Project.git
   ```

2. **Setup Azure SQL Database**
   - Create database `db_realestate_dwh`
   - Run schema scripts to create `stg.*` and `dwh.*` tables
   - Deploy all Stored Procedures from `/StoredProcedures/`

3. **Setup Azure Blob Storage**
   - Upload CSV source files to `Blob_csv` container
   - Update linked service connection strings in ADF

4. **Import ADF Pipelines**
   - Import JSON files from `/pipeline/` into your Azure Data Factory
   - Update dataset linked services to point to your resources

5. **Run the Master Pipeline**
   - Trigger `PL_Master_RealEstate_ETL` in ADF
   - Monitor via `stg.PipelineLog` table

---

## 👤 Author

**Meetkumar Kalpeshkumar Patel**
rishabh Software 
📧 meetkpatel04@gmail.com

---

## 📄 License

This project is for academic and portfolio purposes.

---

> ⭐ If you found this project helpful, please give it a star!
