# VALIDATION_MODE in COPY Command — Explanation

`VALIDATION_MODE` is a special option in Snowflake’s **COPY INTO** command that allows you to **validate staged data before actually loading it into a table**.  
It’s mainly used to:
- Catch errors early (bad rows, type mismatches, malformed files).
- Check file compatibility with the target table.
- Avoid inserting bad data.

---

## 1️⃣ Create Database & Table
```sql
CREATE OR REPLACE DATABASE COPY_DB;

CREATE OR REPLACE TABLE COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30)
);
```

- Creates a new database `COPY_DB`.
- Inside it, a table `ORDERS` with 6 columns (simple schema for orders data).
- Data type mismatch may occur if the source files don’t align (for example, non-numeric data in `PROFIT` column).

---

## 2️⃣ Create Stage (pointing to S3 bucket)
```sql
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    URL='s3://snowflakebucket-copyoption/size/';
```

- A **stage** is a reference to an external storage location (here, an S3 bucket).
- This does **not pull data**; it just registers the bucket with Snowflake.

Check staged files:
```sql
LIST @COPY_DB.PUBLIC.aws_stage_copy;
```

- Shows all files available in that bucket path.

---

## 3️⃣ Validate files with `RETURN_ERRORS`
```sql
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    FILE_FORMAT=(TYPE=CSV FIELD_DELIMITER=',' SKIP_HEADER=1)
    PATTERN='.*Order.*'
    VALIDATION_MODE=RETURN_ERRORS;
```

- **What it does:**
  - Reads all files in the stage that match `.*Order.*` (regex).
  - Tries parsing them as CSV.
  - Instead of inserting, it only **returns rows that would fail**.
- **No data is inserted** into the table.

Check table:
```sql
SELECT * FROM ORDERS; -- still empty
```

---

## 4️⃣ Validate first 5 rows only
```sql
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    FILE_FORMAT=(TYPE=CSV FIELD_DELIMITER=',' SKIP_HEADER=1)
    PATTERN='.*Order.*'
    VALIDATION_MODE=RETURN_5_ROWS;
```

- **What it does:**
  - Parses only the first 5 rows from each file (useful for quick smoke testing).
  - Reports whether they would succeed or fail.
- Still, **no rows are loaded**.

---

## 5️⃣ Switch to a folder with error files
```sql
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    URL='s3://snowflakebucket-copyoption/returnfailed/';

LIST @COPY_DB.PUBLIC.aws_stage_copy;
```

- Stage now points to another S3 folder containing files with known issues.

---

## 6️⃣ Show all errors
```sql
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @COPY_DB.PUBLIC.aws_stage_copy
    FILE_FORMAT=(TYPE=CSV FIELD_DELIMITER=',' SKIP_HEADER=1)
    PATTERN='.*Order.*'
    VALIDATION_MODE=RETURN_ERRORS;
```

- Scans error-prone files.
- Returns details on **all rows that fail validation**.

---

## 7️⃣ Validate first row only
```sql
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @COPY_DB.PUBLIC.aws_stage_copy
    FILE_FORMAT=(TYPE=CSV FIELD_DELIMITER=',' SKIP_HEADER=1)
    PATTERN='.*error.*'
    VALIDATION_MODE=RETURN_1_ROWS;
```

- Only parses the **first row** of files matching `.*error.*`.
- Useful for quick debugging instead of scanning full files.

---

# ✅ Key Takeaways

- **`VALIDATION_MODE=RETURN_ERRORS`**  
  Reports all bad rows. No data loaded.
  
- **`VALIDATION_MODE=RETURN_N_ROWS`**  
  Validates the first N rows (good for quick tests). No data loaded.

- **No rows ever get inserted** when using `VALIDATION_MODE`.

- Once validation passes, you can run the same COPY without `VALIDATION_MODE` to actually load data.

---
