```markdown
# Real-Time Log Processing Data Pipeline (AWS)

##  Project Overview
This project demonstrates an **end-to-end real-time log processing pipeline** built entirely on **AWS**.  
It simulates how production application logs are streamed, stored, transformed, and analyzed using a **serverless data engineering stack**.

**Goal:**  
Ingest live log data → Store in S3 → Transform with Glue (PySpark) → Query with Athena / Redshift.

---

## Architecture

```

Application Logs (Producer)
│
▼
Kinesis Data Firehose  →  S3 (Raw Zone)
│
▼
AWS Glue (ETL: PySpark) →  S3 (Curated Zone: Parquet)
│
▼
Athena / Redshift  →  Analytics, Dashboards

````

---

## 🧩 AWS Services Used

| Service | Purpose |
|----------|----------|
| **Amazon Kinesis Data Firehose** | Ingests streaming JSON logs in real-time and delivers them to S3. |
| **Amazon S3** | Central data lake storing raw and curated data. |
| **AWS Glue Crawler** | Automatically discovers schema from JSON files and creates Glue Data Catalog tables. |
| **AWS Glue ETL Job (PySpark)** | Cleans and transforms raw logs into Parquet format, partitioned by date and hour. |
| **AWS Glue Data Catalog** | Stores metadata and schema for Athena and Redshift queries. |
| **Amazon Athena** | Serverless SQL engine to query data directly from S3. |
| **Amazon Redshift (optional)** | Data warehouse for historical analysis and BI dashboards. |
| **AWS IAM** | Secure role-based access between AWS services. |
| **Amazon CloudWatch (optional)** | Monitors pipeline health and alerts on failures. |

---

##  Tech Stack

- **Language:** Python, PySpark  
- **Services:** Kinesis, S3, Glue, Athena, Redshift  
- **Data Format:** JSON → Parquet  
- **Architecture:** Serverless (no EC2 required)

---

## Project Flow

### 1. Log Ingestion
- A **Python producer script** generates mock logs and sends them to **Kinesis Data Firehose**.
- Firehose buffers and delivers logs to S3 (`/raw/` folder) as compressed JSON.

### 2. Schema Discovery
- A **Glue Crawler** scans S3 and creates a table `logs_db.raw_logs` in the **Glue Data Catalog**.

### 3. Transformation (ETL)
- A **Glue PySpark job** reads raw logs, converts timestamps, filters WARN/ERROR records, and writes to S3 (`/curated/`) in Parquet format, partitioned by date and hour.

### 4. Querying
- **Athena** queries the curated S3 data directly using SQL.
- Run `MSCK REPAIR TABLE logs_analytics.app_logs_curated;` after new partitions are added.
- **Redshift** optionally loads Parquet data for deeper analytics or BI dashboards.

---

## Sample Log Format

```json
{
  "timestamp": "2025-11-09T10:15:23Z",
  "level": "ERROR",
  "service": "payment-api",
  "message": "Card declined",
  "user_id": "u123",
  "ip": "10.1.2.3"
}
````

---

## Setup Instructions

### Step 1: Create S3 Bucket

Create a bucket, e.g. `de-logs-streaming-yourname`, with subfolders:

```
raw/
curated/
```

### Step 2: Create Kinesis Firehose

* Source: **Direct PUT**
* Destination: **S3** (`raw/` folder)
* Buffer size: 1 MB, Buffer interval: 60 sec
* Compression: GZIP
* IAM Role: `FirehoseToS3Role`

### Step 3: Run the Log Producer (Python)

```python
import json, time, random, datetime, boto3

firehose = boto3.client("firehose", region_name="ap-south-1")
stream = "realtime-log-stream"
services = ["auth-api", "payment-api", "order-api"]
levels = ["INFO", "WARN", "ERROR"]

while True:
    log = {
        "timestamp": datetime.datetime.utcnow().isoformat() + "Z",
        "level": random.choices(levels, weights=[0.8, 0.15, 0.05])[0],
        "service": random.choice(services),
        "message": "sample log message",
        "user_id": f"user_{random.randint(1,1000)}",
        "ip": f"10.0.{random.randint(0,255)}.{random.randint(0,255)}"
    }
    firehose.put_record(
        DeliveryStreamName=stream,
        Record={"Data": (json.dumps(log) + "\n").encode("utf-8")}
    )
    time.sleep(0.2)
```

---

### Step 4: Create Glue Database and Crawler

* Database: `logs_db`
* Crawler source: S3 `raw/` path
* Output: Table `logs_db.raw_logs`

---

### Step 5: Create Glue ETL Job (PySpark)

```python
import sys
from awsglue.utils import getResolvedOptions
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_iso8601_timestamp, to_date, hour

args = getResolvedOptions(sys.argv, ["JOB_NAME"])
spark = SparkSession.builder.appName("logs_raw_to_curated").getOrCreate()

df = spark.table("logs_db.raw_logs")

df2 = (
    df.withColumn("ts", from_iso8601_timestamp(col("timestamp")))
      .withColumn("dt", to_date(col("ts")))
      .withColumn("hr", hour(col("ts")))
      .filter(col("ts").isNotNull())
)

df_filtered = df2.filter(col("level").isin("WARN", "ERROR"))

(
    df_filtered
      .repartition("dt", "hr")
      .write.mode("overwrite")
      .partitionBy("dt", "hr")
      .parquet("s3://de-logs-streaming-yourname/curated/")
)
```

---

### Step 6: Query with Athena

```sql
CREATE DATABASE IF NOT EXISTS logs_analytics;

CREATE EXTERNAL TABLE logs_analytics.app_logs_curated (
  timestamp string,
  level string,
  service string,
  message string,
  user_id string,
  ip string,
  ts timestamp
)
PARTITIONED BY (dt date, hr int)
STORED AS PARQUET
LOCATION 's3://de-logs-streaming-yourname/curated/';
```

Then run:

```sql
MSCK REPAIR TABLE logs_analytics.app_logs_curated;
```

Example query:

```sql
SELECT service, count(*) AS error_count
FROM logs_analytics.app_logs_curated
WHERE level = 'ERROR' AND dt = current_date - 1
GROUP BY service
ORDER BY error_count DESC;
```

---

## 📊 Sample Output (Athena Query)

| service     | error_count |
| ----------- | ----------- |
| payment-api | 104         |
| auth-api    | 67          |
| order-api   | 45          |

---

## Key Learnings

* Implemented **real-time streaming ingestion** using Kinesis Firehose.
* Built a **serverless data lake** on S3 with raw and curated layers.
* Automated schema discovery using **Glue Crawlers**.
* Used **PySpark in Glue** for transformation and **Parquet partitioning** for cost optimization.
* Queried data with **Athena** (schema-on-read) and optionally loaded into **Redshift**.

---

## Future Enhancements

* Add **CloudWatch Alarms** for Glue job failures.
* Integrate **Airflow (MWAA)** for orchestration.
* Build **QuickSight dashboards** for real-time error monitoring.
* Replace Firehose with **Kafka (MSK)** for greater control.

---

## Author

**Kumar Chitransh*
Data Engineer | AWS | PySpark | SQL
[chitransh.sudhir@gmail.com](mailto:chitransh.sudhir@gmail.com)
[LinkedIn Profile](https://linkedin.com/in/kumarchitransh)

```
glance.
```
