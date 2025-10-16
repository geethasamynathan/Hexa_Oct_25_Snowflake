# Snowflake: Databases, Schemas, and Tables — Incremental Learning

## 1. Database in Snowflake

### 1.1 Definition
A **Database** in Snowflake is a logical container that stores schemas, tables, views, and other objects.  
It’s the **highest-level** organization unit in Snowflake.

---

### 1.2 Real-World Analogy
Think of a **Database** like a **cabinet** in a library — it holds multiple **folders (schemas)**, and each folder contains files (tables, views).

---

### 1.3 Ways to Create a Database

#### Method 1 — Using SQL
```sql
CREATE DATABASE sales_db;
```

#### Method 2 — With Optional Comment
```sql
CREATE DATABASE sales_db COMMENT = 'Database for storing sales data';
```

#### Method 3 — If Not Exists
```sql
CREATE DATABASE IF NOT EXISTS sales_db;
```

---

### 1.4 Other Operations on Database
```sql
-- Rename a database
ALTER DATABASE sales_db RENAME TO sales_archive_db;

-- Change comment
ALTER DATABASE sales_db SET COMMENT = 'Updated description';

-- Drop a database
DROP DATABASE sales_db;

-- Show all databases
SHOW DATABASES;

-- Use a specific database
USE DATABASE sales_db;
```

---

## 2. Schema in Snowflake

### 2.1 Definition
A **Schema** is a logical grouping of database objects (tables, views, procedures) within a database.

---

### 2.2 Real-World Analogy
A **Schema** is like a **folder** inside a cabinet — it groups related files (tables/views) together.

---

### 2.3 Ways to Create a Schema

#### Method 1 — Basic
```sql
CREATE SCHEMA sales_schema;
```

#### Method 2 — Within a Specific Database
```sql
CREATE SCHEMA sales_db.marketing_schema;
```

#### Method 3 — If Not Exists
```sql
CREATE SCHEMA IF NOT EXISTS sales_schema;
```

#### Method 4 — With Comment
```sql
CREATE SCHEMA sales_schema COMMENT = 'Schema for sales data tables';
```

---

### 2.4 Other Operations on Schema
```sql
-- Rename schema
ALTER SCHEMA sales_schema RENAME TO sales_data_schema;

-- Change comment
ALTER SCHEMA sales_schema SET COMMENT = 'Updated schema description';

-- Drop schema
DROP SCHEMA sales_schema;

-- Show schemas
SHOW SCHEMAS IN DATABASE sales_db;

-- Use a specific schema
USE SCHEMA sales_schema;
```

---

## 3. Tables in Snowflake

### 3.1 Definition
A **Table** stores structured data in rows and columns.  
In Snowflake, tables can be **Permanent**, **Temporary**, or **Transient**.

---

### 3.2 Types of Tables
| Table Type | Description | Example Use |
|------------|-------------|-------------|
| **Permanent** | Default table type; data stored until deleted; protected by Time Travel. | Production data |
| **Transient** | No Fail-safe; lower storage cost; shorter Time Travel. | Staging data |
| **Temporary** | Exists only for session duration; no Fail-safe. | Scratch data |

---

### 3.3 Ways to Create a Table

#### Method 1 — Basic Permanent Table
```sql
CREATE TABLE sales_table (
    order_id INT,
    order_date DATE,
    amount NUMBER(10,2)
);
```

#### Method 2 — Transient Table
```sql
CREATE TRANSIENT TABLE temp_sales (
    order_id INT,
    amount NUMBER(10,2)
);
```

#### Method 3 — Temporary Table
```sql
CREATE TEMPORARY TABLE session_data (
    session_id STRING,
    user_id INT
);
```

#### Method 4 — Create from Select
```sql
CREATE TABLE sales_2024 AS
SELECT * FROM sales_table WHERE YEAR(order_date) = 2024;
```

#### Method 5 — If Not Exists
```sql
CREATE TABLE IF NOT EXISTS sales_table (
    order_id INT,
    order_date DATE,
    amount NUMBER(10,2)
);
```

---

### 3.4 Other Operations on Tables
```sql
-- Add column
ALTER TABLE sales_table ADD COLUMN customer_id INT;

-- Drop column
ALTER TABLE sales_table DROP COLUMN customer_id;

-- Rename table
ALTER TABLE sales_table RENAME TO sales_table_old;

-- Drop table
DROP TABLE sales_table;

-- Truncate table
TRUNCATE TABLE sales_table;

-- Show tables
SHOW TABLES IN SCHEMA sales_schema;

-- Use a table
SELECT * FROM sales_table;
```

---

## 4. Incremental Learning Flow for Students

1. **Start with Database:**
   - Create a database
   - Rename it
   - Drop it
2. **Move to Schema:**
   - Create schema in that database
   - Rename it
   - Drop it
3. **Progress to Tables:**
   - Create different types of tables (Permanent, Transient, Temporary)
   - Insert sample data
   - Alter tables (add/remove columns)
   - Drop tables
4. **Real-World Mini Project:**
   - Create a `sales_db` with `sales_schema`
   - Add a `sales_table`
   - Load sample sales data
   - Create yearly archive tables using `CREATE TABLE AS SELECT`

---

## 5. Best Practices
- Use **`IF NOT EXISTS`** to avoid errors in automation.
- Organize objects logically: Database → Schema → Table.
- Choose the right table type based on data retention needs.
- Always set **comments** for clarity in collaborative environments.
