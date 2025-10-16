# ❄️ Detailed Explanation -- Handling Rejected Records in Snowflake

This document explains **how to handle rejected records while loading
data into Snowflake** using different techniques such as
`VALIDATION_MODE`, `ON_ERROR`, and storing failed rows for further
analysis.

------------------------------------------------------------------------

## 1. Stage Creation & File Listing

``` sql
CREATE OR REPLACE STAGE COPY_DB.PUBLIC.aws_stage_copy
    url='s3://snowflakebucket-copyoption/returnfailed/';

LIST @COPY_DB.PUBLIC.aws_stage_copy;
```

-   **Stage**: A Snowflake stage is a location to store files before
    loading them.\
-   **LIST** command: Lists all files present in the stage.

------------------------------------------------------------------------

## 2. Using `VALIDATION_MODE`

``` sql
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    VALIDATION_MODE = RETURN_ERRORS;
```

-   **VALIDATION_MODE** is used to validate the data before loading.\
-   Options include:
    -   `RETURN_ERRORS` → Returns all rows with errors.\
    -   `RETURN_1_ROWS` → Returns only one erroneous row.

This is useful for testing file quality **before actually loading** into
the target table.

------------------------------------------------------------------------

## 3. Creating Target Table

``` sql
CREATE OR REPLACE TABLE COPY_DB.PUBLIC.ORDERS (
    ORDER_ID VARCHAR(30),
    AMOUNT VARCHAR(30),
    PROFIT INT,
    QUANTITY INT,
    CATEGORY VARCHAR(30),
    SUBCATEGORY VARCHAR(30));
```

-   Defines schema for the target table `ORDERS`.\
-   The table has string & integer fields for order details.

------------------------------------------------------------------------

## 4. Storing Rejected Records

After running `COPY INTO ... VALIDATION_MODE = RETURN_ERRORS`, we can
capture rejected rows.

``` sql
CREATE OR REPLACE TABLE rejected AS 
SELECT rejected_record 
FROM TABLE(result_scan(last_query_id()));
```

-   **`result_scan(last_query_id())`** → Returns results of the last
    executed query.\
-   **`rejected_record`** → Column that stores the failed row content.

You can also append additional rejected rows:

``` sql
INSERT INTO rejected
SELECT rejected_record 
FROM TABLE(result_scan(last_query_id()));
```

Now we can query:

``` sql
SELECT * FROM rejected;
```

------------------------------------------------------------------------

## 5. Without `VALIDATION_MODE` (Using `ON_ERROR`)

``` sql
COPY INTO COPY_DB.PUBLIC.ORDERS
    FROM @aws_stage_copy
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*Order.*'
    ON_ERROR = CONTINUE;
```

-   **`ON_ERROR`** → Defines how to handle bad records:
    -   `CONTINUE` → Skip bad records and continue loading.\
    -   `ABORT_STATEMENT` → Stops loading if any error occurs.\
    -   `SKIP_FILE` → Skip entire file if any error is found.

You can still validate afterwards:

``` sql
SELECT * FROM TABLE(validate(orders, job_id => '_last'));
```

-   `validate()` function checks which rows failed during the last load.

------------------------------------------------------------------------

## 6. Working with Rejected Records

``` sql
SELECT REJECTED_RECORD FROM rejected;
```

-   This retrieves all raw rejected rows.

To split into structured columns:

``` sql
CREATE OR REPLACE TABLE rejected_values AS
SELECT 
  SPLIT_PART(rejected_record, ',', 1) AS ORDER_ID,
  SPLIT_PART(rejected_record, ',', 2) AS AMOUNT,
  SPLIT_PART(rejected_record, ',', 3) AS PROFIT,
  SPLIT_PART(rejected_record, ',', 4) AS QUANTITY,
  SPLIT_PART(rejected_record, ',', 5) AS CATEGORY,
  SPLIT_PART(rejected_record, ',', 6) AS SUBCATEGORY
FROM rejected;

SELECT * FROM rejected_values;
```

-   **`SPLIT_PART`** → Splits the rejected row into columns by delimiter
    (`,`).\
-   This way, you can analyze the **exact cause of rejection** for each
    record.

------------------------------------------------------------------------

## ✅ Key Takeaways

1.  **VALIDATION_MODE** is used for **testing data quality before
    load**.\
2.  **ON_ERROR** is used during load to decide **how Snowflake reacts to
    errors**.\
3.  **Rejected records** can be stored and analyzed by capturing them
    with `result_scan()` and splitting them into structured columns.\
4.  This process is **critical for ETL/ELT pipelines**, ensuring bad
    data does not corrupt target tables.

------------------------------------------------------------------------

This workflow is widely used in **real-world data engineering** to
handle corrupted or incomplete data files gracefully.
