# 🧾 Snowflake Cloning & Time Travel Assignment

## ✅ Step 1 — Create the SUPPLIER Table
```sql
-- Switch to the ACCOUNTADMIN role
USE ROLE ACCOUNTADMIN;

-- Use appropriate database and warehouse
USE DATABASE DEMO_DB;
USE WAREHOUSE COMPUTE_WH;

-- Create the SUPPLIER table from Snowflake sample data
CREATE OR REPLACE TABLE DEMO_DB.PUBLIC.SUPPLIER AS
SELECT * 
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.SUPPLIER;
```
### 📝 Explanation
- Using the **TPCH sample data** provided by Snowflake.
- The `SUPPLIER` table is now copied into your `DEMO_DB.PUBLIC` schema.

---

## ✅ Step 2 — Create a Clone of the Table
```sql
CREATE OR REPLACE TABLE SUPPLIER_CLONE CLONE SUPPLIER;
```
### 🧠 Concept
Cloning in Snowflake is a **zero-copy** operation. It creates a **snapshot** of the source table’s metadata and data **without duplicating storage**.  
Any changes to one table after cloning **don’t affect** the other.

---

## ✅ Step 3 — Update the Clone Table and Note the Query ID
```sql
UPDATE SUPPLIER_CLONE
SET S_PHONE = '###';
```

After running the above statement, note the **Query ID** using:

```sql
SELECT LAST_QUERY_ID();
```

🧾 Example Query ID:
```
01a2b3c4-1234-5678-90de-fghi12345678
```

---

## ✅ Step 4 — Create Another Clone Using Time Travel (Before Update)
```sql
CREATE OR REPLACE TABLE SUPPLIER_CLONE_V1 
CLONE SUPPLIER_CLONE 
BEFORE (STATEMENT => '01a2b3c4-1234-5678-90de-fghi12345678');
```

### 🕒 Explanation
- The `BEFORE (STATEMENT => ...)` clause clones the table **as it existed before** that query was executed.
- This clone (`SUPPLIER_CLONE_V1`) will have the **original phone numbers** (before `S_PHONE` was updated).

---

## ❓ Question
> If we delete the source table, does the clone still exist?

✅ **Answer:** YES — the clone still exists.

### 📘 Explanation
- Clones in Snowflake are **independent objects** once created.
- Deleting the source table does **not affect** the clone.
- The clone keeps its **own data snapshot** from cloning time.
- Snowflake uses **copy-on-write storage**, so shared data blocks are preserved until modified.

---

### 🔍 Example Demo
```sql
DROP TABLE SUPPLIER;

SELECT COUNT(*) FROM SUPPLIER_CLONE;        -- ✅ Still works
SELECT COUNT(*) FROM SUPPLIER_CLONE_V1;     -- ✅ Still works
```

---

## 🧩 Summary Table

| Step | Action | Object | Description |
|------|---------|---------|--------------|
| 1 | Create table | SUPPLIER | Original table from sample data |
| 2 | Clone | SUPPLIER_CLONE | Zero-copy clone |
| 3 | Update | SUPPLIER_CLONE | Modified phone numbers |
| 4 | Clone using time travel | SUPPLIER_CLONE_V1 | Snapshot before the update |
| ❓ | Drop source | SUPPLIER | Clone remains intact ✅ |

---

### 📊 Visual Concept Diagram

```
SUPPLIER ──┬──► SUPPLIER_CLONE (updated)
            │
            └──► SUPPLIER_CLONE_V1 (time-traveled before update)
```
🧠 Even after `SUPPLIER` is dropped, both clones remain functional.
