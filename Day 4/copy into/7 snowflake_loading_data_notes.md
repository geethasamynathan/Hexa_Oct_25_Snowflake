# Snowflake Data Loading 

## 0) Prerequisites & Concepts

### Roles & grants (minimum)
- `USAGE` on **warehouse**, **database**, **schema**
- `CREATE STAGE` to create stages (if needed)
- `USAGE` on **STORAGE INTEGRATION** (for external stages)
- `OWNERSHIP`/`INSERT` on target **tables**

### Stages (where files live before loading)
- **Internal stages**:  
  - **User stage**: `@~` (one per user)  
  - **Table stage**: `@%table_name`  
  - **Named internal stage**: `CREATE STAGE my_int_stage;`
- **External stages** (cloud storage):  
  - **S3**, **Azure Blob/ADLS Gen2**, **GCS** via `CREATE STAGE ... URL=... STORAGE_INTEGRATION=...`

### File formats
Create once, reuse everywhere:
```sql
CREATE OR REPLACE FILE FORMAT ff_csv
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('', 'NULL');
  ```
  ## Reasoning
  ### 1. FIELD_OPTIONALLY_ENCLOSED_BY

#### Purpose:
Specifies a character (usually a double quote ") that may optionally enclose string fields in your CSV file.

#### Why it's used:

To handle text values that might contain commas, line breaks, or special characters.

For example, in a CSV:

`1,"New York, USA",100`


Without enclosing quotes, **New York, USA** would be split into **two fields** because of the comma inside it. **With quotes**, it’s treated **as one field**.

“Optionally” means:

If the quotes are there, Snowflake respects them.

If they’re not there, Snowflake still processes the field normally.

### `2. NULL_IF`

#### Purpose:
Specifies one or more string values that should be interpreted as SQL NULL when loading data.

In your example:

`NULL_IF = ('', 'NULL')`


If the field value is empty (''), Snowflake stores it as NULL.

If the field contains the literal string NULL, Snowflake also stores it as NULL.

## Example:
### CSV data:

```cpp
1,John,35
2,,40
3,NULL,45
```
Row 2 → The second field ('') becomes NULL.

Row 3 → The second field ('NULL') becomes NULL.
```sql
CREATE OR REPLACE FILE FORMAT ff_json TYPE = JSON
 STRIP_OUTER_ARRAY = TRUE;
 ```

**STRIP_OUTER_ARRAY = TRUE**	If the JSON file contains a single outer array, Snowflake will treat each element of that array as a separate row when loading.

### Why use STRIP_OUTER_ARRAY = TRUE?
If your JSON looks like this:
```JSON
[
  { "id": 1, "name": "Alice" },
  { "id": 2, "name": "Bob" }
]

```
**Without** STRIP_OUTER_ARRAY = TRUE → Snowflake sees the whole array as one row.

**With** STRIP_OUTER_ARRAY = TRUE → Snowflake “strips” the outer array and loads **two separate rows** (one for Alice, one for Bob).
```SQL
CREATE OR REPLACE FILE FORMAT ff_parquet TYPE = PARQUET;
```
## **1. View All File Formats in the Current Schema**
```sql
SHOW FILE FORMATS;
```
Lists all file formats available in the current schema you are using.

2. View File Formats in a Specific Schema
```sql

SHOW FILE FORMATS IN SCHEMA my_database.my_schema;
```
3. View File Formats in a Specific Database
```sql

SHOW FILE FORMATS IN DATABASE my_database;
```
4. Filter by Name Pattern
```sql

SHOW FILE FORMATS LIKE 'my_file_format%';
```
This helps if you remember part of the name.

5. Get Full Details of a File Format
Step 1 – List the file formats:

```sql

SHOW FILE FORMATS LIKE 'sales_csv_format';
```
Note the name and schema from the output.

Step 2 – Describe the file format:

```sql
DESCRIBE FILE FORMAT sales_csv_format;
```
or, if in a different schema:

```sql
DESCRIBE FILE FORMAT my_database.my_schema.sales_csv_format;
```
---

## 1) Load from **local files** via **Internal Stage** (SnowSQL/CLI)

> Best for ad‑hoc or small batch loads from your laptop/server.

### Step‑by‑step
1) **Create target table**
```sql
CREATE OR REPLACE TABLE sales_raw (
  order_id NUMBER,
  order_date DATE,
  amount NUMBER(10,2),
  region STRING
);
```

2) **Create a named internal stage** (optional but recommended)
```sql
CREATE OR REPLACE STAGE stg_sales
  FILE_FORMAT = ff_csv; -- reference your file format
```

3) **Upload files to the stage** (from your machine) using **SnowSQL**:
```bash
snowsql -a <account> -u <user>

# From SnowSQL prompt (or snowsql -q inline)
!put file://C:\Temp\sales_*.csv @stg_sales auto_compress=true;
# or to table stage:
!put file://C:\Temp\sales_*.csv @%sales_raw auto_compress=true;
```
In snowsight we can browse and add (or) upload the files. 
to view the files we can use below command
`LIST @STG_SALES;`

4) **Copy from stage into table**
```sql
COPY INTO sales_raw
FROM @stg_sales
FILE_FORMAT = (FORMAT_NAME = ff_csv)
ON_ERROR = 'CONTINUE'            
PURGE = TRUE;                    
```
## 1️⃣ COPY INTO sales_raw

**Purpose:** Loads data into the **target table** sales_raw.

`sales_raw` must already exist in your database with columns that match (or can be mapped to) your CSV file’s data.

## 2️⃣ FROM @stg_sales

`@stg_sales` refers to a **named internal stage** where your CSV files are stored.

Snowflake will load all matching files in this stage unless you specify a file name.

## 3️⃣ FILE_FORMAT = (FORMAT_NAME = ff_csv)

Tells Snowflake **how to interpret** the files in the stage.

ff_csv is a **file format object** you created earlier with details like:

Delimiter (, or |)

Header row skip count

String enclosing character (")

Null value representation

Without this, Snowflake wouldn’t know how to read the CSV’s structure.

## 4️⃣ ON_ERROR = 'CONTINUE'

Defines **error-handling behavior** during the load:

`'CONTINUE'` → If a row has an error (e.g., bad data type), Snowflake **skips that row** and continues loading the rest.

### Alternatives:

`'ABORT_STATEMENT'` (default) → Stop loading if any row fails.

`'SKIP_FILE'` → Skip entire file if an error occurs.

## 5️⃣ PURGE = TRUE

After a successful load, Snowflake removes (deletes) the file from the stage.

This prevents accidentally reloading the same file in the future.

**If PURGE = FALSE** (default), the file stays in the stage after loading.
-

## 5) **Validate before full load** (optional)
```sql
COPY INTO sales_raw
FROM @stg_sales
FILE_FORMAT = (FORMAT_NAME = ff_csv)
VALIDATION_MODE = 'RETURN_ERRORS';   
```

---

## 2) Load from **Cloud Storage** via **External Stage** (Bulk)

> Best for production/batch pipelines; supports **S3**, **Azure**, **GCS**.

### 2A) Azure Blob/ADLS Gen2 (recommended with **STORAGE INTEGRATION**)

**One‑time (security admin):**
```sql
CREATE OR REPLACE STORAGE INTEGRATION az_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = AZURE
  ENABLED = TRUE
  AZURE_TENANT_ID = '<tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://<account>.dfs.core.windows.net/<container>/path');

GRANT USAGE ON INTEGRATION az_int TO ROLE data_loader;
```

**Create external stage (data/warehouse role):**
```sql
CREATE OR REPLACE STAGE ext_sales_az
  URL = 'azure://<account>.dfs.core.windows.net/<container>/landing/sales'
  STORAGE_INTEGRATION = az_int
  FILE_FORMAT = ff_csv;
```

**Bulk load:**
```sql
COPY INTO sales_raw
FROM @ext_sales_az
PATTERN = '.*\.csv(?:\.gz)?'
FILE_FORMAT = (FORMAT_NAME = ff_csv)
ON_ERROR = 'ABORT_STATEMENT'
FORCE = FALSE;   
```

### 2B) Amazon S3 (with integration)
```sql
CREATE OR REPLACE STORAGE INTEGRATION s3_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<acct-id>:role/snowflake-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://my-bucket/landing/sales/');

CREATE OR REPLACE STAGE ext_sales_s3
  URL = 's3://my-bucket/landing/sales/'
  STORAGE_INTEGRATION = s3_int
  FILE_FORMAT = ff_csv;

COPY INTO sales_raw
FROM @ext_sales_s3 FILE_FORMAT = (FORMAT_NAME = ff_csv);
```

---

## 3) **Snowsight Web UI** “Load Data” Wizard (no‑code)

> Best for demos and quick loads without CLI.

**Steps**
1. In **Snowsight** → **Databases** → choose **Schema** → **+** → **Load data**.  
2. Choose/define **table** and **file format**.  
3. **Drag‑drop files** (local) or **select from stage**.  
4. Preview mapping, set **header**, **delimiter**, **enclosure**, etc.  
5. Click **Load** → view load results.

---

## 4) **Continuous Loading** with **Snowpipe** (Auto‑Ingest Micro‑Batch)

> Near‑real‑time loading as files arrive in cloud storage (seconds to minutes).

### Steps
1) **Create target table** and **external stage**.
2) **Create Pipe**:
```sql
CREATE OR REPLACE PIPE pipe_sales
  AUTO_INGEST = TRUE
  AS
  COPY INTO sales_raw
  FROM @ext_sales_az
  FILE_FORMAT = (FORMAT_NAME = ff_csv)
  ON_ERROR = 'CONTINUE';
```

3) **Configure cloud notifications** to the pipe’s **notification channel**.

4) **Monitor**
```sql
SELECT * FROM INFORMATION_SCHEMA.COPY_HISTORY(TABLE_NAME => 'SALES_RAW', START_TIME => DATEADD('hour', -24, CURRENT_TIMESTAMP()));
SHOW PIPES;  DESC PIPE pipe_sales;
```

---

## 5) **Real‑Time** with **Snowpipe Streaming**

> Sub‑second ingestion directly into Snowflake tables via SDK (Java/Python).

**High‑level flow**
- Producer reads from source (e.g., Kafka) and calls **Snowpipe Streaming** API.
- SDK writes directly to target table.

---

## 6) Ingestion via **Connectors & Drivers**

- **Kafka Connector** (Sink)  
- **Spark Connector**  
- **Python/Node/Java/JDBC/ODBC**  
- **SQL API**  

---

## 7) Semi‑Structured Files (JSON/Parquet/Avro/ORC/XML)

**Load JSON into VARIANT**
```sql
CREATE OR REPLACE TABLE clickstream (payload VARIANT);

COPY INTO clickstream
FROM @ext_json_stage
FILE_FORMAT = (TYPE = JSON STRIP_OUTER_ARRAY = TRUE);
```

**Query JSON**
```sql
SELECT
  payload:sessionId::string AS session_id,
  payload:eventType::string AS event_type,
  TO_TIMESTAMP_NTZ(payload:ts) AS ts
FROM clickstream;
```

**Parquet directly to typed columns**
```sql
CREATE OR REPLACE TABLE trips (id NUMBER, km NUMBER(10,2), ts TIMESTAMP_NTZ);

COPY INTO trips
FROM @ext_parquet_stage
FILE_FORMAT = (TYPE = PARQUET)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

---

## 8) Transform While Loading (COPY Options)

```sql
COPY INTO sales_clean
FROM @ext_sales_az
FILE_FORMAT = (FORMAT_NAME = ff_csv)
ON_ERROR = 'CONTINUE'
FORCE = FALSE
TRUNCATECOLUMNS = TRUE
ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

---

## 9) Incremental / CDC Pattern (Streams + Tasks)

### Steps
1) Load files to **staging table**.
2) Create stream:
```sql
CREATE OR REPLACE STREAM str_sales_stage ON TABLE sales_stage;
```
3) Create task:
```sql
CREATE OR REPLACE TASK t_merge_sales
  WAREHOUSE = etl_wh
  SCHEDULE = 'USING CRON 0 0 * * * Asia/Kolkata'
AS
MERGE INTO sales_final tgt
USING (
  SELECT * FROM sales_stage WHERE METADATA$ACTION IN ('INSERT', 'UPDATE')
) src
ON tgt.order_id = src.order_id
WHEN MATCHED THEN UPDATE SET amount = src.amount, order_date = src.order_date
WHEN NOT MATCHED THEN INSERT (order_id, order_date, amount) VALUES (src.order_id, src.order_date, src.amount);
```
4) Enable task:
```sql
ALTER TASK t_merge_sales RESUME;
```

---

## 10) Monitoring & Troubleshooting

```sql
SELECT * FROM INFORMATION_SCHEMA.COPY_HISTORY(TABLE_NAME => 'SALES_RAW', START_TIME => DATEADD('day', -1, CURRENT_TIMESTAMP()));
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.LOAD_HISTORY WHERE LAST_LOAD_TIME >= DATEADD('day', -1, CURRENT_TIMESTAMP());
```

---

## 11) Best Practices
- Use stages; avoid massive loops of INSERT.
- Compress files (`.gz`), size 100–250MB compressed.
- Partition in cloud storage.
- Validate with `VALIDATION_MODE`.
- Separate raw vs. curated tables.
- Automate with Snowpipe or Streaming.
- Monitor with `COPY_HISTORY` & `LOAD_HISTORY`.

---

## 12) Quick Reference — COPY INTO Patterns

```sql
COPY INTO tgt FROM @my_stage FILE_FORMAT=(FORMAT_NAME=ff_csv);
COPY INTO tgt FROM @%tgt FILE_FORMAT=(FORMAT_NAME=ff_csv);
COPY INTO tgt FROM @ext_stage PATTERN='.*2025/08/.*\.csv\.gz' FILE_FORMAT=(FORMAT_NAME=ff_csv) ON_ERROR='CONTINUE';
COPY INTO tgt FROM @ext_stage FILE_FORMAT=(FORMAT_NAME=ff_csv) VALIDATION_MODE='RETURN_ERRORS';
```
