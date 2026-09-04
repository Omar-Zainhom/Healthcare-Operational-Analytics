# 🏥 Executive Healthcare Operational Analytics

An end-to-end Healthcare Analytics solution built for hospital executive leadership and clinical operations managers. This repository showcases a full-stack data pipeline integrating **SQL** for scalable data cleaning and business logic, **Python** for descriptive data profiling, and **Power BI** for interactive operational dashboards.

---

## 📸 Executive Dashboard Overview

![Healthcare Dashboard Overview](./dashboard-overview.png)

---

## 🛠️ Data Architecture & Tech Stack

| Phase | Technology | Key Operations |
| :--- | :--- | :--- |
| **Data Cleaning** | MySQL | String standardization, Date math, Deduplication via CTEs/Window functions, Key generation |
| **Exploratory Analysis** | Python (`pandas`, `seaborn`) | Demographic distribution, Outlier detection, Correlation analysis |
| **Data Visualization** | Power BI, DAX, Power Query | Interactive filtering, DAX measures, Operational KPIs, Drill-down analysis |
| **Documentation** | Technical PDF Report | Comprehensive code scripts, business queries, and visualization breakdowns |

---

## 🧹 Data Pipeline Overview

The raw healthcare records were structured and processed through a multi-step engineering pipeline to ensure data integrity and analytical readiness:

* **Text Formatting & Cleaning:** Standardized mixed-case patient names into proper Title Case and corrected negative/erroneous billing amounts.
* **Feature Engineering:** Derived operational fields such as `Stay_Duration_Days` and `Daily_Billing_Amount`.
* **Deduplication:** Applied `ROW_NUMBER()` window functions partitioned by patient identifiers to retain only unique, latest admission records.
* **Relational Schema Building:** Assigned synthetic `Visit_ID` (Primary Key) and `Patient_ID` (Surrogate Key) for proper data modeling.

> 📄 **Complete SQL Cleaning Script:** For the full, step-by-step SQL data cleaning script, please refer to the primary technical documentation file:  
> **`Healthcare Data Analysis - SQL & Python Documentation.pdf`**

---

## 📊 Sample Analytical Queries & Business Insights

Here is a snippet of key business analytical queries designed to answer critical operational questions:

```sql
-- 1. Normal Result Rate per Condition & Medication
SELECT 
    `Medical Condition`,
    `Medication`,
    COUNT(*) AS Total_Patients,
    ROUND(AVG(CASE WHEN `Test Results` = 'Normal' THEN 100.0 ELSE 0.0 END), 1) AS Normal_Result_Rate_Pct
FROM hospital_data
GROUP BY `Medical Condition`, `Medication`
ORDER BY Normal_Result_Rate_Pct DESC
LIMIT 5;

-- 2. Emergency Blood Type Demand Allocation
SELECT 
    `Blood Type`,
    COUNT(*) AS Emergency_Patients_Count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS Share_Pct
FROM hospital_data
WHERE `Admission Type` = 'Emergency'
GROUP BY `Blood Type`
ORDER BY Emergency_Patients_Count DESC;
```

> 💡 **More Insights Available:** The project contains **7 comprehensive business queries** covering Emergency Abnormal Risks, Extended Stay Inpatient Profiling (>20 days), Revenue Efficiency per Admission Type, Senior vs. Non-Senior Risk Segments, and Gender Distribution Analysis. All 7 queries are fully documented in **`Healthcare Data Analysis - SQL & Python Documentation.pdf`**.

---

## 🐍 Python Data Profiling (EDA) Summary

* **Correlation Identification:** Evaluated relationships between patient age, stay duration, and billing costs.
* **Outlier Detection:** Segmented high-cost admissions using 90th percentile thresholding (`quantile(0.90)`).
* **Admission Trends:** Analyzed patient admission volumes and seasonal variations across months and years.

```python
import pandas as pd
import numpy as np

# Load Cleaned Dataset & Parse Dates
df = pd.read_csv("healthcare_cleaned.csv", sep="\t")

# High-Cost Outlier Segmentation (90th Percentile)
high_cost_threshold = df["Billing Amount"].quantile(0.90)
df["High Cost Status"] = np.where(df["Billing Amount"] >= high_cost_threshold, "High Cost", "Regular")
```

---

## 📂 Key Project Files

* 📄 **`Healthcare Data Analysis - SQL & Python Documentation.pdf`**: Complete technical documentation containing all SQL cleaning scripts, Python EDA charts, and the full suite of 7 business queries.
* 📂 **`Healthcare_Analytics_Dashboard.pbix`**: Interactive Power BI dashboard file with full data models and DAX measures.
* 🖼️ **`dashboard-overview.png`**: High-resolution preview screenshot of the main operational dashboard view.

---

## 👤 Author & Contact

**Omar Mohamed Zainhom Farghali**  
*Data Analysis Specialist | Computer Science Student at Cairo University*

* **GitHub:** [@Omar-Zainhom](https://github.com/Omar-Zainhom)
* **Project Repository:** [Healthcare-Operational-Analytics](https://github.com/Omar-Zainhom/Healthcare-Operational-Analytics)
