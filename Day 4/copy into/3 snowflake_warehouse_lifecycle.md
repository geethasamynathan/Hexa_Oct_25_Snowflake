# Snowflake: Warehouse Lifecycle Management
## 1. Create Warehouse
```sql
CREATE WAREHOUSE my_wh
WITH
  WAREHOUSE_SIZE = 'SMALL'
  WAREHOUSE_TYPE = 'STANDARD'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;
```
**Explanation:**
- **`WAREHOUSE_SIZE`**: Determines compute capacity (`XSMALL` → `6XLARGE`).
- **`WAREHOUSE_TYPE`**: `STANDARD` (general workloads) or `SNOWPARK-OPTIMIZED` for Snowpark.
- **`AUTO_SUSPEND`**: Suspends after inactivity (in seconds).
- **`AUTO_RESUME`**: Starts automatically when a query runs.
- **`INITIALLY_SUSPENDED`**: Starts in suspended mode after creation.

---

## 2. Alter Warehouse
```sql
ALTER WAREHOUSE my_wh
SET WAREHOUSE_SIZE = 'MEDIUM',
    AUTO_SUSPEND = 120,
    AUTO_RESUME = FALSE;
```
**Explanation:**
- **Purpose:** Modify existing warehouse settings without recreating it.
- **Common Changes:**
  - Resize to handle more/less workload.
  - Adjust auto-suspend/resume settings.
- **Note:** Changes take effect immediately.

---

## 3. Suspend Warehouse
```sql
ALTER WAREHOUSE my_wh SUSPEND 
```
**Explanation:**
- Pauses compute billing immediately.
- Useful for manual cost control when a warehouse isn’t needed.

---

## 4. Resume Warehouse
```sql
ALTER WAREHOUSE  my_wh RESUME;
```
**Explanation:**
- Starts a suspended warehouse so queries can run.
- If **`AUTO_RESUME`** is enabled, Snowflake will do this automatically.

---

## 5. Drop Warehouse
```sql
DROP WAREHOUSE my_wh;
```
or safely:
```sql
DROP WAREHOUSE IF EXISTS my_wh;
```
**Explanation:**
- Deletes the compute resource but **does not delete data**.
- Use **`IF EXISTS`** to avoid errors in automation scripts.

---

## 6. Check Warehouse Status
```sql
SHOW WAREHOUSES;
```
**Explanation:**
- Lists all warehouses, sizes, states, and usage statistics.

---

## 7. Real-World Example: ETL Project Lifecycle
**Scenario:** You need a temporary ETL warehouse for 1 week of heavy data loading.

```sql
-- Step 1: Create
CREATE WAREHOUSE etl_wh
WITH WAREHOUSE_SIZE = 'LARGE'
     AUTO_SUSPEND = 300
     AUTO_RESUME = TRUE
     INITIALLY_SUSPENDED = TRUE;

-- Step 2: Resize during peak load
ALTER WAREHOUSE etl_wh SET WAREHOUSE_SIZE = 'XLARGE';

-- Step 3: Suspend manually after big load
SUSPEND WAREHOUSE etl_wh;

-- Step 4: Resume for next batch
RESUME WAREHOUSE etl_wh;

-- Step 5: Drop after project completion
DROP WAREHOUSE IF EXISTS etl_wh;
```

---

## 8. Best Practices
- Use **separate warehouses** for ETL, reporting, and ML workloads to prevent resource contention.
- Always enable **`AUTO_SUSPEND`** to save costs.
- Scale up for short bursts instead of keeping large warehouses running 24/7.
- Monitor usage with:
```sql
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY;
```
