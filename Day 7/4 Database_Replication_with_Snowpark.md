# 🧊 Configure Database Replication for Disaster Recovery (DR) with Snowpark

## 🎯 Objective

To set up **Database Replication** between two Snowflake accounts
(Primary & Secondary) for **Disaster Recovery (DR)**, and validate it
using **Snowpark**.

------------------------------------------------------------------------

## 🧠 Concept Overview

### 🔹 What is Database Replication?

Database Replication allows you to **automatically copy databases,
schemas, and objects** between Snowflake accounts or regions.\
It ensures **business continuity** in case the primary region becomes
unavailable.

### 🔹 Why Use It?

  -----------------------------------------------------------------------
  Purpose                        Explanation
  ------------------------------ ----------------------------------------
  **Disaster Recovery (DR)**     Maintain a backup copy of data in
                                 another region/cloud

  **Geo-redundancy**             Store data copies in multiple regions
                                 for resilience

  **Analytics distribution**     Share consistent data with regional
                                 analytics teams

  **Failover Testing**           Simulate outages to ensure readiness
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 🏗️ Architecture Diagram

    ┌──────────────────────────────────────┐
    │ Primary Account (Region A)           │
    │ Database: SALES_DB                   │
    │ Warehouse: COMPUTE_WH                │
    └──────────────┬───────────────────────┘
                   │ Replication
                   ▼
    ┌──────────────────────────────────────┐
    │ Secondary Account (Region B)         │
    │ Replica: SALES_DB_REPLICA            │
    │ Read-only until failover             │
    └──────────────────────────────────────┘

------------------------------------------------------------------------

## ⚙️ Step-by-Step Configuration

### 🔹 Step 1. Prerequisites

  Requirement   Details
  ------------- --------------------------------------------
  Role          Must use **ACCOUNTADMIN**
  Edition       **Enterprise Edition** or higher
  Feature       **Replication & Failover** enabled
  Permissions   USAGE and REPLICATION privileges
  Snowpark      Installed in your Python/Scala environment

------------------------------------------------------------------------

### 🔹 Step 2. Identify Primary and Secondary Accounts

  Environment   Region                Account Locator
  ------------- --------------------- -----------------
  Primary       Azure East India      `XY12345`
  Secondary     Azure Central India   `AB67890`

> Both must belong to the same organization.

------------------------------------------------------------------------

### 🔹 Step 3. Enable Replication (Primary Side)

``` sql
ALTER ACCOUNT SET REPLICATION_ALLOWED_TO_ACCOUNTS = ('AB67890');
```

✅ Allows the secondary account to receive replicated data.

------------------------------------------------------------------------

### 🔹 Step 4. Create the Source Database

``` sql
CREATE OR REPLACE DATABASE SALES_DB;
USE DATABASE SALES_DB;

CREATE OR REPLACE TABLE SALES (
  ORDER_ID VARCHAR,
  REGION VARCHAR,
  AMOUNT NUMBER(10,2)
);

INSERT INTO SALES VALUES
('O1001', 'South', 85000),
('O1002', 'North', 72000),
('O1003', 'West', 54000);
```

------------------------------------------------------------------------

### 🔹 Step 5. Enable Replication for the Database

``` sql
ALTER DATABASE SALES_DB ENABLE REPLICATION;
```

Marks `SALES_DB` as a **replicable database**.

------------------------------------------------------------------------

### 🔹 Step 6. Create a Replication Group (Optional)

``` sql
CREATE REPLICATION GROUP SALES_RG
  OBJECT_TYPES = DATABASE
  DATABASES = (SALES_DB)
  ALLOWED_TO_ACCOUNTS = ('AB67890');
```

> Used to replicate multiple databases together.

------------------------------------------------------------------------

### 🔹 Step 7. Create the Replica in the Secondary Account

Switch to **secondary account** (`AB67890`):

``` sql
CREATE DATABASE SALES_DB_REPLICA AS REPLICA OF XY12345.SALES_DB;
```

✅ Snowflake automatically replicates all data and metadata.

------------------------------------------------------------------------

### 🔹 Step 8. Validate Replication

``` sql
SHOW DATABASES;
SELECT * FROM SALES_DB_REPLICA.PUBLIC.SALES;
```

✅ Confirms data visibility. Replica is read-only.

------------------------------------------------------------------------

### 🔹 Step 9. Schedule Automatic Replication

``` sql
ALTER DATABASE SALES_DB_REPLICA REFRESH;
```

Or schedule periodic refresh:

``` sql
CREATE TASK AUTO_REFRESH_REPLICA
  WAREHOUSE = COMPUTE_WH
  SCHEDULE = '5 MINUTE'
AS
  ALTER DATABASE SALES_DB_REPLICA REFRESH;
```

------------------------------------------------------------------------

### 🔹 Step 10. Perform Failover (Simulate Disaster)

``` sql
ALTER DATABASE SALES_DB_REPLICA PRIMARY;
```

✅ Promotes the replica to **primary** after failover.

------------------------------------------------------------------------

## 🧠 Validate Using Snowpark (Python Example)

### Step 1: Setup Snowpark Session

``` python
from snowflake.snowpark import Session

connection_parameters = {
    "account": "ab67890",
    "user": "dr_user",
    "password": "YourPassword",
    "role": "ACCOUNTADMIN",
    "warehouse": "COMPUTE_WH",
    "database": "SALES_DB_REPLICA",
    "schema": "PUBLIC"
}

session = Session.builder.configs(connection_parameters).create()
```

### Step 2: Read Data

``` python
sales_df = session.table("SALES")
sales_df.show()
```

### Step 3: Validate Record Count

``` python
count_df = sales_df.count()
print(f"Total Records in Replica: {count_df}")
```

### Step 4: Validate Aggregations

``` python
summary = sales_df.group_by("REGION").agg({"AMOUNT": "sum"})
summary.show()
```

✅ Confirms data integrity post replication.

------------------------------------------------------------------------

### 🔹 Step 11. Reverse Replication (Optional)

``` sql
ALTER DATABASE SALES_DB_REPLICA ENABLE REPLICATION TO ACCOUNTS ('XY12345');
```

Allows the former primary to receive updates once back online.

------------------------------------------------------------------------

## 🔐 Security & Governance

  Practice               Description
  ---------------------- --------------------------------------------------
  Restrict replication   Allow only trusted accounts
  Encryption             All traffic is encrypted by Snowflake
  Audit                  Use `ACCOUNT_USAGE.REPLICATION_DATABASE_HISTORY`
  DR Testing             Run periodic failover tests
  Automate               Use Snowpark scripts for validation

------------------------------------------------------------------------

## 📊 End-to-End Summary

  Phase      Action                         Account
  ---------- ------------------------------ -----------
  Setup      Enable replication on source   Primary
  Enable     Allow replication to target    Primary
  Create     Create replica                 Secondary
  Validate   Query replicated data          Secondary
  Failover   Promote replica                Secondary
  Reverse    Re-enable replication          Primary

------------------------------------------------------------------------

## ✅ Example Output

  ORDER_ID   REGION   AMOUNT
  ---------- -------- --------
  O1001      South    85000
  O1002      North    72000
  O1003      West     54000

✅ Same dataset visible across both regions ensures successful
replication.

------------------------------------------------------------------------

## 🚀 Benefits

  Benefit                    Description
  -------------------------- ------------------------------------
  **Zero Data Loss**         Near real-time replication (RPO=0)
  **Quick Recovery**         Failover within minutes (RTO\<5)
  **Snowpark Integration**   Automate validation and alerts
  **No ETL Needed**          Fully managed Snowflake feature

------------------------------------------------------------------------

## 🏁 Conclusion

Snowflake's **Database Replication + Snowpark** ensures a reliable,
automated DR strategy:\
- Replicate seamlessly across regions\
- Failover instantly\
- Validate data programmatically using Snowpark
