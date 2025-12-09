# healthcare-dental-analytics
“End-to-end project using Selenium, BigQuery, and Python testing framework with DHCS dental utilization data.”
# HealthCare Dental Utilization – Automation, SQL, and Testing

This project simulates an end-to-end workflow for a junior systems/data analyst working with
California DHCS Medi-Cal dental utilization data.

It is split into three parts:

1. **Automation (Python + Selenium)**  
   - Automates navigation to the DHCS Dental Reports page  
   - Locates the latest **Dental FFS and DMC Performance Fact Sheet**  
   - Downloads the latest PDF to a local folder  
   - Includes documentation and screenshots

2. **SQL & BigQuery Analysis**  
   - Loads the DHCS *Medi-Cal Dental Utilization and Sealants by County* dataset into BigQuery  
   - Cleans and types the raw CSV into an analysis-ready table (`mdsd_clean`)  
   - Splits data into two time windows (2013–2017, 2018–2021)  
   - Joins and aggregates Annual Dental Visits (ADV) for **Los Angeles** and **San Diego**  
   - Creates a view for downstream visualizations  
   - Includes documentation and screenshots of the final charts

3. **Testing Framework (Python + BigQuery)**  
   - Python script (`healthcare_adv_vis.py`) that:
     - Accepts a county name as input
     - Validates it against California counties in `mdsd_clean`
     - Queries `dental_joined_periods` for Ages 10–44
     - Visualizes ADV totals by age group using matplotlib
     - Handles invalid inputs with clear error messages
   - Manual test cases covering:
     - Los Angeles (happy path)
     - San Diego (happy path)
     - Misspelled / non-existent county
     - County outside California
   - Includes test documentation and screenshots

## Repository Structure

```text
automation/
  automation_download_dhcs.py
  Automation_Project_Python_Selenium_Documentation.docx
  screenshots/

sql/
  bigquery_healthcare.sql
  BigQuery_Medical_Data_Documentation.docx
  screenshots/

testing/
  healthcare_adv_vis.py
  HealthCare_Project_Testing_Framework_Documentation.docx
  screenshots/
