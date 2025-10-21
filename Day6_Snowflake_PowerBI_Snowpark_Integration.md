# 🧭 Day 6 — Advanced Analytics and Integration with Power BI and Snowpark

## 🎯 Objective
By the end of this module, learners will be able to:

1. Use **Snowpark (Python & Scala)** to transform and analyze data in Snowflake directly from Azure Databricks.  
2. Connect **Power BI** to Snowflake using **DirectQuery** for live dashboards.  
3. Train and deploy **machine-learning models** using **Snowpark + Databricks**.  
4. Enable **Azure Synapse Link** for unified analytical queries.  
5. Build **real-time analytics pipelines** using **Snowpark UDFs**, **Azure ML**, and **Power BI**.

---

## 🧩 Topics Overview

### 1️⃣ Snowpark for Python and Scala in Databricks
**Snowpark** is Snowflake’s developer framework for running data engineering and ML logic **inside Snowflake compute** (without moving data).

| Feature | Description |
|----------|--------------|
| Execution | Runs inside Snowflake’s virtual warehouse |
| Languages | Python, Scala, Java, and SQL |
| Core Object | `Session` (connection to Snowflake) |
| Data Representation | `DataFrame` API (similar to Spark) |
| Deployment | Register as **Stored Procedures** or **UDFs** |

**Code Example – Snowpark Python in Databricks**
```python
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col

# Connection Parameters
connection_parameters = {
    "account": "myorg-account",
    "user": "geetha",
    "password": "MySnowflakecred1",
    "role": "ACCOUNTADMIN",
    "warehouse": "COMPUTE_WH",
    "database": "DEMO_DB",
    "schema": "PUBLIC"
}

# Start a Snowpark Session
session = Session.builder.configs(connection_parameters).create()

# Query Snowflake table
df = session.table("SALES")

# Transform data
result = df.filter(col("PROFIT") > 5000).group_by("CATEGORY").agg({"SALES": "sum"})

result.show()
```

✅ Runs entirely in Snowflake’s compute engine—no data transfer overhead.

---

### 2️⃣ Power BI Integration with Snowflake
Power BI connects to Snowflake using either **DirectQuery** or **Import** mode.

#### 🔹 Steps to Connect
1. Open **Power BI Desktop → Home → Get Data → Snowflake**  
2. Enter the **Server**:  
   ```
   <your_account>.snowflakecomputing.com
   ```  
   and the **Warehouse** name (e.g., `COMPUTE_WH`).
3. Choose the **authentication method** (username/password or SSO).  
4. Select **DirectQuery** for real-time data.  
5. Build visuals—Sales by Region, Profit by Category, etc.  
6. Publish to **Power BI Service** for scheduled refresh or live dashboards.

---

### 3️⃣ Machine Learning with Snowpark and Databricks
**Workflow**
1. Load data from Snowflake into Databricks via Snowpark.  
2. Train ML model (e.g., Random Forest Regressor).  
3. Register model in **Azure ML or MLflow**.  
4. Push predictions back to Snowflake table via Snowpark.

**Sample Code**
```python
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestRegressor
from snowflake.snowpark import Session

# Step 1: Load Snowflake data
session = Session.builder.configs(connection_parameters).create()
df = session.table("SALES").to_pandas()

# Step 2: Train model
X = df[["QUANTITY", "DISCOUNT"]]
y = df["PROFIT"]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

model = RandomForestRegressor().fit(X_train, y_train)

# Step 3: Save predictions back to Snowflake
df["PREDICTED_PROFIT"] = model.predict(X)
session.create_dataframe(df).write.save_as_table("SALES_PREDICTED", mode="overwrite")
```

---

### 4️⃣ Azure Synapse Link for Snowflake
Allows Synapse to **query Snowflake data** directly—no ETL.

**Steps**
1. In **Azure Portal**, open **Synapse Workspace → Linked Services → + New**.  
2. Choose **Snowflake** as a data source.  
3. Provide:  
   - Server Name = `<account>.snowflakecomputing.com`  
   - Warehouse = `COMPUTE_WH`  
   - Database/Schema = `DEMO_DB.PUBLIC`  
4. Test connection and publish.  
5. Use **SQL Serverless** pool to run:
   ```sql
   SELECT * FROM OPENROWSET(
       PROVIDER='Snowflake',
       CONNECTION_ID='SnowflakeConnection',
       OBJECT='DEMO_DB.PUBLIC.SALES'
   );
   ```

---

### 5️⃣ Real-Time Analytics with Azure Databricks, Snowflake, and Snowpark
**Architecture Flow**
```
Azure Event Hub → Databricks Streaming → Snowflake (staging table)
     ↓
Snowpark processing & feature engineering
     ↓
Power BI Dashboard (Live DirectQuery)
```
- Use **Structured Streaming** in Databricks to insert real-time records.  
- Snowpark UDFs process incoming events inside Snowflake.  
- Power BI visualizes live metrics.

---

### 6️⃣ Snowpark UDFs for ML and Analytics
You can write **Python UDFs** to perform in-database ML inference.

**Example – Predictive UDF**
```python
import joblib
from snowflake.snowpark.functions import udf

# Load trained model
model = joblib.load("/dbfs/models/sales_rf.pkl")

@udf
def predict_sales(qty, disc):
    return float(model.predict([[qty, disc]])[0])
```

✅ Deploy and use inside SQL:
```sql
SELECT ORDER_ID, predict_sales(QUANTITY, DISCOUNT) AS SALES_FORECAST
FROM SALES;
```

---

## 🧪 LAB EXERCISES

| Lab # | Objective | Task |
|:--|:--|:--|
| **1** | Query Snowflake with Snowpark in Databricks | Use Python Snowpark Session to filter & aggregate data |
| **2** | Build Power BI Dashboard | Connect via DirectQuery and visualize profit by category |
| **3** | Train ML Model | Use Snowpark + Databricks to predict sales trend |
| **4** | Integrate Synapse Link | Query Snowflake tables from Synapse SQL Serverless |
| **5** | Automate ML Deployment | Register model in Azure ML and call from Snowpark UDF |
| **6** | Create Real-Time Dashboard | Stream data via Event Hub → Databricks → Snowflake → Power BI |
| **7** | Develop Snowpark UDF | Implement predictive function for sales forecasting |

---

## 🧠 Case Study – Sales Forecasting for Aryan Retail
A retail company stores 10 years of sales data in Snowflake.  
They want to predict future sales and visualize real-time performance.

**Architecture**
```
Azure Databricks (Snowpark) → Train Model on Snowflake data
Azure ML → Register Model + Deploy as REST Endpoint
Snowflake UDF → Call Azure ML for Predictions
Power BI → Live Dashboard on Predicted Sales
```

**Outcome**
- 20 % improvement in forecast accuracy  
- Reduced data latency from hours to seconds  
- Unified data governance across Azure ecosystem  

---

## 📊 Conceptual Diagram

```
Power BI (Live Dashboard)
        ↑
   Snowflake ↔ Snowpark UDFs ↔ Azure ML
        ↑
Databricks (Training & Streaming)
        ↑
Azure Event Hub / Synapse Link
```

---

## 🧩 Trainer Tips
- Emphasize **zero-data-movement** analytics.  
- Compare **Snowpark vs Spark** architecture.  
- Demo both **DirectQuery** and **Import** in Power BI.  
- Let learners **view Query History** in Snowflake for performance insights.  
- Encourage reuse of **UDFs for ML inference**.  
