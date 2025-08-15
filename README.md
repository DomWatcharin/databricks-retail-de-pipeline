# databricks-retail-de-pipeline

# Databricks Retail Data Engineering Project (Serverless Workspace Compatible)

This project demonstrates an **end-to-end Retail Data Engineering pipeline** on Databricks using **PySpark + Delta Lake + Medallion architecture (Bronze / Silver / Gold)**.

It is **fully compatible with serverless Databricks workspaces**, meaning it does not define custom clusters in the Jobs API and relies on **notebook-scoped libraries** for any dependencies.

---

## 📂 Project Structure

| Notebook Name | Stage | Purpose |
|---------------|-------|---------|
| `00_config` | Config | Sets catalog, schema, volume paths |
| `01_generate_sample_data` | Data Gen | Generates synthetic raw data for demo |
| `10_bronze_batch_dims` | Bronze | Batch load of dimension CSVs (customers, products, stores) |
| `11_bronze_stream_sales_autoloader` | Bronze | Streaming ingest of POS sales data via Auto Loader |
| `20_silver_dimensions` | Silver | Clean + deduplicate customer & store dims |
| `21_silver_products_scd2` | Silver | Maintain product dimension as SCD Type 2 |
| `22_silver_fact_sales` | Silver | Build sales fact table |
| `30_gold_marts` | Gold | Build gold-layer BI marts |
| `40_quality_checks` | Gold | Simple data quality assertions |

---

## 🗂 Unity Catalog Setup

Run in **Databricks SQL Editor** (adjust names if needed):

```sql
CREATE CATALOG IF NOT EXISTS retail;
CREATE SCHEMA  IF NOT EXISTS retail.core;

CREATE VOLUME IF NOT EXISTS retail.core.retail_vol
COMMENT 'Storage for retail DE demo';
```

**Recommended volume directory layout:**
```
/raw/customers
/raw/products
/raw/stores
/raw/pos_sales
/_schemas/pos_sales
/_checkpoints/bronze/pos_sales
/bronze
/silver
/gold
```

---

## 📓 Notebook Order

Run these notebooks in order:

1. `00_config`
2. `01_generate_sample_data`
3. `10_bronze_batch_dims`
4. `11_bronze_stream_sales_autoloader`
5. `20_silver_dimensions`
6. `21_silver_products_scd2`
7. `22_silver_fact_sales`
8. `30_gold_marts`
9. `40_quality_checks`

---

## 🚀 Creating the Job in a Serverless Workspace

### 1. Create `job_serverless_min.json` locally
```json
{
  "name": "Retail-DE-Pipeline",
  "tasks": [
    {
      "task_key": "bronze_dims",
      "notebook_task": {
        "notebook_path": "/Users/<your-email>/10_bronze_batch_dims"
      }
    },
    {
      "task_key": "bronze_sales",
      "depends_on": [{"task_key": "bronze_dims"}],
      "notebook_task": {
        "notebook_path": "/Users/<your-email>/11_bronze_stream_sales_autoloader"
      }
    },
    {
      "task_key": "silver_dims",
      "depends_on": [{"task_key": "bronze_sales"}],
      "notebook_task": {
        "notebook_path": "/Users/<your-email>/20_silver_dimensions"
      }
    },
    {
      "task_key": "silver_products_scd2",
      "depends_on": [{"task_key": "silver_dims"}],
      "notebook_task": {
        "notebook_path": "/Users/<your-email>/21_silver_products_scd2"
      }
    },
    {
      "task_key": "silver_fact",
      "depends_on": [{"task_key": "silver_products_scd2"}],
      "notebook_task": {
        "notebook_path": "/Users/<your-email>/22_silver_fact_sales"
      }
    },
    {
      "task_key": "gold_marts",
      "depends_on": [{"task_key": "silver_fact"}],
      "notebook_task": {
        "notebook_path": "/Users/<your-email>/30_gold_marts"
      }
    },
    {
      "task_key": "dq_checks",
      "depends_on": [{"task_key": "gold_marts"}],
      "notebook_task": {
        "notebook_path": "/Users/<your-email>/40_quality_checks"
      }
    }
  ],
  "schedule": {
    "quartz_cron_expression": "0 15 2 * * ?",
    "timezone_id": "Asia/Bangkok",
    "pause_status": "UNPAUSED"
  }
}
```

Replace `<your-email>` with your actual Databricks user email.

---

### 2. Create the Job via CLI
- **Legacy CLI (0.17.x)**:
```bash
databricks jobs create --json-file job_serverless_min.json
```
- **New CLI**:
```bash
databricks jobs create --json @job_serverless_min.json
```

---

### 3. Run the Job
```bash
databricks jobs run-now --job-id <JOB_ID>
```

---

## 🐍 Installing Extra Libraries in Serverless
Use notebook-scoped installs at the top of your notebooks:
```python
%pip install pandas==2.2.2
dbutils.library.restartPython()
```

---

## 📊 Querying Gold Tables in Databricks SQL
```sql
SELECT event_date, sku, name, SUM(revenue) AS revenue
FROM retail.core.gm_daily_store_product_sales
WHERE event_date = date_sub(current_date(), 1)
GROUP BY event_date, sku, name
ORDER BY revenue DESC
LIMIT 10;
```

## Querying Yesterday’s top 10 products by revenue in Databricks SQL
```sql
SELECT event_date, sku, name, SUM(revenue) AS revenue
FROM retail.core.gm_daily_store_product_sales
WHERE event_date = date_sub(current_date(), 1)
GROUP BY event_date, sku, name
ORDER BY revenue DESC
LIMIT 10;
```

## Querying High-value customers (LTV > 10,000) in Databricks SQL
```sql
SELECT customer_id, first_name, last_name, lifetime_value
FROM retail.core.gm_customer_360
WHERE lifetime_value > 10000
ORDER BY lifetime_value DESC;
```

---

## 📌 License
MIT
