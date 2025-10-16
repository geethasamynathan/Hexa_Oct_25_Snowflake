
# **Snowflake Notes – Using `SIZE_LIMIT` in COPY Command**

## 1. **Database & Table Creation**
```sql
CREATE OR REPLACE DATABASE COPY_DB;

CREATE OR REPLACE TABLE COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));
```

### 🔹 Explanation:
- `CREATE OR REPLACE DATABASE COPY_DB;`  
  Creates a new database named **COPY_DB**. If it already exists, it will be replaced.  
- `CREATE OR REPLACE TABLE` defines a table `ORDERS` inside `COPY_DB.PUBLIC` schema.  
- Columns like `ORDER_ID`, `AMOUNT`, `PROFIT`, etc. are used to store order details.  

✅ **Expected Output**:  
```
+--------------------------------+
| status                         |
+--------------------------------+
| Table ORDERS successfully created. |
+--------------------------------+
```

---

## 2. **Stage Creation**
```sql
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/size/';
```

### 🔹 Explanation:
- A **stage** is a storage location where files are kept before loading them into tables.  
- Here, an **external stage** is created pointing to an **Amazon S3 bucket**.  
- This stage (`aws_stage_copy`) will hold CSV files.  

✅ **Expected Output**:  
```
+--------------------------------+
| status                         |
+--------------------------------+
| Stage AWS_STAGE_COPY successfully created. |
+--------------------------------+
```

---

## 3. **Listing Files in Stage**
```sql
LIST @aws_stage_copy;
```

### 🔹 Explanation:
- This lists all files present in the stage `aws_stage_copy`.  
- You can check available CSV files before loading them.  

✅ **Expected Output Example**:
```
+------------------------------------------+-------+-------------+
| name                                     | size  | last_modified |
+------------------------------------------+-------+-------------+
| s3://snowflakebucket-copyoption/size/OrderDetails1.csv | 1200  | 2025-08-15 10:30:00 |
| s3://snowflakebucket-copyoption/size/OrderDetails2.csv | 1500  | 2025-08-15 10:31:00 |
+------------------------------------------+-------+-------------+
```

---

## 4. **Loading Data with `COPY INTO` and `SIZE_LIMIT`**
```sql
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    SIZE_LIMIT=20000;
```

### 🔹 Explanation:
- **COPY INTO** loads data from files in the stage into the table `ORDERS`.  
- `file_format= (type = csv field_delimiter=',' skip_header=1)` tells Snowflake the file is CSV, uses `,` as delimiter, and ignores header row.  
- `pattern='.*Order.*'` ensures only files with the word *Order* in their name are loaded.  
- **`SIZE_LIMIT=20000`**:  
  - Restricts the **maximum file size (in bytes)** that Snowflake will load at one time.  
  - Useful when files are too large and you want to avoid memory or performance issues.  
  - Here, only files ≤ **20,000 bytes (≈20 KB)** will be considered.  

✅ **Expected Output Example**:
```
+-------------------+--------------------+----------+-------------+
| file              | rows_loaded        | status   | error_count |
+-------------------+--------------------+----------+-------------+
| OrderDetails1.csv | 1000               | LOADED   | 0           |
| OrderDetails2.csv | 1200               | LOADED   | 0           |
| OrderDetails3.csv | 0                  | SKIPPED  | Exceeds SIZE_LIMIT |
+-------------------+--------------------+----------+-------------+
```

---

# **Key Notes:**
1. `SIZE_LIMIT` applies to **individual files**, not the whole batch.  
2. If a file exceeds the limit, it will be **skipped**, and Snowflake will not attempt to load it.  
3. Use this when handling **mixed file sizes** in your staging area.  
4. Best practice: combine with `ON_ERROR` to control error-handling.  
