# Databricks-AML-Datapipeline

An end-to-end batch data pipeline built on Databricks that ingests the IBM Transactions for Anti Money Laundering (AML) dataset and produces daily transaction metrics by payment currency, using a Bronze → Silver → Gold medallion architecture.

> **Note:** The dataset is synthetic and designed for AML research. The pipeline reports transactions *labelled* as laundering in the source data. It does not detect confirmed financial crime.

## Architecture

```
CSV files → Unity Catalog Volume → Bronze (Delta) → Silver (Delta) → Gold (Delta)
                     └──────── Databricks Workflow: validation, logging, retries ────────┘
```

## Tech Stack

- Databricks (Free Edition), Databricks Workflows
- PySpark
- Delta Lake
- Unity Catalog (catalog, schema, Volume)

## Dataset

- **Source:** [IBM Transactions for Anti Money Laundering (AML)](https://www.kaggle.com/datasets/ealtman2019/ibm-transactions-for-anti-money-laundering-aml) on Kaggle (`HI-Small` variant)
- **Files used:** `HI-Small_Trans.csv`, `HI-Small_accounts.csv`, `HI-Small_Patterns.txt`
- **Transactions:** 5,078,345 rows, 11 columns, September 1–18, 2022
- The raw data is not included in this repository. See [Setup](#setup-and-execution) to obtain it.

## Pipeline Layers

| Layer | Table | Description |
|---|---|---|
| Bronze | `bronze_aml_transactions` | Raw transactions read from the Volume with an explicit schema and snake_case column names. No cleaning applied. |
| Silver | `silver_aml_transactions` | Parsed timestamps (`yyyy/MM/dd HH:mm`), amounts cast to `DECIMAL(22,6)`, exact duplicates removed. |
| Gold | `gold_daily_transaction_metrics` | Daily metrics per `transaction_date` and `payment_currency`. |

All tables are in `aml_transactions.aml_transactions_schema`.

**Gold grain:** one row per `transaction_date` + `payment_currency`.

**Gold columns:** `transaction_date`, `payment_currency`, `tot_transaction_count`, `total_amount_paid`, `laundering_count`, `non_laundering_count`, `net_laundering_amount`, `laundering_labelled_transaction_pct`

## Data Quality and Validation

**Silver checks**
- Null check across all transaction columns
- Timestamp parse validation and date range check
- Zero or negative amount check
- Laundering label limited to `0` and `1`
- Exact duplicate removal

**Gold checks**
- Uniqueness of the date-currency grain
- Count totals reconcile with Silver

Validation checks run before each layer. *(List the specific checks implemented, e.g. input file existence.)*

## Workflow, Failure Handling, and Idempotency

- Separate notebooks per layer, orchestrated as a Databricks Workflow: Bronze → Silver → Gold.
- Downstream tasks run only if the upstream task succeeds. A failed validation fails the task and stops the pipeline.
- Retries are configured for transient failures only. Data-quality and schema failures are deterministic and fail immediately.
- Writes use overwrite mode. Reruns on the same input produce identical results with no duplicates.

## Results

| Metric | Value |
|---|---|
| Source rows | 5,078,345 |
| Exact duplicate excess rows removed | 9 |
| Silver rows | 5,078,336 |
| Laundering-labelled transactions | 5,177 |
| Non-laundering-labelled transactions | 5,073,159 |
| Gold rows | 230 |

Gold counts reconcile with Silver. Silver has no nulls, invalid timestamps, non-positive amounts, or unexpected labels.

## Repository Structure

```
├── README.md
├── notebooks/
│   ├── 01_bronze_ingestion
│   ├── 02_silver_transformation
│   └── 03_gold_daily_metrics
├── docs/
│   └── architecture.png
└── .gitignore
```