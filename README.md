# 🏥 Executive Healthcare Analytics & Operational Dashboard

An end-to-end Healthcare Data Analytics project designed for hospital executive leadership and clinical operations managers. This repository showcases a complete analytics pipeline integrating **SQL** for database cleaning and complex business intelligence queries, **Python** for exploratory data profiling, and **Power BI** for interactive decision-support dashboards.

---

## 📸 Dashboard Visuals

![Healthcare Dashboard Overview](./dashboard-overview.png)

---

## 🛠️ Tools & Technologies

* **SQL (MySQL):** Database Cleaning, Deduplication, Window Functions, Business Logic Aggregations
* **Python (Pandas / NumPy / Seaborn / Matplotlib):** Comprehensive EDA, Distribution Profiling, Correlation Analysis
* **Power BI Desktop:** Interactive Visualization, Data Modeling, DAX Measures, Operational Metrics
* **Power Query:** Schema Structuring & Transformation Pipeline
* **GitHub:** Portfolio Hosting, Asset Management & Documentation

---

## 🧹 Complete SQL Data Cleaning Script

Raw patient records were cleaned and transformed in MySQL to fix string formatting, calculate stay duration, handle negative billing amounts, deduplicate via CTE/Window functions, and create synthetic primary/foreign keys:

```sql
USE healthcare_db;

-- 1. Standardize Patient Name Formatting (Title Case)
UPDATE healthcare_dataset
SET Name = CONCAT(
    UPPER(SUBSTRING(Name, 1, 1)), 
    LOWER(SUBSTRING(Name, 2, LOCATE(' ', Name) - 1)), 
    ' ', 
    UPPER(SUBSTRING(Name, LOCATE(' ', Name) + 1, 1)), 
    LOWER(SUBSTRING(Name, LOCATE(' ', Name) + 2))
)
WHERE Name LIKE '% %';

-- 2. Round Billing Amounts to 2 Decimal Places
UPDATE healthcare_dataset
SET `billing amount` = ROUND(`billing amount`, 2);

-- 3. Calculate Stay Duration in Days
ALTER TABLE healthcare_dataset ADD COLUMN Stay_Duration_Days INT;

UPDATE healthcare_dataset
SET Stay_Duration_Days = DATEDIFF(
    STR_TO_DATE(`Discharge Date`, '%Y-%m-%d'),
    STR_TO_DATE(`Date of Admission`, '%Y-%m-%d')
);

-- 4. Correct Negative Billing Amounts
UPDATE healthcare_dataset
SET `Billing Amount` = ABS(`Billing Amount`)
WHERE `Billing Amount` < 0;

-- 5. Deduplicate Records Using Window Functions (ROW_NUMBER)
CREATE TABLE healthcare_clean AS
SELECT * FROM (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY Name, `Date of Admission`, `Medical Condition`
               ORDER BY `Discharge Date` DESC
           ) AS row_num
    FROM healthcare_dataset
) AS temp
WHERE row_num = 1;

ALTER TABLE healthcare_clean DROP COLUMN row_num;
DROP TABLE healthcare_dataset;
RENAME TABLE healthcare_clean TO healthcare_dataset;

-- 6. Add Calculated Metrics (Daily Billing Amount)
ALTER TABLE healthcare_dataset ADD COLUMN Daily_Billing_Amount DECIMAL(10, 2);

UPDATE healthcare_dataset
SET Daily_Billing_Amount = ROUND(`Billing Amount` / NULLIF(Stay_Duration_Days, 0), 2);

-- 7. Generate Primary Key (Visit_ID) & Surrogate Key (Patient_ID)
ALTER TABLE healthcare_dataset 
ADD COLUMN Visit_ID INT AUTO_INCREMENT PRIMARY KEY FIRST,
ADD COLUMN Patient_ID INT AFTER Visit_ID;

ALTER TABLE healthcare_dataset AUTO_INCREMENT = 1001;

UPDATE healthcare_dataset h
JOIN (
    SELECT Name, DENSE_RANK() OVER (ORDER BY Name) + 4999 AS p_id
    FROM healthcare_dataset
) AS src ON h.Name = src.Name
SET h.Patient_ID = src.p_id;
```

---

## 📊 Complete SQL Analytics Insights (All Queries)

Below are the 7 key SQL analytical queries executed to support clinical decision-making:

```sql
-- Insight 1: Normal Result Rate per Condition and Medication
SELECT 
    `Medical Condition`,
    `Medication`,
    COUNT(*) AS Total_Patients,
    ROUND(AVG(CASE WHEN `Test Results` = 'Normal' THEN 100.0 ELSE 0.0 END), 1) AS Normal_Result_Rate_Pct
FROM hospital_data
GROUP BY `Medical Condition`, `Medication`
ORDER BY Normal_Result_Rate_Pct DESC
LIMIT 5;

-- Insight 2: Emergency Patients Breakdown by Blood Type
SELECT 
    `Blood Type`,
    COUNT(*) AS Emergency_Patients_Count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS Share_Pct
FROM hospital_data
WHERE `Admission Type` = 'Emergency'
GROUP BY `Blood Type`
ORDER BY Emergency_Patients_Count DESC;

-- Insight 3: Emergency Conditions with Highest Abnormal Risk
SELECT 
    `Medical Condition`,
    COUNT(*) AS Emergency_Patients,
    ROUND(AVG(CASE WHEN `Test Results` = 'Abnormal' THEN 100.0 ELSE 0.0 END), 1) AS Abnormal_Risk_Pct,
    ROUND(AVG(`Stay_Duration_Days`), 1) AS Avg_Stay_Days
FROM hospital_data
WHERE `Admission Type` = 'Emergency'
GROUP BY `Medical Condition`
ORDER BY Abnormal_Risk_Pct DESC;

-- Insight 4: High-Duration Inpatients (> 20 Days Stay)
SELECT 
    `Medical Condition`,
    COUNT(*) AS Long_Stay_Patients,
    ROUND(AVG(`Stay_Duration_Days`), 1) AS Avg_Stay_Days
FROM hospital_data
WHERE `Stay_Duration_Days` > 20
GROUP BY `Medical Condition`
ORDER BY Long_Stay_Patients DESC;

-- Insight 5: Revenue Performance by Admission Type
SELECT 
    `Admission Type`,
    ROUND(AVG(`Stay_Duration_Days`), 1) AS Avg_Stay_Days,
    ROUND(AVG(`Billing Amount` * 1.0 / `Stay_Duration_Days`), 0) AS Daily_Revenue_USD
FROM hospital_data
GROUP BY `Admission Type`
ORDER BY Daily_Revenue_USD DESC;

-- Insight 6: Senior vs. Non-Senior Demographic & Risk Analysis
SELECT 
    CASE 
        WHEN `Age` > 60 THEN 'Senior (>60)' 
        ELSE 'Non-Senior (<=60)' 
    END AS Demographic_Group,
    COUNT(*) AS Total_Patients,
    ROUND(AVG(`Billing Amount`), 0) AS Avg_Billing_USD,
    ROUND(AVG(`Stay_Duration_Days`), 1) AS Avg_Stay_Days,
    ROUND(AVG(CASE WHEN `Test Results` = 'Abnormal' THEN 100.0 ELSE 0.0 END), 1) AS Abnormal_Results_Pct
FROM hospital_data
GROUP BY Demographic_Group
ORDER BY Demographic_Group DESC;

-- Insight 7: Gender Risk Distribution Across Medical Conditions
SELECT 
    `Medical Condition`,
    `Gender`,
    COUNT(*) AS Total_Patients,
    ROUND(AVG(CASE WHEN `Test Results` = 'Abnormal' THEN 100.0 ELSE 0.0 END), 1) AS Abnormal_Rate_Pct
FROM hospital_data
GROUP BY `Medical Condition`, `Gender`
ORDER BY `Medical Condition`, `Gender`;
```

---

## 🐍 Python Exploratory Data Analysis (EDA) Pipeline

Key excerpts from the Python profiling script (`pandas`, `seaborn`, `numpy`):

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# 1. Load Cleaned Dataset
df = pd.read_csv("healthcare_cleaned.csv", sep="\t")

# 2. Date Formatting
df["Date of Admission"] = pd.to_datetime(df["Date of Admission"], format="mixed")
df["Discharge Date"] = pd.to_datetime(df["Discharge Date"], format="mixed")

# 3. High-Cost Outlier Segmentation (90th Percentile)
high_cost_threshold = df["Billing Amount"].quantile(0.90)
df["High Cost Status"] = np.where(df["Billing Amount"] >= high_cost_threshold, "High Cost", "Regular")

# 4. Correlation Matrix
numeric_cols = ["Age", "Billing Amount", "Room Number", "Stay_Duration_Days", "Daily_Billing_Amount"]
correlation_matrix = df[numeric_cols].corr()
print(correlation_matrix)

# 5. Admission Trends by Month & Year
df["Admission Year"] = df["Date of Admission"].dt.year
df["Admission Month"] = df["Date of Admission"].dt.month
monthly_admissions = df["Admission Month"].value_counts().sort_index()
```

---

## 📑 Project Files

* 📂 **`Healthcare_Analytics_Dashboard.pbix`**: Main Power BI workbook containing data models, visual pages, and DAX measures.
* 📄 **`Healthcare_Data_Analysis_Documentation.pdf`**: Full technical documentation report with step-by-step logic.
* 🖼️ **`dashboard-overview.png`**: Primary executive view screenshot.

---

## 👤 Author & Contact

**Omar Mohamed Zainhom Farghali**  
*Data Analysis Specialist | Computer Science Student at Cairo University*

* **GitHub:** [@Omar6655](https://github.com/Omar6655)
* **Project Repository:** [Healthcare_Dashboard](https://github.com/Omar6655/Healthcare_Dashboard)
