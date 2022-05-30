# Health measures case study
This is a final recap about SQL learnings of the first 4 weeks of the course [Serious SQL](https://www.datawithdanny.com/courses/serious-sql).

+ Basic sintax
`SELECT, GROUP BY, CASE WHEN, WHERE, DISTINCT`
+ Creation of new tables
`COMMON TABLE EXPRESSION (CTEs) and TEMP TABLES`
+ Numeric functions
`COUNT, AVG, MODE, STDDEV, NTILE, WIDTH_BUCKET, ...`

`health.user_logs` is a database which measure blood_glucose blood_pressure and weight from a set of users. Composed by 43891 rows of data. 

Here you can see a brief view of the records

| id                                       | log_date                 | measure        | measure_value | systolic | diastolic |
|------------------------------------------|--------------------------|----------------|---------------|----------|-----------|
| d14df0c8c1a5f172476b2a1b1f53cf23c6992027 | 2020-10-15T00:00:00.000Z | blood_pressure | 140           | 140      | 113       |
| 0f7b13f3f0512e6546b8d2c0d56e564a2408536a | 2020-10-21T00:00:00.000Z | blood_glucose  | 166           | 0        | 0         |
| 0f7b13f3f0512e6546b8d2c0d56e564a2408536a | 2020-10-22T00:00:00.000Z | blood_glucose  | 142           | 0        | 0         |
| 87be2f14a5550389cb2cba03b3329c54c993f7d2 | 2020-10-12T00:00:00.000Z | weight         | 129.060012817 | 0        | 0         |
| 0efe1f378aec122877e5f24f204ea70709b1f5f8 | 2020-10-07T00:00:00.000Z | blood_glucose  | 138           | 0        | 0         |

## First questions about health.user_logs

### 1.  How many unique users exist in the logs dataset?

A simple question, but nor for it less important.
>**There are 554 different id**

```SQL
SELECT
  COUNT(DISTINCT(id))
FROM health.user_logs
```

### 2.  How many total measurements do we have per user on average?
It's important to track if the users are recording frequently their health values.

>On average, every user submitted **79 measures**

```SQL
WITH measures_per_user AS (
  SELECT 
    id,
    COUNT(*) AS measures_per_id
  FROM health.user_logs
  GROUP BY id)
SELECT
  ROUND(AVG(measures_per_id),
            0) AS avg_measures_per_user
FROM measures_per_user;
```

### 3.  What is the median number of measurements per user?
Check the median number of measures per user instead of the mean will help to detect if there problems with the data, as median is a metric much more robust against outliers than the mean.

>The median number of measures per user is 2, **The difference in the average number of measures (79) indicates the presence of some id which could had an enormous number of records**

```SQL
WITH measures_per_user AS (
SELECT
  id,
  COUNT(*) AS measurements
FROM health.user_logs
GROUP BY
  id)
SELECT
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY measurements) AS median_number_of_measures_per_user
FROM measures_per_user;
```
### 4.  How many users have 3 or more measurements?

It's possible to calculate it directly from the original table, or extract it from a CTE where every ID has the number of different records .

>There are **209 id with 3 or more measures**

```SQL
SELECT
  id,
  COUNT(*) AS num_of_measures
FROM health.user_logs
GROUP BY id
HAVING COUNT(*) >= 3;
```

### 5.  How many users have 1,000 or more measurements?

The same query used above can solve this question changing the number in the last row.

>**5 ids has more than 1000 measures**,  this high number of records can indicated very active users, or some testing ids used by designers to check the app works properly

```SQL
SELECT
  id,
  COUNT(*) AS num_of_measures
FROM health.user_logs
GROUP BY id
HAVING COUNT(*) >= 1000;
```
### 6. How many users have logged blood glucose measurements?
Can be important to determine how many users are logging their blood glucose if this database was developed for diabetic people.

In order to know how many users incorporated their glucose values we can know grouping by `id` and filtering by "measure = `blood_glucose` "

```SQL
-- Inspect the id with blood_glucose measures and
-- how many measures they have
SELECT
  COUNT (*) AS blood_glucose_measures,
  id
FROM health.user_logs
WHERE measure = 'blood_glucose'
GROUP BY id;

-- To directly know how many users introduce glucose 
-- values
SELECT
  COUNT (*) AS ids_with_blood_glucose
FROM (
  SELECT 
    DISTINCT id
  FROM health.user_logs
  WHERE measure = 'blood_glucose') AS subquery
```
>**325 users incorporate blood glucose values to the database**

### 7. How many users have at least 2 types of measurements?

Apart of the number of user who introduce their blood glucose values, how many users incorporate more than 1 measure to the database ?

```SQL
-- To inspect the records
SELECT
  id,
  COUNT(DISTINCT measure) AS type_of_measure
FROM health.user_logs
GROUP BY id
HAVING COUNT(DISTINCT measure) >= 2;

-- Just to have the number of different ID with two or
-- more different measures types.
SELECT
  COUNT(*)
FROM
  (SELECT 
    id,
    COUNT(DISTINCT measure) AS number_of_measures
   FROM health.user_logs
   GROUP BY id
   HAVING COUNT(DISTINCT measure) >= 2) AS subquery;
```
>**204 different users** introduce 2 or more types of measures

### 8. How many users have all 3 measures - blood glucose, weight and blood pressure?

Who and how many users incorporate the three different parameters asked (weigth, blood_glucose and blood_pressure) ?

Those users help to better understand the relation of the three variables.

>**50 users** incorporate to the database the three different measures asked. **Just 9% of the users**

```SQL
WITH CTE AS (
SELECT
  id,
  COUNT (DISTINCT measure) AS types_of_measures
FROM health.user_logs
GROUP BY 
  id)

SELECT  
  types_of_measures,
  COUNT (*) AS users,
  ROUND (COUNT (*) * 100.00 / (SELECT COUNT(*) FROM CTE),
         2) AS percentage
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
>**Median systolic pressure is 126**
>**Median diastolic pressure is 79**  

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
  PERCENTILE_CONTINUOUS(0.5) WITHIN GROUP (ORDER BY id) AS median_value
FROM user_measure_count;*/
-- The way to extract the median is "PERCENTILE_CONT"
-- values should be ordered by measure_count instead of by id
SELECT
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY measure_count) AS median_value
FROM user_measure_count;

-- 4. How many users have 3 or more measurements?
/*SELECT
  COUNT(*)
FROM user_measure_count
HAVING measure >= 3;*/
-- Change "HAVING" per "WHERE", HAVING is just for grouped variables
-- Correct variable name, measure_count instead of measure
SELECT
  COUNT(*)
FROM user_measure_count
WHERE measure_count >= 3;


-- 5. How many users have 1,000 or more measurements?
/*SELECT
  SUM(id)
FROM user_measure_count
WHERE measure_count >= 1000;*/

-- Change SUM per COUNT, as we want to know the number of rows that meet the condition, not to sum character values of IDs.
SELECT
  COUNT(id)
FROM user_measure_count
WHERE measure_count >= 1000;

-- 6. Have logged blood glucose measurements?
/* SELECT
  COUNT DISTINCT id
FROM health.user_logs
WHERE measure is 'blood_sugar'; */

-- Change "measure is" per "measure = "
-- the measure asked is 'blood_glucose'
SELECT
  COUNT DISTINCT id
FROM health.user_logs
WHERE measure = 'blood_sugar';

-- 7. Have at least 2 types of measurements?
/*SELECT
  COUNT(*)
FROM user_measure_count
WHERE COUNT(DISTINCT measures) >= 2;*/

-- There is no "measures" column in the CTE, unique_measures has the information required and should be equal or greater than 2 
SELECT
  COUNT(*)
FROM user_measure_count
WHERE unique_measures >= 2;

-- 8. Have all 3 measures - blood glucose, weight and blood pressure?
/*SELECT
  COUNT(*)
FROM usr_measure_count
WHERE unique_measures = 3;*/

-- Check the name of the temporary table
SELECT
  COUNT(*)
FROM user_measure_count
WHERE unique_measures = 3;

-- 9.  What is the median systolic/diastolic blood pressure values?
/*SELECT
  PERCENTILE_CONT(0.5) WITHIN (ORDER BY systolic) AS median_systolic
  PERCENTILE_CONT(0.5) WITHIN (ORDER BY diastolic) AS median_diastolic
FROM health.user_logs
WHERE measure is blood_pressure;*/

-- Add "GROUP" after WITHIN when PERCENTILE_CONT is called.
-- In the WHERE filter substitute "is blood pressure" by " = 'blood_pressure' "

SELECT
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY systolic) AS median_systolic
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY diastolic) AS median_diastolic
FROM health.user_logs
WHERE measure = 'blood_pressure';
```
