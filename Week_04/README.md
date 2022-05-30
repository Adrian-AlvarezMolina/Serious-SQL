  -- COUNT (*) / (SUM(COUNT(*)))
FROM CTE
GROUP BY types_of_measures;
```
### 9. For users that have blood pressure measurements, what is the median systolic/diastolic blood pressure values?

This parameter could be important. However, as the data still contains some errors which can drastically affect the mean is safer to check the median (less impacted by outliers).

```SQL
SELECT
  PERCENTILE_CONT(0.5) WITHIN GROUP(
    ORDER BY systolic) AS systolic_median,
  PERCENTILE_CONT(0.5) WITHIN GROUP(
    ORDER BY diastolic) AS diastolic_median
FROM health.user_logs 
WHERE measure = 'blood_pressure';
```
**Median systolic pressure is 126**
**Median diastolic pressure is 79**  

---
### 10. Debbuging the code given by the team

Some code was proposed to solve the previous questions, but it contain some minor errors which can be solved.
```SQL
-- 1. How many unique users exist in the logs dataset?
/*SELECT
  COUNT DISTINCT user_id
FROM health.user_logs;*/
  
  -- Corrected
SELECT
  COUNT (DISTINCT id)
FROM health.user_logs;

-- for questions 2-8 we created a temporary table
/* DROP TABLE IF EXISTS user_measure_count;
CREATE TEMP TABLE user_measure_cout
SELECT
    id,
    COUNT(*) AS measure_count,
    COUNT(DISTINCT measure) as unique_measures
  FROM health.user_logs
  GROUP BY 1; */

  -- Create the temp table with "AS ()", 
  -- Correct a spelling error un the name of the table
DROP TABLE IF EXISTS user_measure_count;
CREATE TEMP TABLE user_measure_count AS (
SELECT
    id,
    COUNT(*) AS measure_count,
    COUNT(DISTINCT measure) as unique_measures
  FROM health.user_logs
  GROUP BY 1); 

-- 2. How many total measurements do we have per user on average?
/*SELECT
  ROUND(MEAN(measure_count))
FROM user_measure_count;*/
-- Change MEAN per "AVG", incorporate the digits to round
SELECT
  ROUND(AVG (measure_count),
        2)
FROM user_measure_count;

-- 3. What about the median number of measurements per user?
/*SELECT
