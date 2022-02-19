## Identifying Duplicate Records
### Exercises
#### 1. Highest Duplicate
> 1. Which `id` value has the most number of duplicate records in the health.user_logs table?
```sql
WITH groupby_counts AS (
  SELECT
    id,
    log_date,
    measure,
    measure_value,
    systolic,
    diastolic,
    COUNT(*) AS frequency
  FROM health.user_logs
  GROUP BY
    id,
    log_date,
    measure,
    measure_value,
    systolic,
    diastolic
)
SELECT
  id,
  SUM(frequency) AS total_duplicate_rows
FROM groupby_counts
WHERE frequency > 1
GROUP BY id
ORDER BY total_duplicate_rows DESC
LIMIT 10;
```
#### 2. Second Highest Duplicate
>  2. Which `log_date` value had the most duplicated records after removing the maxx duplicate id value from question 1?
```sql
WITH groupby_counts AS (
  SELECT
    id,
    log_date,
    measure,
    measure_value,
    systolic,
    diastolic,
    COUNT(*) AS frequency
  FROM health.user_logs
  WHERE id != '054250c692e07a9fa9e62e345231df4b54ff435d'
  GROUP BY
    id,
    log_date,
    measure,
    measure_value,
    systolic,
    diastolic
)
SELECT
  log_date,
  SUM(frequency) AS total_duplicate_rows
FROM groupby_counts
WHERE frequency > 1
GROUP BY log_date
ORDER BY total_duplicate_rows DESC
LIMIT 10;
```
#### 3. Highest Occuring Value
> 3. Which `measure_value` had  the most occurences in the `health.user_logs` value when `meausre = 'weight'`?

```sql
SELECT 
  measure_value,
  COUNT(*) frequency
FROM health.user_logs
WHERE measure = 'weight'
GROUP BY measure_value
ORDER BY frequency DESC
LIMIT 10;
```
#### 4. Single & Total Duplicated Rows
> 4. How many single duplicated rows exist wen `measure = 'blood_pressure'` in the `health.user_logs`? How about the total number of duplicate records in the same table?

```sql
WITH groupby_counts AS (
   SELECT
      id,
      log_date,
      measure,
      measure,
      measure_value,
      systolic,
      diastolic,
      COUNT(*) AS frequency
    FROM health.user_logs
    WHERE measure = 'blood_pressure'
    GROUP BY
      id,
      log_date,
      measure,
      measure,
      measure_value,
      systolic,
      diastolic
)
SELECT 
  COUNT(*) as single_duplicate_rows,
  SUM(frequency) AS total_duplicate_rows
FROM groupby_counts
WHERE frequency > 1;
```
#### 5. Percentage of Table Records
> 5. What percentage of records `measure_value = 0` when `measure = 'blood_pressure'` in the `health.user_logs` table? How many records are there also for this same conditions?

```sql
WITH all_measure_values AS (
   SELECT
      measure_value,
      COUNT(*) AS total_records,
      SUM(COUNT(*)) OVER () AS overall_total
    FROM health.user_logs
    WHERE measure = 'blood_pressure'
    GROUP BY 1
)
SELECT 
  measure_value,
  total_records,
  overall_total,
  ROUND(
    100 * total_records::NUMERIC / overall_total, 2
  ) AS percentage
FROM all_measure_values
WHERE measure_value = 0;
```
ALTERNATIVELY?
```sql
WITH measure_value_0 AS (
  SELECT
    measure_value,
    COUNT(*) AS total_records_measure_0
  FROM health.user_logs
  WHERE measure_value = 0
  GROUP BY 1
)

WITH all_measure_values AS (
  SELECT
    amv.measure_value,
    COUNT(*) AS total_records_all_measures
  FROM health.user_logs
  WHERE measure_value != 0
  GROUP BY 1
)
SELECT 
  measure_value,
  total_records_all_measures
  total_records_measure_0 AS total_records,
  ROUND(
    100 * total_records_all_measure::NUMERIC / total_recors_measure_0), 2
  ) AS percentage
FROM measure_value_0 mvz
JOIN all_measure_values amv ON measure_value.mvz = measure_value.amv
WHERE measure = 'blood_pressure';
```
#### 6. Percentage of Duplicates
> 6. What percentage of records are duplicates in the `health.user_logs` table?

```sql
WITH groupby_counts AS (
  SELECT
    id,
    log_date,
    measure,
    measure_value,
    systolic,
    diastolic,
    COUNT(*) AS frequency
  FROM health.user_logs
  GROUP BY
    id,
    log_date,
    measure,
    measure_value,
    systolic,
    diastolic
)
SELECT
  -- Need to subtract 1 from the frequency to count actual duplicates!
  -- Also don't forget about the integer floor division!
  ROUND(
    100 * SUM(CASE
        WHEN frequency > 1 THEN frequency - 1
        ELSE 0 END
    )::NUMERIC / SUM(frequency),
    2
  ) AS duplicate_percentage
FROM groupby_counts;
```
#### 7. Calculating All the Summary Statistics
##### 7.1. Average, Median & Mode
> 1. What is the average, median and mode values of blood glucose values to 2 decimal places?

```sql
SELECT
  ROUND(AVG(measure_value), 2) AS average_value,
  ROUND(
    CAST(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY measure_value) AS NUMERIC),
    2
  ) AS median_value,
  ROUND(
    MODE() WITHIN GROUP (ORDER BY measure_value),
    2
  ) AS mode_value
FROM health.user_logs
WHERE measure = 'blood_glucose';
```

##### 7.2. Most Frequent Values
> 2. What is the most frequently occuring `measure_value` value for all blood glucose measurements?

```sql
SELECT
    measure_value,
    COUNT(*) AS frequency
FROM health.user_logs
WHERE measure = 'blood_glucose'
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```

##### 7.3. Pearson Coefficient of Skewness
> 3. Calculate the 2 Pearson Coefficient of Skewness for blood glucose measures given the following formulas:

            ***Coefficient1: ModeSkewness = _***Mean - Mode***_
                                                ***StandardDeviation***
            ***Coefficient2: MedianSkewness = 3 * _***Mean - Median***_
                                                  ***StandardDeviation***
```sql
WITH cte_blood_glucose_stats AS (
  SELECT 
    AVG(measure_value) AS mean_value,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY measure_value) AS median_value,
    MODE() WITHIN GROUP (ORDER BY measure_value) AS mode_value,
    STDDEV(measure_value) AS stddev_value
  FROM health.user_logs
  WHERE measure = 'blood_glucose'
)
SELECT
  ( mean_value - mode_value ) / stddev_value AS pearson_corr_1,
  3 * ( mean_value - median_value) / stddev_value AS pearson_corr_2
FROM cte_blood_glucose_stats;
```

