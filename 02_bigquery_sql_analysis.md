# HealthCare Project – BigQuery & SQL Documentation

## 1. Objective

The objective of this exercise is to demonstrate the end-to-end use of SQL and BigQuery to work with a large public health dataset:

- Split a multi-year dataset into two time periods (2013–2017 and 2018–2021).
- Clean and standardize the raw data into a typed, analysis-ready table.
- Join the two time-windowed datasets to compare utilization across periods.
- Filter and aggregate results for specific counties (Los Angeles and San Diego).
- Calculate total Annual Dental Visits (ADV) for selected age groups.
- Visualize the results in a data visualization tool to compare patterns between the two counties.

---

## 2. Hypothetical Business Purpose

You are working as a data analyst for a state health agency (or a consulting team supporting them) that oversees Medi-Cal dental utilization. Leadership wants to understand how dental access and utilization have changed over time for key counties and age bands, to inform funding decisions and outreach programs.

Specifically, the business asks:

### Time segmentation

Compare utilization between two policy periods:

- **Period 1:** 2013–2017 (baseline years before certain policy and outreach changes).
- **Period 2:** 2018–2021 (post-policy/intervention period).

### Geographic focus

Focus on **Los Angeles County** and **San Diego County**, two large counties with significant Medi-Cal enrollment and potentially different access challenges.

### Population segments

Analyze **Annual Dental Visits** for age groups where preventive and early treatment are especially critical:

- Age 10–14  
- Age 15–18  
- Age 19–20  
- Age 21–34  
- Age 35–44  

### Key questions

- Did total Annual Dental Visit utilization increase or decrease for these age groups between the two periods?
- How do trends differ between Los Angeles and San Diego?
- Which age groups show relatively higher or lower utilization, and did those patterns change over time?

To answer these questions, you ingest the DHCS *“Medi-Cal Dental Utilization and Sealants by County”* dataset into BigQuery, transform and aggregate it with SQL, and then build bar chart visualizations comparing **LA vs SD by age group and time period**. This workflow is representative of the kind of ad-hoc, data-driven analysis a systems/data analyst might perform for leadership and program stakeholders.

---

## 3. Technical Implementation (SQL Queries)

### 3.1 Clean and Prepare the Dataset

**Goal:** Convert the raw CSV (all STRING fields) into a typed, analysis-ready table with a numeric `year`, numeric `users`, and a normalized utilization rate.

```sql
CREATE OR REPLACE TABLE
  `wide-hexagon-480521-b5.healthcare_project.mdsd_clean` AS
SELECT
  TRIM(County) AS county,
  Calendar_Year AS calendar_year_raw,
  CAST(REGEXP_EXTRACT(Calendar_Year, r'([0-9]{4})') AS INT64) AS year,
  TRIM(Measure) AS measure,
  TRIM(Age_Filter) AS age_filter,
  SAFE_CAST(REPLACE(Users, ',', '') AS INT64) AS users,
  SAFE_CAST(
    REPLACE(Denominator_3_Months_Continuous_Eligibility, ',', '')
    AS INT64
  ) AS denominator_3mo,
  SAFE_CAST(
    REPLACE(REPLACE(Utilization_Pct, '%', ''), ',', '') AS FLOAT64
  ) / 100.0 AS utilization_rate,
  Users_Annotation_code,
  Users_Annotation_Description,
  Denominator_Annotation_code,
  Denominator_Annotation_Description,
  Utilization_Annotation_code,
  Utilization_Annotation_Description
FROM
  `wide-hexagon-480521-b5.healthcare_project.mdsd_raw`;
```
### 3.2 Split the Dataset into Two Time Periods

**Goal**: Create two separate tables for the two business-defined periods:

Dataset 1: 2013–2017

Dataset 2: 2018–2021

```sql
-- 2013–2017 subset
CREATE OR REPLACE TABLE
  `wide-hexagon-480521-b5.healthcare_project.dental_2013_2017` AS
SELECT
  *
FROM
  `wide-hexagon-480521-b5.healthcare_project.mdsd_clean`
WHERE
  year BETWEEN 2013 AND 2017;

-- 2018–2021 subset
CREATE OR REPLACE TABLE
  `wide-hexagon-480521-b5.healthcare_project.dental_2018_2021` AS
SELECT
  *
FROM
  `wide-hexagon-480521-b5.healthcare_project.mdsd_clean`
WHERE
  year BETWEEN 2018 AND 2021;
```
### 3.3 Join the Split Datasets (Compare Periods)

***Goal***: Aggregate by county and age_filter for each period, then join them to compare 2013–2017 vs 2018–2021 side by side for Annual Dental Visit measures.

```sql
CREATE OR REPLACE TABLE
  `wide-hexagon-480521-b5.healthcare_project.dental_joined_periods` AS
WITH p1 AS (
  -- Period 1: 2013–2017
  SELECT
    county,
    age_filter,
    SUM(users) AS total_users_2013_2017
  FROM
    `wide-hexagon-480521-b5.healthcare_project.dental_2013_2017`
  WHERE
    measure LIKE 'Annual Dental Visit%'
  GROUP BY
    county,
    age_filter
),
p2 AS (
  -- Period 2: 2018–2021
  SELECT
    county,
    age_filter,
    SUM(users) AS total_users_2018_2021
  FROM
    `wide-hexagon-480521-b5.healthcare_project.dental_2018_2021`
  WHERE
    measure LIKE 'Annual Dental Visit%'
  GROUP BY
    county,
    age_filter
)
SELECT
  p1.county,
  p1.age_filter,
  p1.total_users_2013_2017,
  p2.total_users_2018_2021
FROM
  p1
LEFT JOIN
  p2
ON
  p1.county = p2.county
  AND p1.age_filter = p2.age_filter;
```

### 3.4 Filter the Joined Dataset by County (LA & SD)

***Goal***: Restrict the joined dataset to Los Angeles and San Diego counties.

```sql
SELECT
  county,
  age_filter,
  total_users_2013_2017,
  total_users_2018_2021
FROM
  `wide-hexagon-480521-b5.healthcare_project.dental_joined_periods`
WHERE
  county IN ('Los Angeles', 'San Diego')
ORDER BY
  county,
  age_filter;
```

### 3.5 Calculate Total Annual Dental Visits (Example: Los Angeles County)

***Goal***: Calculate total Annual Dental Visits in Los Angeles County for the specified age groups across both periods.

```sql
SELECT
  SUM(total_users_2013_2017) AS total_adv_2013_2017,
  SUM(total_users_2018_2021) AS total_adv_2018_2021,
  SUM(total_users_2013_2017 + total_users_2018_2021) AS total_adv_2013_2021
FROM
  `wide-hexagon-480521-b5.healthcare_project.dental_joined_periods`
WHERE
  county = 'Los Angeles'
  AND age_filter IN (
    'Age 10-14',
    'Age 15-18',
    'Age 19-20',
    'Age 21-34',
    'Age 35-44'
  );
```

### 3.6 Create a View for Data Visualization (LA vs SD by Age Group)

****Goal***: Prepare a clean, aggregated view for downstream visualization (e.g., bar charts comparing LA vs SD by age group and period).

```sql
CREATE OR REPLACE VIEW
  `wide-hexagon-480521-b5.healthcare_project.adv_la_sd_by_age` AS
SELECT
  county,
  age_filter,
  total_users_2013_2017,
  total_users_2018_2021
FROM
  `wide-hexagon-480521-b5.healthcare_project.dental_joined_periods`
WHERE
  county IN ('Los Angeles', 'San Diego')
  AND age_filter IN (
    'Age 10-14',
    'Age 15-18',
    'Age 19-20',
    'Age 21-34',
    'Age 35-44'
  );

-- Sanity check / export for visualization
SELECT *
FROM `wide-hexagon-480521-b5.healthcare_project.adv_la_sd_by_age`
ORDER BY county, age_filter;
```
![LA vs SD Annual Dental Visits by Age Group and Period](images/detal_datav.png)
