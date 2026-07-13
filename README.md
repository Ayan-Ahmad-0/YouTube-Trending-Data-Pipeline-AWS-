# YouTube Trending Data Pipeline

An end-to-end, serverless data pipeline on AWS that ingests YouTube trending video data across multiple regions, cleanses and validates it, and produces analytics-ready aggregated tables using a **Bronze → Silver → Gold (medallion) architecture**.

![Architecture Diagram](YouTube%20Trending%20Data%20Pipeline/images/youtube_pipeline_architecture.png)
## 📋 Overview

This pipeline automatically:
1. Pulls trending video statistics and category metadata from the **YouTube Data API** across multiple regions (US, GB, CA, IN)
2. Lands raw data in a **Bronze** S3 layer
3. Cleanses, deduplicates, and transforms it into a **Silver** layer (Parquet, partitioned by region)
4. Runs automated **data quality checks** before promoting data further
5. Aggregates cleansed data into **Gold**-layer analytics tables (trending, channel, and category-level insights)
6. Sends **success/failure notifications** via SNS at each stage
7. Is fully orchestrated end-to-end by **AWS Step Functions**

---

## 🏗️ Architecture

**Data Sources** → **Bronze (raw)** → **Silver (cleansed)** → **Data Quality Gate** → **Gold (aggregated)** → **Analytics/Consumption**

| Layer | Purpose | Format | Storage |
|---|---|---|---|
| **Bronze** | Raw, untouched ingestion | JSON | S3 |
| **Silver** | Cleansed, validated, deduplicated | Parquet | S3 (partitioned by region) |
| **Gold** | Business-level aggregations | Parquet | S3 (partitioned by region) |

### Tech Stack

- **Ingestion:** AWS Lambda + YouTube Data API, triggered on a schedule via Amazon EventBridge
- **Storage:** Amazon S3 (Bronze / Silver / Gold buckets)
- **Transformation:**
  - AWS Lambda (`awswrangler`) for reference/category data → Silver
  - AWS Glue (PySpark) for statistics data → Silver, and Silver → Gold aggregations
- **Cataloging:** AWS Glue Data Catalog (per-layer databases)
- **Querying:** Amazon Athena
- **Data Quality:** AWS Lambda running automated validation checks (row counts, null %, schema, value ranges, freshness) before Gold aggregation is allowed to proceed
- **Orchestration:** AWS Step Functions (parallel branches, retries, error handling)
- **Alerting/Monitoring:** Amazon SNS (success/failure notifications), Amazon CloudWatch (logging)
- **IAM:** Dedicated least-privilege roles per Lambda function and Glue job

---

## 🔄 Pipeline Flow (Step Functions)

```
IngestFromYouTubeAPI
        │
        ▼
WaitForS3Consistency
        │
        ▼
┌───────────────────────────────┐
│      ProcessInParallel        │
│  ┌─────────────┐ ┌──────────┐ │
│  │ Transform    │ │ Bronze → │ │
│  │ Reference    │ │ Silver   │ │
│  │ Data (Lambda)│ │ (Glue)   │ │
│  └─────────────┘ └──────────┘ │
└───────────────────────────────┘
        │
        ▼
RunDataQualityChecks (Lambda)
        │
        ▼
EvaluateDataQuality ──(fail)──► NotifyDQFailure
        │ (pass)
        ▼
RunSilverToGoldGlueJob (Glue)
        │
        ▼
NotifySuccess (SNS)
```

Every task step includes **retry logic** (exponential backoff) and **catch blocks** that route failures to dedicated SNS notification states — so any breakage at ingestion, transformation, data quality, or aggregation triggers an immediate alert rather than failing silently.

---

## 🗂️ Gold Layer Tables

| Table | Description |
|---|---|
| `trending_analytics` | Daily trending video summaries per region (total views, likes, engagement rate, unique channels/categories) |
| `channel_analytics` | Channel-level performance metrics, ranked by total views within each region |
| `category_analytics` | Category-level trends over time, including each category's share of total views per region/day |

---

## ✅ Data Quality Checks

Before Silver data is promoted to Gold, an automated quality gate validates:

- **Row count** — minimum threshold met
- **Null percentage** — critical columns (`video_id`, `title`, `channel_title`, `views`, `region`) stay under a configurable null threshold
- **Schema validation** — all expected columns are present
- **Value range checks** — no negative or implausibly extreme view counts
- **Freshness** — data is recent enough to be considered valid for the current run

If any check fails, the pipeline halts before Gold aggregation and sends a detailed failure report via SNS.

---

## 📁 Repository Structure

```
youtube-data-pipeline/
├── README.md
├── architecture-diagram.png
├── lambda/
│   ├── ingestion/                  # Pulls data from YouTube Data API → Bronze
│   ├── json_to_parquet/            # Reference/category data → Silver (Parquet)
│   └── data_quality/               # Automated DQ checks on Silver layer
├── glue_jobs/
│   ├── bronze_to_silver.py         # Statistics ETL: Bronze → Silver
│   └── silver_to_gold.py           # Aggregations: Silver → Gold
├── step_functions/
│   └── state_machine.json          # Full pipeline orchestration definition
└── iam_policies/
    ├── lambda_role_policy.json
    ├── glue_role_policy.json
    └── step_functions_role_policy.json
```

---

## ⚙️ Configuration

The pipeline is parameterized via environment variables (Lambda) and job parameters (Glue), so no credentials or resource names are hardcoded in application logic.

**Example environment variables (Lambda):**
```
S3_BUCKET_BRONZE
S3_BUCKET_SILVER
GLUE_DB_SILVER
SNS_ALERT_TOPIC_ARN
YOUTUBE_API_KEY
YOUTUBE_REGIONS
```

**Example job parameters (Glue):**
```
--bronze_database
--bronze_table
--silver_database
--silver_table
--silver_bucket
--gold_database
--gold_bucket
```

---

## 🔐 IAM & Security Notes

Each component runs under a dedicated, least-privilege IAM role:
- **Lambda execution role** — scoped S3 read/write on relevant buckets, Glue Catalog access, SNS publish, Athena query execution
- **Glue job role** — scoped S3 access on Bronze (read) and Silver/Gold (read/write), Glue Catalog access
- **Step Functions execution role** — scoped to invoke only this pipeline's specific Lambda functions and Glue jobs, plus SNS publish

---

## 🚧 Engineering Challenges Solved

A few notable issues worked through during development:
- Diagnosed and resolved multiple IAM permission gaps across Lambda, Glue, and Step Functions roles (Glue Catalog access, S3 read/write, Athena query execution)
- Built a working S3 event-driven trigger to automatically kick off Silver-layer transformation on new object creation
- Designed a parallel-branch Step Functions workflow with independent retry/catch handling per branch
- Implemented an automated data quality gate that blocks downstream aggregation on failed validation, with SNS alerting
- Resolved subtle configuration bugs (bucket name casing mismatches, whitespace in job parameters) surfaced through systematic CloudWatch log analysis

---

## 📊 Sample Output

*(Add a screenshot here of your S3 partitioned folder structure, an Athena query result, or the Step Functions execution graph showing a successful run.)*

---

## 🛠️ Future Improvements

- Add a QuickSight dashboard on top of the Gold layer for visual analytics
- Expand region coverage
- Add automated testing for Lambda functions and Glue scripts
- Tighten IAM policies further (currently some use wildcard resources for Athena actions where AWS doesn't support resource-level scoping)

---


