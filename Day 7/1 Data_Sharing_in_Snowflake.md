# 🧊 Data Sharing in Snowflake

## 📘 Table of Contents

1.  Introduction
2.  Why Data Sharing is Important
3.  Key Concepts in Snowflake Data Sharing
4.  Types of Data Sharing
5.  Secure Data Sharing Architecture
6.  Step-by-Step Demo --- Provider & Consumer
7.  Cross-Cloud & Cross-Region Sharing\
8.  Reader Accounts
9.  Data Clean Room (Advanced)
10. Best Practices
11. Limitations
12. Summary

------------------------------------------------------------------------

## 1️⃣ Introduction

**Data Sharing** in **Snowflake** allows you to **share live,
ready-to-query data** with other Snowflake accounts **without copying or
moving** the data.

> 🧠 It's *instant, secure, and zero-copy* --- meaning shared data
> doesn't consume extra storage.

------------------------------------------------------------------------

## 2️⃣ Why Data Sharing is Important

  -----------------------------------------------------------------------
  Scenario                              Benefit
  ------------------------------------- ---------------------------------
  Partner collaboration                 Share sales, inventory, or
                                        analytics data securely

  Vendor reporting                      Give suppliers real-time insights
                                        without data exports

  Multi-region enterprise               Centralize data once and share
                                        globally

  Data monetization                     Sell datasets on Snowflake
                                        Marketplace
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3️⃣ Key Concepts

  -----------------------------------------------------------------------
  Term                  Description
  --------------------- -------------------------------------------------
  **Provider**          The Snowflake account that owns and shares the
                        data

  **Consumer**          The account that receives and accesses shared
                        data

  **Share**             A Snowflake object that defines what data is
                        shared

  **Reader Account**    A managed Snowflake account created by provider
                        for users without Snowflake

  **Marketplace**       Public listing of datasets shared through
                        Snowflake Marketplace
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 4️⃣ Types of Data Sharing in Snowflake

  -----------------------------------------------------------------------
  Type                  Description
  --------------------- -------------------------------------------------
  **Direct Sharing**    Between two Snowflake accounts in the same region

  **Cross-Region        Between accounts in different Snowflake regions
  Sharing**             

  **Cross-Cloud         Between accounts hosted on different clouds (AWS,
  Sharing**             Azure, GCP)

  **Data Marketplace    Publish datasets for external organizations
  Sharing**             

  **Reader Account      For users without an existing Snowflake account
  Sharing**             
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 5️⃣ Secure Data Sharing Architecture

     ┌──────────────────────────────┐
     │        PROVIDER ACCOUNT       │
     │ ┌───────────────┐             │
     │ │ Database      │             │
     │ │ └──> SHARE    │──────────┐  │
     └──────────────────────────────┘  │
                                      ▼
     ┌──────────────────────────────┐
     │        CONSUMER ACCOUNT       │
     │ ┌───────────────┐             │
     │ │ Shared DB     │             │
     │ │ (Read-only)   │             │
     └──────────────────────────────┘

> ✅ The consumer queries shared data directly from the provider's
> storage using Snowflake's metadata pointers.

------------------------------------------------------------------------

## 6️⃣ Step-by-Step Demo --- Provider & Consumer

### 🧑‍💻 As Provider

#### Step 1: Create Database and Table

``` sql
CREATE OR REPLACE DATABASE SALES_DB;
USE DATABASE SALES_DB;

CREATE OR REPLACE TABLE SALES (
  REGION VARCHAR,
  PRODUCT VARCHAR,
  AMOUNT NUMBER
);

INSERT INTO SALES VALUES
('South', 'Laptop', 85000),
('North', 'Mobile', 60000),
('East', 'Tablet', 30000);
```

#### Step 2: Create a Share

``` sql
CREATE OR REPLACE SHARE SALES_SHARE;
```

#### Step 3: Grant Access to the Share

``` sql
GRANT USAGE ON DATABASE SALES_DB TO SHARE SALES_SHARE;
GRANT USAGE ON SCHEMA SALES_DB.PUBLIC TO SHARE SALES_SHARE;
GRANT SELECT ON TABLE SALES_DB.PUBLIC.SALES TO SHARE SALES_SHARE;
```

#### Step 4: Add Consumer Account

``` sql
ALTER SHARE SALES_SHARE ADD ACCOUNT = '<consumer_account_locator>';
```

> 🔍 You can get the consumer's account locator from their Snowflake
> URL, e.g.,\
> `xy12345.ap-south-1.azure.snowflakecomputing.com` → account locator is
> **xy12345**

------------------------------------------------------------------------

### 👩‍💻 As Consumer

#### Step 5: View Shared Data

``` sql
SHOW SHARES;
```

#### Step 6: Create a Database from Shared Data

``` sql
CREATE DATABASE SHARED_SALES_DB FROM SHARE <provider_account_locator>.SALES_SHARE;
```

#### Step 7: Query Shared Data

``` sql
SELECT * FROM SHARED_SALES_DB.PUBLIC.SALES;
```

> ⚡ No data is copied! You are querying directly from the provider's
> data storage.

------------------------------------------------------------------------

## 7️⃣ Cross-Cloud & Cross-Region Sharing

If accounts are on **different regions or clouds**, use **Snowflake's
"Replication & Sharing" feature**:

``` sql
ALTER SHARE SALES_SHARE ENABLE REPLICATION TO ACCOUNTS IN REGION 'AWS_US_EAST_1';
```

> 🌍 This feature ensures seamless global data sharing while maintaining
> governance.

------------------------------------------------------------------------

## 8️⃣ Reader Accounts

If the recipient does not have a Snowflake account:

``` sql
CREATE MANAGED ACCOUNT retail_reader
  ADMIN_NAME = 'reader_admin'
  ADMIN_PASSWORD = 'Reader@123';
  
GRANT IMPORTED PRIVILEGES ON SHARE SALES_SHARE TO ACCOUNT retail_reader;
```

✅ A **Reader Account** is a lightweight Snowflake account managed by
the provider --- ideal for vendors or clients without Snowflake
subscriptions.

------------------------------------------------------------------------

## 9️⃣ Data Clean Room (Advanced Concept)

> Introduced with **Snowflake Cortex AI and Secure Join**, enabling
> **multi-party data collaboration** without exposing raw data.

------------------------------------------------------------------------

## 🔟 Best Practices

  -----------------------------------------------------------------------
  Area                Best Practice
  ------------------- ---------------------------------------------------
  Security            Use roles and schema-level grants

  Naming              Follow consistent naming conventions for shares

  Governance          Track usage via ACCOUNT_USAGE views

  Performance         Monitor consumer queries

  Cost                Data sharing itself is **free**, but compute cost
                      applies to consumers
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## ⚠️ 11. Limitations

-   Shared data is **read-only** for consumers
-   No DML operations (INSERT, UPDATE, DELETE) allowed
-   Time Travel and Fail-Safe features are not shared
-   Consumer cannot modify metadata (e.g., create views in shared
    schema)

------------------------------------------------------------------------

## ✅ 12. Summary

| **Concept**                  | **Provider** | **Consumer** |
|------------------------------|--------------|--------------|
| **Create database/table**    | ✅            | ❌            |
| **Query shared data**        | ✅            | ✅            |
| **Edit shared data**         | ✅            | ❌            |
| **Create Share**             | ✅            | ❌            |
| **Create Database from Share** | ❌          | ✅            |

------------------------------------------------------------------------

## 📊 Real-World Example

**Scenario:**\
Hexa Retail Pvt. Ltd. (Provider) wants to share its "Monthly_Sales" data
with Capgemini India (Consumer).

  Step   Description
  ------ -----------------------------------------------------------
  1      Hexa creates `SALES_SHARE` in its Snowflake account
  2      Grants access to Capgemini's account locator
  3      Capgemini creates `Hexa_Sales_DB` from that share
  4      Capgemini runs live analytics --- no data transfer needed

------------------------------------------------------------------------

## 🧠 Quick Recap Diagram

           ┌───────────────────────────────┐
           │       Hexa Retail Pvt Ltd      │
           │        (Provider Account)       │
           └──────────────┬─────────────────┘
                          │ Secure Share
                          ▼
           ┌───────────────────────────────┐
           │        Capgemini India         │
           │       (Consumer Account)        │
           └───────────────────────────────┘
