# Netflix Data Pipeline — Azure End-to-End Data Engineering Project

An automated pipeline that ingests raw Netflix catalog data from GitHub, processes it through a Bronze → Silver → Gold architecture on Azure Databricks, and delivers analytics-ready star schema tables with built-in data quality checks. Built to mirror how a real production data platform is structured, not just a single notebook running transformations.

![Architecture Diagram](docs/architecture-diagram.png)

## What This Solves

Raw data sitting in scattered CSV files isn't usable for reporting or analysis. This pipeline automates the journey from source files to trustworthy, query-ready tables — with incremental loading so it only processes new data, and data quality rules that catch bad records before they reach the reporting layer.

## Architecture

| Layer | Tool | What Happens |
|---|---|---|
| **Ingestion** | Azure Data Factory | Pulls Netflix dataset files from GitHub and lands them in the Raw container |
| **Bronze** | Databricks Auto Loader | Incrementally loads new raw files into Bronze as Delta tables |
| **Silver** | Databricks (PySpark) | Cleans, casts, and transforms data; builds dimension tables |
| **Gold** | Delta Live Tables | Applies data quality rules and produces curated fact/dimension tables |
| **Orchestration** | Databricks Workflows | Runs all Bronze → Gold notebooks as a scheduled job with task dependencies |
| **Serving** | Azure Synapse, Power BI | *(confirm current status before publishing — update this row)* |

## Pipeline Walkthrough

### 1. Ingestion — GitHub to Raw (Azure Data Factory)
The `pipeline_netflix` pipeline calls the GitHub API to fetch file metadata, then runs a Validation activity that checks `netflix_titles.csv` actually exists in the Raw container before proceeding — a guardrail against running the copy loop against a missing or not-yet-landed source file. Once validated, it loops through a parameterized array of source folders (`netflix_titles`, `netflix_directors`, `netflix_cast`, `netflix_countries`, `netflix_category`) and copies each one into the Raw container in ADLS Gen2. Adding a new source folder is a config change, not a pipeline rebuild.

### 2. Bronze — Incremental Loading (Auto Loader)
`1_Autoloader.ipynb` uses Databricks Auto Loader (`cloudFiles`) to pick up new files from Raw and stream them into Bronze as Delta tables, with schema location tracking so schema changes are handled automatically rather than breaking the pipeline.

### 3. Silver — Cleaning and Transformation
Two notebooks handle this layer:
- `2_Silver.ipynb` is a reusable, parameterized notebook that moves each lookup/dimension table (directors, cast, countries, category) from Bronze to Silver as Delta. The list of tables to process is generated dynamically by `3_LookupNotebook.ipynb`.
- `4_Silver_dataTransformation.ipynb` handles the core `netflix_titles` table: null handling, type casting, splitting compound fields (title, rating), flagging content type (Movie vs. TV Show), and ranking by duration.

A weekday parameter (`5_LookupNotebook.ipynb`) drives conditional logic in the Databricks Workflow, controlling which activities run on which days.

### 4. Gold — Star Schema with Data Quality (Delta Live Tables)
`7_DLT_Notebook.ipynb` builds the Gold layer using Delta Live Tables, streaming each Silver table forward and enforcing quality rules with `expect_all_or_drop` (for example, rejecting any record with a null `show_id`). The final `gold_netflixtitles` table goes through a staged transform before landing, with an additional quality check on the derived flag column. This produces five curated tables (`gold_netflixtitles`, `gold_netflixdirectors`, `gold_netflixcast`, `gold_netflixcountries`, `gold_netflixcategory`) — a fact/dimension-shaped Gold layer, though the tables aren't yet linked with formal keys, that step would happen at the modeling stage (e.g. in Synapse) to turn this into a true star schema.

### 5. Orchestration
All Databricks notebooks run as tasks inside a single Databricks Workflow, with task dependencies controlling execution order — not manually triggered notebook-by-notebook.

## Repo Structure

```
├── adf/                  # Exported ADF pipeline JSON
├── databricks/
│   ├── 1_Autoloader.ipynb
│   ├── 2_Silver.ipynb
│   ├── 3_LookupNotebook.ipynb
│   ├── 4_Silver_dataTransformation.ipynb
│   ├── 5_LookupNotebook.ipynb
│   └── 7_DLT_Notebook.ipynb
├── docs/
│   └── architecture-diagram.png
└── README.md
```

## Tech Stack
Azure Data Factory · Azure Databricks · Delta Lake · Delta Live Tables · PySpark · Azure Data Lake Storage Gen2 · Databricks Workflows

## What I'd Improve Next
- Add monitoring/alerting on pipeline failures
- Extend DLT expectations to cover more fields beyond null checks
- Wire up the serving layer (Synapse/Power BI) for end-to-end reporting

---
*This is my first end-to-end Data Engineering project, built to apply Medallion Architecture, incremental ingestion, and data quality patterns in a real Azure environment.*
