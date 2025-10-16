# 🧊 Creating Warehouses, Roles, and Users in Snowflake

This session demonstrates how to create **virtual warehouses**, **roles**, and **users** for two distinct teams — **Data Scientists** and **DBAs** — using Snowflake’s Role-Based Access Control (RBAC) model.

---

## 🧱 1️⃣ Creating Virtual Warehouses

A **virtual warehouse** is a compute engine in Snowflake used to run SQL queries. It can scale up/down and suspend automatically to save cost.

### ✅ For Data Scientists
```sql
CREATE WAREHOUSE DS_WH 
WITH WAREHOUSE_SIZE = 'SMALL'
WAREHOUSE_TYPE = 'STANDARD' 
AUTO_SUSPEND = 300 
AUTO_RESUME = TRUE 
MIN_CLUSTER_COUNT = 1 
MAX_CLUSTER_COUNT = 1 
SCALING_POLICY = 'STANDARD';
```
**Explanation:**
- `WAREHOUSE_SIZE = 'SMALL'` → Moderate compute (2 credits/hour).  
- `AUTO_SUSPEND = 300` → Suspends after 5 minutes of inactivity.  
- `AUTO_RESUME = TRUE` → Automatically starts when a query runs.  
- `MIN/MAX_CLUSTER_COUNT = 1` → One cluster only (no scaling).  
- Used by **Data Scientists** for analytics workloads.

---

### ✅ For DBAs
```sql
CREATE WAREHOUSE DBA_WH 
WITH WAREHOUSE_SIZE = 'XSMALL'
WAREHOUSE_TYPE = 'STANDARD' 
AUTO_SUSPEND = 300 
AUTO_RESUME = TRUE 
MIN_CLUSTER_COUNT = 1 
MAX_CLUSTER_COUNT = 1 
SCALING_POLICY = 'STANDARD';
```
**Explanation:**
- `XSMALL` → Lower compute cost (1 credit/hour).  
- Ideal for light DBA or metadata queries.

---

## 🧑‍💻 2️⃣ Creating Roles and Granting Warehouse Access
Roles define what a user is allowed to do.

```sql
CREATE ROLE DATA_SCIENTIST;
GRANT USAGE ON WAREHOUSE DS_WH TO ROLE DATA_SCIENTIST;

CREATE ROLE DBA;
GRANT USAGE ON WAREHOUSE DBA_WH TO ROLE DBA;
```
**Explanation:**
- Each role is tied to a specific warehouse.  
- Prevents Data Scientists from using DBA resources and vice versa.

---

## 👥 3️⃣ Creating Users and Assigning Roles

Each user has login credentials, a default role, and a default warehouse.

### 🧠 Data Scientists
```sql
CREATE USER DS1 PASSWORD = 'DS1' LOGIN_NAME = 'DS1' 
  DEFAULT_ROLE = 'DATA_SCIENTIST' DEFAULT_WAREHOUSE = 'DS_WH' MUST_CHANGE_PASSWORD = FALSE;

CREATE USER DS2 PASSWORD = 'DS2' LOGIN_NAME = 'DS2' 
  DEFAULT_ROLE = 'DATA_SCIENTIST' DEFAULT_WAREHOUSE = 'DS_WH' MUST_CHANGE_PASSWORD = FALSE;

CREATE USER DS3 PASSWORD = 'DS3' LOGIN_NAME = 'DS3' 
  DEFAULT_ROLE = 'DATA_SCIENTIST' DEFAULT_WAREHOUSE = 'DS_WH' MUST_CHANGE_PASSWORD = FALSE;

GRANT ROLE DATA_SCIENTIST TO USER DS1;
GRANT ROLE DATA_SCIENTIST TO USER DS2;
GRANT ROLE DATA_SCIENTIST TO USER DS3;
```
**Explanation:**
- Creates three data-science users.  
- Each automatically connects with role **DATA_SCIENTIST** and warehouse **DS_WH**.  
- The `GRANT ROLE` step activates permissions.

---

### 🧰 DBAs
```sql
CREATE USER DBA1 PASSWORD = 'DBA1' LOGIN_NAME = 'DBA1' 
  DEFAULT_ROLE = 'DBA' DEFAULT_WAREHOUSE = 'DBA_WH' MUST_CHANGE_PASSWORD = FALSE;

CREATE USER DBA2 PASSWORD = 'DBA2' LOGIN_NAME = 'DBA2' 
  DEFAULT_ROLE = 'DBA' DEFAULT_WAREHOUSE = 'DBA_WH' MUST_CHANGE_PASSWORD = FALSE;

GRANT ROLE DBA TO USER DBA1;
GRANT ROLE DBA TO USER DBA2;
```
**Explanation:**
- Two DBA accounts for administrative tasks, each using the smaller `DBA_WH` warehouse.

---

## 🧹 4️⃣ Cleaning Up (Dropping Demo Objects)
When the demo ends, clean the environment:
```sql
DROP USER DBA1;
DROP USER DBA2;
DROP USER DS1;
DROP USER DS2;
DROP USER DS3;

DROP ROLE DATA_SCIENTIST;
DROP ROLE DBA;

DROP WAREHOUSE DS_WH;
DROP WAREHOUSE DBA_WH;
```
**Explanation:**
- Ensures Snowflake resources (and credits) aren’t consumed unnecessarily.  
- Keeps the environment tidy for next labs.

---

## 🧭 Key Teaching Points

| Concept | Description |
|----------|--------------|
| **Warehouse** | Compute engine to execute queries. |
| **Role** | Defines privileges and access (RBAC model). |
| **User** | Individual account linked to a role and warehouse. |
| **Auto-Suspend/Resume** | Saves cost by pausing compute automatically. |
| **Separation of Duties** | Ensures governance and cost optimization by isolating resources. |

---

✨ **Summary:**  
This exercise demonstrates **Snowflake’s RBAC model** in action — creating dedicated compute warehouses for different teams, granting appropriate access, and assigning roles to users. It’s a foundational step in enterprise-level Snowflake administration and governance.
