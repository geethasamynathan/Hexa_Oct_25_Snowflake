# Lab – Snowflake External Stage, Data Loading, and Cluster Key Optimization

## 🎯 Objective
This lab demonstrates how to:
- Create a **publicly accessible stage** in Snowflake (using AWS S3).  
- **Load data** from S3 into Snowflake using the **COPY INTO** command.  
- Understand **query performance** before and after applying a **cluster key**.  
- Analyze the impact of **clustering** on query performance and caching.

---

## 🧩 Step 1 – Create an External Stage

### 📘 Concept
A **Stage** in Snowflake is a location used to store files (internal or external) that Snowflake can access for loading/unloading data.  
Here, we’re creating an **external stage** that points to an **AWS S3 bucket**.

### 🧠 Syntax
```sql
CREATE OR REPLACE STAGE MANAGE_DB.external_stages.aws_stage
    url='s3://bucketsnowflakes3';
```

### 💡 Notes
- `MANAGE_DB.external_stages.aws_stage`:  
  The full path includes the database (`MANAGE_DB`), schema (`external_stages`), and stage name (`aws_stage`).
- `url`:  
  Specifies the **Amazon S3 bucket** that stores your files.  
  This bucket must be **publicly accessible** or have proper access permissions configured.

---

## 🧾 Step 2 – List Files in the Stage

### 📘 Concept
You can verify that the stage has the correct data files before loading them.

### 🧠 Command
```sql
LIST @MANAGE_DB.external_stages.aws_stage;
```

### 💡 Notes
- The `@` symbol represents a **stage reference**.  
- This command lists all files available in the specified stage along with metadata such as size and timestamp.

---

## 📦 Step 3 – Load Data into a Snowflake Table

### 📘 Concept
The **COPY INTO** command loads data from an external stage into a Snowflake table.  
You can specify the file format, header handling, and file name patterns.

### 🧠 Command
```sql
COPY INTO OUR_FIRST_DB.PUBLIC.ORDERS
    FROM @MANAGE_DB.external_stages.aws_stage
    file_format= (type = csv field_delimiter=',' skip_header=1)
    pattern='.*OrderDetails.*';
```

### 💡 Notes
- `file_format`: Defines the type of input file (here CSV), field delimiter (`,`), and header handling.
- `pattern`: A regular expression that selects only the files whose names match **OrderDetails**.
- The data is now successfully loaded into `OUR_FIRST_DB.PUBLIC.ORDERS`.

---

## 🔍 Step 4 – Verify the Data Load

### 🧠 Command
```sql
SELECT * FROM OUR_FIRST_DB.PUBLIC.ORDERS;
```

### 💡 Notes
This retrieves all records from the **ORDERS** table, confirming the load process.

---

## 🧱 Step 5 – Create a New Table for Performance Testing

### 📘 Concept
We create a new table `ORDERS_CACHING` and populate it with data for testing **clustering and query performance**.

### 🧠 Command
```sql
CREATE OR REPLACE TABLE ORDERS_CACHING (
  ORDER_ID     VARCHAR(30),
  AMOUNT       NUMBER(38,0),
  PROFIT       NUMBER(38,0),
  QUANTITY     NUMBER(38,0),
  CATEGORY     VARCHAR(30),
  SUBCATEGORY  VARCHAR(30),
  DATE         DATE
);
```

### 💡 Notes
- The schema matches the source `ORDERS` table.
- The `DATE` column will be generated dynamically using a **random function**.

---

## 🧩 Step 6 – Insert Data with Random Dates

### 📘 Concept
To simulate large data for performance testing, we use a **CROSS JOIN** and generate random date values.

### 🧠 Command
```sql
INSERT INTO ORDERS_CACHING 
SELECT
  t1.ORDER_ID,
  t1.AMOUNT,
  t1.PROFIT,
  t1.QUANTITY,
  t1.CATEGORY,
  t1.SUBCATEGORY,
  DATE(UNIFORM(1500000000,1700000000,(RANDOM())))
FROM ORDERS t1
CROSS JOIN (SELECT * FROM ORDERS) t2;
```

### 💡 Notes
- `UNIFORM()` generates random numbers between the given range (Unix timestamps).  
- `RANDOM()` acts as a seed for generating pseudo-random numbers.  
- The result simulates **millions of rows** by cross joining two tables.

---

## ⚡ Step 7 – Query Performance Before Clustering

### 📘 Concept
Run a filter query on the `DATE` column **before adding any cluster key** to observe the baseline performance.

### 🧠 Command
```sql
SELECT * FROM ORDERS_CACHING WHERE DATE = '2020-06-09';
```

### 💡 Notes
- Since no **clustering** exists, Snowflake scans the entire micro-partition set.  
- The performance is usually **slower** for large datasets.

---

## 🧹 Step 8 – Drop and Recreate the Table (Optional Reset)

```sql
DROP TABLE ORDERS_CACHING;
```

---

## 🧱 Step 9 – Add a Cluster Key

### 📘 Concept
A **Cluster Key** defines how data is physically organized within Snowflake micro-partitions.  
It improves query performance for filter predicates on that column.

### 🧠 Command
```sql
ALTER TABLE ORDERS_CACHING CLUSTER BY (DATE);
```

### 💡 Notes
- Clustering reorganizes data internally for faster retrieval.  
- It can take **30 minutes to 1 hour** to take full effect, depending on warehouse size.  
- Use different query filters to avoid cache influence while testing.

---

## ⚙️ Step 10 – Test Query Performance After Clustering

```sql
SELECT * FROM ORDERS_CACHING WHERE DATE = '2020-01-05';
```

### 💡 Notes
- Performance should now be noticeably **improved** for queries filtering by `DATE`.  
- Cluster pruning reduces the number of scanned micro-partitions.

---

## 🚫 Step 11 – Non-Ideal Clustering Example

### 📘 Concept
Sometimes, using a **function-based clustering key** can be less effective.  
Here, we use `MONTH(DATE)` as the cluster key to demonstrate poor clustering design.

### 🧠 Commands
```sql
SELECT * FROM ORDERS_CACHING WHERE MONTH(DATE)=11;

ALTER TABLE ORDERS_CACHING CLUSTER BY (MONTH(DATE));
```

### 💡 Notes
- Function-based clustering (`MONTH(DATE)`) may cause **data skew** because multiple rows share the same month value.
- Always choose a **column** (not an expression) that distributes values **evenly** across micro-partitions.

---

## 🧠 Summary

| Step | Task | Key Learning |
|------|------|---------------|
| 1 | Create Stage | Store files in AWS S3 and make them accessible to Snowflake |
| 2 | List Stage | Verify available data files |
| 3 | Copy Into | Load data efficiently from stage |
| 4 | Verify | Confirm successful data load |
| 5 | Create Table | Prepare structure for testing |
| 6 | Insert Data | Generate random data for performance testing |
| 7 | Test Query | Observe performance before clustering |
| 9 | Add Cluster Key | Optimize physical data organization |
| 10 | Re-Test Query | Measure improved performance |
| 11 | Function Key | Understand pitfalls of improper clustering |

---

## 💬 Key Takeaways
- **External Stages** simplify integration with cloud storage like AWS S3.  
- **COPY INTO** is the preferred method for loading bulk data into Snowflake.  
- **Clustering** significantly improves query performance for large datasets by minimizing data scanning.  
- Choose cluster keys based on **frequently filtered columns** and ensure **even data distribution** for optimal results.
