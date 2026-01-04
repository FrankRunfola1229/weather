# Weather — Ultra-Cheap Azure Data Pipeline (ADF + ADLS Gen2 + Databricks, No SQL)

A tiny, portfolio-ready **Azure-only** data engineering project that ingests **free public weather data** from **Open-Meteo** (no API key), lands it in **ADLS Gen2**, and transforms it using **Databricks notebooks** following **Medallion architecture (Bronze/Silver/Gold)**. Orchestrated end-to-end with **Azure Data Factory**.

**Goal:** Build a clean, interview-friendly project you can explain in 60 seconds — without paying for SQL databases or expensive services.

---

## 1) What this project does (in plain English)

Each day, the pipeline:
1. Calls Open-Meteo’s **Historical Weather API** for a small list of cities (hourly weather data).
2. Saves the raw JSON responses to **ADLS Bronze**.
3. Runs Databricks notebooks to:
   - **Silver:** Flatten and clean the hourly data (one row per city-hour).
   - **Gold:** Create daily city KPIs (min/max/avg temp, precipitation totals, wind stats, degree-hour metrics).
4. Stores outputs as **Delta files** in ADLS (still “no SQL database”).

---

## 2) Architecture

### Services used
- **Azure Data Lake Storage Gen2 (ADLS):** storage for Bronze/Silver/Gold layers
- **Azure Data Factory (ADF):** orchestration (API → ADLS + trigger Databricks)
- **Azure Databricks:** transformations and Delta outputs
- **Managed Identities:** storage access without storing secrets (recommended)

### Data flow (high level)
1. ADF runs daily
2. ADF calls Open-Meteo `/v1/archive` for each city
3. ADF writes raw JSON to ADLS **Bronze**
4. ADF triggers Databricks notebooks:
   - Bronze → Silver
   - Silver → Gold
5. Results stored as **Delta files** in ADLS

---

## 3) Keep it ultra cheap (do this or you’ll burn money)

**Hard rules**
- Use **5 cities max**
- Process **1 day per run**
- Databricks:
  - **Single-node cluster**
  - **Auto-terminate after 10–15 minutes**
  - Run as a **Job** (don’t leave interactive clusters running)
- ADF:
  - One pipeline + one trigger
  - Minimal activities (Lookup, ForEach, Copy, Databricks)

---

## 4) Repo structure

```
weather-medallion01/
  config/
    cities.json
  notebooks/
    01_bronze_ingest.py
    02_silver_clean_hourly.py
    03_gold_city_daily_kpis.py
  adf/
    pipeline_steps.md
  README.md
```

---

## 5) Medallion layers and storage layout

### Bronze (raw)
- Store the raw API payload as JSON files
   - `abfss://bronze@<storage>.dfs.core.windows.net/open_meteo/archive/dt=YYYY-MM-DD/city=<city>/payload.json`

- Also write a Bronze Delta dataset (append-only)
   - `abfss://bronze@<storage>.dfs.core.windows.net/delta/open_meteo_hourly_bronze/`

### Silver (clean/flat)
- One row per **city + hour**
- Flatten nested JSON arrays into tabular columns
- Enforce data types and minimal quality rules
  - `abfss://silver@<storage>.dfs.core.windows.net/delta/open_meteo_hourly_silver/`

### Gold (analytics-ready)
- One row per **city + date**
- Daily KPIs and derived metrics
  - `abfss://gold@<storage>.dfs.core.windows.net/delta/open_meteo_city_daily_gold/`

---

## 6) Data source (Open-Meteo)

### Historical Weather API
- Endpoint path: `/v1/archive`
- Host (commonly used): `https://archive-api.open-meteo.com`
- No API key needed for typical usage.

### Example request (1 city, 1 day)
```
https://archive-api.open-meteo.com/v1/archive
  ?latitude=35.2271
  &longitude=-80.8431
  &start_date=2025-12-25
  &end_date=2025-12-25
  &hourly=temperature_2m,relative_humidity_2m,precipitation,wind_speed_10m
  &timezone=auto
```

**Practical note:** historical data can lag a bit. To avoid “not ready” issues, run for `utcNow() - 5 days`.

---

## 7) Azure setup (step-by-step)

### 7.1 Create resources
1. Create Resource Group: `rg-weather-medallion01`
2. Create Storage Account (ADLS Gen2):
   - Performance: **Standard**
   - Enable **Hierarchical namespace** (required for Gen2)
3. Create containers:
   - `bronze`
   - `silver`
   - `gold`
   - `config`
4. Create:
   - **Azure Data Factory**
   - **Azure Databricks workspace**

---

## 8) Upload city config to ADLS

Upload `config/cities.json` to:

`abfss://config@<storage>.dfs.core.windows.net/weather-medallion01/cities.json`

### `config/cities.json`
```json
[
  {"city":"CharlotteNC","latitude":35.2271,"longitude":-80.8431},
  {"city":"NewYorkNY","latitude":40.7128,"longitude":-74.0060},
  {"city":"ChicagoIL","latitude":41.8781,"longitude":-87.6298},
  {"city":"SeattleWA","latitude":47.6062,"longitude":-122.3321},
  {"city":"MiamiFL","latitude":25.7617,"longitude":-80.1918}
]
```

---

## 9) Security (no storage keys)

### 9.1 ADF → ADLS (Managed Identity)
1. In ADF, enable **System Assigned Managed Identity**
2. In Storage Account → **Access Control (IAM)**:
   - Grant ADF MI: **Storage Blob Data Contributor**
3. In ADF, create an ADLS Gen2 linked service using **Managed Identity**

### 9.2 Databricks → ADLS (Managed Identity)
Use an Azure-managed identity approach (workspace/compute MI or access connector depending on workspace settings):
1. Grant Databricks MI: **Storage Blob Data Contributor**
2. Use direct ABFSS paths (no mounting required)

---

## 10) Build the ADF pipeline

### 10.1 Linked Services
- `LS_ADLS` (ADLS Gen2, Managed Identity)
- `LS_OpenMeteo_Rest` (REST, anonymous)
- `LS_Databricks` (Managed Identity if supported in your workspace; otherwise PAT token fallback)

### 10.2 Datasets
- `DS_CitiesConfig` (ADLS JSON file)
- `DS_OpenMeteo_Rest` (REST dataset)
- `DS_BronzeJsonSink` (ADLS file sink)

### 10.3 Pipeline: `pl_openmeteo_to_medallion`

#### Pipeline parameters
- `p_process_date` (string)
  - default: `@formatDateTime(addDays(utcNow(), -5),'yyyy-MM-dd')`
- `p_hourly_vars` (string)
  - default: `temperature_2m,relative_humidity_2m,precipitation,wind_speed_10m`

#### Activities (in order)

**1) Lookup_Cities**
- Source: `cities.json` in ADLS config container
- Output: array of cities

**2) ForEach_City** (items = cities array)
- **Copy_OpenMeteo_To_Bronze**
  - Source: REST
  - Sink: ADLS (single file)
  - Sink path:
    - `bronze/open_meteo/archive/dt=@{pipeline().parameters.p_process_date}/city=@{item().city}/payload.json`
  - REST base URL:
    - `https://archive-api.open-meteo.com`
  - Relative path:
    - `/v1/archive`
  - Query parameters:
    - `latitude=@{item().latitude}`
    - `longitude=@{item().longitude}`
    - `start_date=@{pipeline().parameters.p_process_date}`
    - `end_date=@{pipeline().parameters.p_process_date}`
    - `hourly=@{pipeline().parameters.p_hourly_vars}`
    - `timezone=auto`

**3) Databricks activity (Job/Notebook)**
Run these notebooks in sequence, passing `process_date`:
- `01_bronze_ingest.py`
- `02_silver_clean_hourly.py`
- `03_gold_city_daily_kpis.py`

---

## 11) Databricks notebooks

> Replace `<YOUR_STORAGE_ACCOUNT_NAME>` with your storage account name.

### 11.1 `notebooks/01_bronze_ingest.py`
Reads raw JSON payloads and writes Bronze Delta.

```python
from pyspark.sql import functions as F

dbutils.widgets.text("process_date", "")
process_date = dbutils.widgets.get("process_date")

storage = "<YOUR_STORAGE_ACCOUNT_NAME>"
bronze_raw_path = f"abfss://bronze@{storage}.dfs.core.windows.net/open_meteo/archive/dt={process_date}/*/payload.json"
bronze_delta_path = f"abfss://bronze@{storage}.dfs.core.windows.net/delta/open_meteo_hourly_bronze/"

raw = (
    spark.read
         .option("multiLine", "true")
         .json(bronze_raw_path)
         .withColumn("_ingest_ts", F.current_timestamp())
         .withColumn("_process_date", F.lit(process_date))
)

(raw.write
    .format("delta")
    .mode("append")
    .partitionBy("_process_date")
    .save(bronze_delta_path)
)

display(raw.limit(5))
```

---

### 11.2 `notebooks/02_silver_clean_hourly.py`
Flattens hourly arrays into one row per hour with types and basic checks.

```python
from pyspark.sql import functions as F

dbutils.widgets.text("process_date", "")
process_date = dbutils.widgets.get("process_date")

storage = "<YOUR_STORAGE_ACCOUNT_NAME>"
bronze_delta_path = f"abfss://bronze@{storage}.dfs.core.windows.net/delta/open_meteo_hourly_bronze/"
silver_delta_path = f"abfss://silver@{storage}.dfs.core.windows.net/delta/open_meteo_hourly_silver/"

bronze = spark.read.format("delta").load(bronze_delta_path).filter(F.col("_process_date") == process_date)

zipped = (
    bronze
    .select(
        F.col("latitude"), F.col("longitude"),
        F.col("timezone"),
        F.col("_ingest_ts"), F.col("_process_date"),
        F.arrays_zip(
            F.col("hourly.time").alias("time"),
            F.col("hourly.temperature_2m").alias("temperature_2m"),
            F.col("hourly.relative_humidity_2m").alias("relative_humidity_2m"),
            F.col("hourly.precipitation").alias("precipitation"),
            F.col("hourly.wind_speed_10m").alias("wind_speed_10m"),
        ).alias("z")
    )
    .withColumn("z", F.explode("z"))
    .select(
        "latitude","longitude","timezone","_ingest_ts","_process_date",
        F.col("z.time").alias("time"),
        F.col("z.temperature_2m").cast("double").alias("temperature_2m"),
        F.col("z.relative_humidity_2m").cast("double").alias("relative_humidity_2m"),
        F.col("z.precipitation").cast("double").alias("precipitation"),
        F.col("z.wind_speed_10m").cast("double").alias("wind_speed_10m"),
    )
)

silver = (
    zipped
    .withColumn("event_ts", F.to_timestamp("time"))
    .drop("time")
    .withColumn("event_date", F.to_date("event_ts"))
    .filter(F.col("temperature_2m").isNotNull())
)

(silver.write
    .format("delta")
    .mode("append")
    .partitionBy("event_date")
    .save(silver_delta_path)
)

display(silver.limit(10))
```

---

### 11.3 `notebooks/03_gold_city_daily_kpis.py`
Builds daily aggregates + degree-hour metrics.

```python
from pyspark.sql import functions as F

dbutils.widgets.text("process_date", "")
process_date = dbutils.widgets.get("process_date")

storage = "<YOUR_STORAGE_ACCOUNT_NAME>"
silver_delta_path = f"abfss://silver@{storage}.dfs.core.windows.net/delta/open_meteo_hourly_silver/"
gold_delta_path = f"abfss://gold@{storage}.dfs.core.windows.net/delta/open_meteo_city_daily_gold/"

silver = spark.read.format("delta").load(silver_delta_path).filter(F.col("event_date") == F.lit(process_date))

base_temp_c = 18.0

enriched = (
    silver
    .withColumn("cdh", F.when(F.col("temperature_2m") > base_temp_c, F.col("temperature_2m") - base_temp_c).otherwise(F.lit(0.0)))
    .withColumn("hdh", F.when(F.col("temperature_2m") < base_temp_c, F.lit(base_temp_c) - F.col("temperature_2m")).otherwise(F.lit(0.0)))
)

daily = (
    enriched.groupBy("event_date","timezone","latitude","longitude")
    .agg(
        F.min("temperature_2m").alias("temp_min_c"),
        F.max("temperature_2m").alias("temp_max_c"),
        F.avg("temperature_2m").alias("temp_avg_c"),
        F.sum("precipitation").alias("precip_total_mm"),
        F.avg("wind_speed_10m").alias("wind_avg"),
        F.sum("cdh").alias("cooling_degree_hours"),
        F.sum("hdh").alias("heating_degree_hours"),
        F.count("*").alias("hour_rows")
    )
    .withColumn("_build_ts", F.current_timestamp())
)

(daily.write
    .format("delta")
    .mode("append")
    .partitionBy("event_date")
    .save(gold_delta_path)
)

display(daily.orderBy("event_date").limit(20))
```

---

## 12) Run it end-to-end

1. Create Azure resources (RG, ADLS Gen2, ADF, Databricks)
2. Upload `cities.json` to ADLS
3. Build ADF pipeline and run manually once
4. Confirm Bronze JSON files landed in:
   - `bronze/open_meteo/archive/dt=YYYY-MM-DD/city=.../payload.json`
5. Confirm Delta outputs exist:
   - `bronze/delta/open_meteo_hourly_bronze/`
   - `silver/delta/open_meteo_hourly_silver/`
   - `gold/delta/open_meteo_city_daily_gold/`
6. Add a daily trigger (optional) — keep it daily, not hourly

---

## 13) Key Functionality
  - Built an **Azure-only** ingestion and transformation pipeline using **ADF, ADLS Gen2, Databricks**
  - Implemented **Medallion architecture (Bronze/Silver/Gold)** with **Delta** storage on ADLS
  - Orchestrated REST API ingestion and notebook execution with parameterized partitioning and repeatable runs
  - Flattened nested JSON into curated datasets and produced **daily KPI aggregates** for analytics consumption

---

## 14) Easy upgrades (if you want to level up)
  - Add lightweight data quality checks (null rates, bounds, row-count drift alerts)
  - Add backfill mode for last N days
  - Add a city dimension dataset (still files in ADLS)
  - Add CI/CD (ADF ARM template + Databricks Repos + pipeline)

---

## License / Notes
This project is meant for portfolio use. Verify Open-Meteo terms if you plan to commercialize or run at high frequency.
