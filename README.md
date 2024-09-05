# Tetanus Vaccine Dashboard 

## Table of Contents
1. [Introduction](#introduction)
2. [Tools Used and Dashboard Preview](#tools-used-and-dashboard-preview)
3. [Data Cleaning and Preparation](#data-cleaning-and-preparation)
4. [Results](#results)

## Introduction
The Tetanus Vaccination Dashboard for the hospital provides a comprehensive overview of patient vaccination status, helping healthcare providers efficiently manage and track tetanus immunization efforts. This dashboard is designed to identify patients who have received specific tetanus vaccine codes, monitor the recency of their vaccinations, and highlight those due for a booster shot. Additionally, it enables the hospital staff to focus on active patients who are currently engaged with the hospital's services, ensuring timely and effective vaccine administration. With this tool, the hospital can enhance patient safety and promote proactive health management.

The main goal of this project is to develop a tetanus vaccination dashboard by integrating data from multiple tables (encounters, immunizations, and patients), each capturing different facets of patient interactions with the hospital. The dataset employed in this project is synthetically generated, ensuring that no real patient data is involved.

Dashboard Requirements:
- List of patients who have received codes 5303 or 5324.
- Most recent date of the shot, or null if there is no history.
- A calculated field indicating the number of years since the last tetanus shot.
- Only include patients who are still active with the hospital, defined as having had an encounter within the last two years.
    
**Link to dashboard**: https://public.tableau.com/app/profile/chung.hsi.shan/viz/TetanusVaccineDashboardforHospital/Dashboard1

**Link to dataset**:
https://datawizardry.academy/


## Tools Used and Dashboard Preview
- **PostgreSQL**: Used for all data cleaning, transformation and analysis.
- **Tableau**: Dashboard creation and data visualization.
- Below are screenshots taken from the Tetanus Vaccine Dashboard, which is created using Tableau:
![Dashboard 1 (1)](https://github.com/user-attachments/assets/27ef9c71-fd88-43dc-a8ba-6f3cd9069ca1)

## Data Cleaning and Preparation
1. **Remove Duplicates**: Identifies and removes duplicate records based on a unique combination of columns.
2. **Handle Missing Values**: Detects missing values in crucial fields and considers possible actions.
3. **Validate Data**: Checks for data consistency and correctness (e.g., negative values in numeric fields and ensures the right ID length).
4. **Standardizing Data Formats**: Check for data consistency and apply the right format to the different fields.

### Code Snippets
```sql
/* For simplicity, only the code that is used to clean the encounters table is shown,
the same methodology is applied to the immunizations and patients table. */

-- 1. Remove duplicates based on the defined unique fields

WITH ranked_encounters AS (
    SELECT 
        id,
        ROW_NUMBER() OVER (
            PARTITION BY start, stop, patient, organization, payer 
            ORDER BY id
        ) AS row_num
    FROM public.encounters
)
DELETE FROM public.encounters
WHERE id IN (
    SELECT id
    FROM ranked_encounters
    WHERE row_num > 1
);

-- 2. Check for missing values

SELECT *
FROM public.encounters
WHERE id IS NULL 
    OR start IS NULL
    OR stop IS NULL
    OR patient IS NULL
    OR organization IS NULL
    OR provider IS NULL
    OR payer IS NULL
    OR encounterclass IS NULL
    OR base_encounter_cost IS NULL
    OR total_claim_cost IS NULL
    OR payer_coverage IS NULL
    OR reasoncode IS NULL;

-- 3. Validate data to ensure data fits within constraints

SELECT *
FROM public.encounters
WHERE total_claim_cost < 0 
    OR base_encounter_cost < 0
    OR payer_coverage < 0;

-- 4. Standardizing data formats

UPDATE public.encounters
SET start = start::timestamp,
    stop = stop::timestamp;

/* Finally, this is the final query that filters based on the dashboard requirements
- List of patients who have received codes 5303 or 5324.
- Most recent date of the shot, or null if there is no history.
- A calculated field indicating the number of years since the last tetanus shot.
- Only include patients who are still active with the hospital, defined as having had an encounter within the last two years.*/

WITH patient_history AS (
    SELECT patient, MAX(date) AS latest_tetanus_date
    FROM public.immunizations
    WHERE code = '5303' OR code = '5324'
    GROUP BY patient
),
active_patients AS (
    SELECT DISTINCT patient
    FROM public.encounters AS e
    JOIN public.patients AS pat 
		ON e.patient = pat.id
    WHERE start BETWEEN '2019-01-01 00:00' AND '2022-12-31 23:59'
      AND pat.deathdate IS NULL
)

SELECT pat.id, 
	pat.first ||' '|| pat.last as pat_name,
	pat.birthdate, 
	EXTRACT (YEAR FROM age(current_date, pat.birthdate)) +(EXTRACT (MONTH FROM age(current_date, pat.birthdate))/12) +(EXTRACT (DAY FROM age(current_date, pat.birthdate))/365.25) AS age_in_years,
	pat.race, 
	pat.ethnicity, 
	pat.gender, 
	pat.city, 
	pat.county,
	pat.state,
	ph.latest_tetanus_date,
    CASE 
        WHEN ph.latest_tetanus_date IS NOT NULL THEN EXTRACT(YEAR FROM age('2022-12-31', ph.latest_tetanus_date))
        ELSE NULL
    END AS years_since_last_shot
FROM public.patients AS pat
LEFT JOIN patient_history AS ph ON pat.id = ph.patient
WHERE pat.id IN (SELECT patient FROM active_patients) 
	AND ((ph.latest_tetanus_date IS NULL) OR (EXTRACT(YEAR FROM age('2022-12-31', ph.latest_tetanus_date)) >=0 ));

```


## Results
The Massachusetts Hospital Tetanus Shot Dashboard allows users to visualize key trends in patient eligibility for tetanus shots across various regions, age groups, and races. It enables users to filter the data by county, age, years since the last shot, and race to focus on specific populations. The map visualization provides insights into eligible patient distributions by county and city, while the profile table gives detailed patient information such as name, age, and years since the last shot. Additionally, the bar chart highlights how eligibility varies by age group and race, offering a comprehensive overview that helps healthcare providers prioritize vaccination efforts based on demographic and geographic insights.
