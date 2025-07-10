# ❄️ Snowflake Game Logs Ingestion & Transformation Pipeline

This project demonstrates an end-to-end **serverless data pipeline in Snowflake** that automatically ingests semi-structured JSON logs (game event data) from an AWS S3 bucket, processes them in near real-time using **Snowpipe**, and transforms the data into an analytical format using **Streams** and **Tasks**.

---

## 🎯 Objective

To implement a scalable and automated data ingestion pipeline in Snowflake that:

- Detects and ingests new log files from AWS S3.
- Extracts and parses relevant fields from JSON data.
- Tracks newly ingested records using **Streams** for Change Data Capture (CDC).
- Transforms and loads the processed data into an **enhanced table** for downstream analytics and reporting.

---

## 📌 Architecture Overview

```
          +------------------+          
          |  AWS S3 Bucket   |   (raw JSON log files)
          +--------+---------+
                   |
                   ▼
         +---------+----------+      
         | External Stage     |  (UNI_KISHORE_PIPELINE)
         +---------+----------+
                   |
                   ▼
        +----------+----------+  
        | Snowpipe (Pipe)     |  (PIPE_GET_NEW_FILES)
        |  Auto-ingests files |
        +----------+----------+
                   |
                   ▼
        +----------+----------+
        | Raw Table (CTAS)    |  (ED_PIPELINE_LOGS)
        +----------+----------+
                   |
                   ▼
         +---------+----------+
         | Stream (CDC)       |  (ED_CDC_STREAM)
         +---------+----------+
                   |
                   ▼
         +---------+----------+
         | Task (Scheduler)   |  (CDC_LOAD_LOGS_ENHANCED)
         +---------+----------+
                   |
                   ▼
         +---------+----------+
         | Enhanced Table     |  (LOGS_ENHANCED)
         +--------------------+
```

---

## 🗃️ Snowflake Object Overview

### 📂 **Database**: `AGS_GAME_AUDIENCE`

### 📁 **Schemas**: `RAW`

### 1. **External Stage**

- **Name**: `UNI_KISHORE_PIPELINE`
- Points to an S3 bucket where the raw JSON log files are stored.

### 2. **File Format**

- A JSON file format named `ff_json_logs` is assumed to be used for parsing.

### 3. \*\*Initial Table: \*\*\`\`

```sql
CREATE TABLE RAW.PL_GAME_LOGS (RAW_LOG VARIANT);
```

- Used to store raw JSON logs in semi-structured format.

### 4. \*\*View: \*\*\`\`

```sql
CREATE OR REPLACE VIEW PL_LOGS AS
SELECT
  $1:user_event::VARCHAR AS USER_EVENT,
  $1:ip_address::VARCHAR AS IP_ADDRESS,
  $1:datetime_iso8601::DATETIME AS DATETIME_ISO8601,
  $1:user_login::VARCHAR AS USER_LOGIN,
  $1 AS RAW_LOG
FROM PL_GAME_LOGS
WHERE IP_ADDRESS IS NOT NULL;
```

- Converts semi-structured logs into tabular format for readability.

### 5. \*\*Intermediate Table: \*\*\`\`

- Created using a **CTAS (Create Table As Select)** operation from the stage:

```sql
CREATE TABLE ED_PIPELINE_LOGS AS (
  SELECT
    METADATA$FILENAME AS LOG_FILE_NAME,
    METADATA$FILE_ROW_NUMBER AS LOG_FILE_ROW_ID,
    CURRENT_TIMESTAMP(0) AS LOAD_LTZ,
    GET($1,'datetime_iso8601')::TIMESTAMP_NTZ AS DATETIME_ISO8601,
    GET($1,'user_event')::TEXT AS USER_EVENT,
    GET($1,'user_login')::TEXT AS USER_LOGIN,
    GET($1,'ip_address')::TEXT AS IP_ADDRESS
  FROM @UNI_KISHORE_PIPELINE (FILE_FORMAT => 'ff_json_logs')
);
```

### 6. \*\*Snowpipe: \*\*\`\`

```sql
CREATE PIPE PIPE_GET_NEW_FILES
AUTO_INGEST = TRUE
AWS_SNS_TOPIC = 'arn:aws:sns:us-west-2:321463406630:dngw_topic'
AS
COPY INTO ED_PIPELINE_LOGS FROM (
  SELECT ...
  FROM @UNI_KISHORE_PIPELINE (FILE_FORMAT => 'ff_json_logs')
);
```

- Snowpipe listens for new files using **AWS SNS** and loads them into `ED_PIPELINE_LOGS`.

### 7. \*\*Stream: \*\*\`\`

```sql
CREATE OR REPLACE STREAM ED_CDC_STREAM
ON TABLE ED_PIPELINE_LOGS;
```

- Captures changes (new records) inserted into `ED_PIPELINE_LOGS`.

### 8. \*\*Task: \*\*\`\`

- Scheduled task to load records from the stream into the final enhanced table.

```sql
EXECUTE TASK CDC_LOAD_LOGS_ENHANCED;
```

---

## 🥺 Data Validation Steps

To verify data flow and processing:

| Step | Purpose                      | Example SQL                                        |
| ---- | ---------------------------- | -------------------------------------------------- |
| 1️⃣  | Check files in stage         | `LIST @UNI_KISHORE_PIPELINE;`                      |
| 2️⃣  | Verify rows in raw table     | `SELECT COUNT(*) FROM PL_GAME_LOGS;`               |
| 3️⃣  | Check parsed view data       | `SELECT * FROM PL_LOGS;`                           |
| 4️⃣  | Check pipe status            | `SELECT SYSTEM$PIPE_STATUS('PIPE_GET_NEW_FILES');` |
| 5️⃣  | Check if stream has new data | `SELECT SYSTEM$STREAM_HAS_DATA('ED_CDC_STREAM');`  |
| 6️⃣  | Count data in final table    | `SELECT COUNT(*) FROM ENHANCED.LOGS_ENHANCED;`     |

---

## 🔄 Maintenance Tips

- **Pause/Resume Snowpipe:**

```sql
ALTER PIPE PIPE_GET_NEW_FILES SET PIPE_EXECUTION_PAUSED = TRUE;
ALTER PIPE PIPE_GET_NEW_FILES SET PIPE_EXECUTION_PAUSED = FALSE;
```

- **Truncate & Reset Pipeline:**

```sql
TRUNCATE TABLE ED_PIPELINE_LOGS;
TRUNCATE TABLE ENHANCED.LOGS_ENHANCED;
```

- **Monitor Stream Activity:**

```sql
SELECT * FROM ED_CDC_STREAM;
```

---

## ✅ Key Features

- **Serverless ingestion** with Snowpipe & auto-ingest from AWS S3.
- **Schema-on-read** using VARIANT and JSON parsing.
- **Metadata tracking** (filename, row ID, ingestion time).
- **Real-time CDC** with Streams.
- **Scheduled transformation** with Tasks.

---
