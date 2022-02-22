## Identifying Duplicate Records

### Exercises

#### 1. Highest Duplicate
>
> 1. Which `id` value has the most number of duplicate records in the health.user_logs table?
>
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
>
>  2. Which `log_date` value had the most duplicated records after removing the maxx duplicate id value from question 1?
>
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
>
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
>
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
>
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
>
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
>
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
>
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
>
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

### Beyong Summary Statistics

#### Algorithmic Thinking

After performing basic summary statistics on our data set, we want extend by analysing and validating the results we see to be true.

1. Order all of the values from smallest to largest

We visualize the `weight` data point and line them up from smallest to largest.

```sql
SELECT
  measure_value
FROM health.user_logs
WHERE measure = 'weight'
```

2. Split them into 100 equal buckets. Assign a number from 1 through to 100 from each bucket

By spliting our sorted dataset into 100 buckets, we are effectively generating our new "groups" or buckets of data to continue with our algorithm.

Each bucket will have 1% of te total number of records from the entire dataset.

To get a better insight into this, we need to first find out how many records did we have where `measure = 'weight'`.

```sql
SELECT
  COUNT(*)
FROM health.user_logs
WHERE measure = `weight`;
```

The total number records we get is `2782`.

3. For each bucket we need to:

* Calculate the minimum value and the maximum value for the ceiling and floor values
* Count how many records there are

4. Combine all aggregrated bucket results into a final summary table

#### SQL Implementation

1. Order and Assign
  To implement the algorithm, we start by
    i. Ordering all of the weight measurement values from smallest to largest
    ii. Splitting them into 100 equal buckets and ssigning numbers from 1 through to 100 for each bucket.

We use the `NTILE` window function to achieve this task as below:

```sql
SELECT
  measure_value,
  NTILE(100) OVER (
    ORDER BY
      measure_value
  ) AS percentile
FROM health.user_logs
WHERE measure = 'weight'
```

2. Bucket Calculations
Since we now ave our percentile values and our dataset is split into 100 buckets, we can then do a `GROUP BY` on the `percentile` column from the previous SQL output table to calculate our `MIN` and `MAX` `measure_value` ranges for each bucket and also the `COUNT` of records for the `percentile_counts` field.

The SQL query to achieve this is an augemtation of the previous query as it is transformed into a CTE and we do a single `SELECT` statement and aggregrations for simplicity.

```sql
WITH percentile_values AS (
  SELECT
    measure_value,
    NTILE(100) OVER (
      ORDER BY
        measure_value
    ) AS percentile
  FROM health.user_logs
  WHERE measure = 'weight'
)
SELECT
  percentile,
  MIN(measure_value) AS floor_value,
  MAX(measure_value) AS ceiling_value,
  COUNT(*) AS percentile_counts
FROM percentile_values
GROUP BY percentile
ORDER BY percentile;
```

#### Critical Thinking

After implementing the algorithm to understand the data the question we ask is: then what?  What do we do with this cumulative distibutioin table?

The first place to start is to take a careful look at tthe tails of the values - namely `percentile = 1`and `percentile = 100`

The table below gives us baseline to work with:

> <<< insert table here >>>

So from first glance we can get the following insights right away:

  1. 28 values lie between 0 and ~29KG
  2. 27 values lie between 136.53KG and 39,642,120KG

None of these insights look normal. We need to probe and dick deeper to understand what it means.

We'll make some assumptions as we based on the insights we generated from our table above.

When we think of those small values in the 1st percentile under 29KG, we can assume that:

  1. There were some incorrectly measured values - leading to some 0KG weight measurements
  2. Some of the low weights under 29KG were actually valid measurements from yound children who were part of the customer base

For the 100th percentile we could consider:

  1. Does the 130KG floor value make sense?
  2. How many error values were actually recorded in the dataset?

#### Deep Dive Into 100th Percentile

We will take a further look into the 100th percentile sub-dataset from the already defined query using a window functions such as `ROW_NUMBER`, `RANK` and `DENSE_RANK` in the following query:

```sql
WITH percentile_values AS (
  SELECT
    measure_value,
    NTILE(100) OVER (
      ORDER BY
        measure_value
    ) AS percentile
  FROM health.user_logs
  WHERE measure = 'weight'
)
SELECT
  measure_value,
  -- these are examples of window functions below
  ROW_NUMBER() OVER (ORDER BY measure_value DESC) as row_number_order,
  RANK() OVER (ORDER BY measure_value DESC) as rank_order,
  DENSE_RANK() OVER (ORDER BY measure_value DESC) as dense_rank_order
FROM percentile_values
WHERE percentile = 100
ORDER BY measure_value DESC;
```

> <<<insert table>>>

The main difference between the window functions above lies in the ties in the ordering columns. With `RANK` and `DENSE_RANK`, all rows with the same value for the ordering columns will end up with an equal rank. ROW_NUMBER on the order hand will assign an incremental rank for the rows with ties. For the `RANK` ordering columns, the ranking is preserved for example [1,1,3,4] whereas `DENSE_RANK` will never give any gaps ex. [1,1,2,3].s.

#### Large Outliers

After running the previous query we could identify 3 HUGE values which are outliers in our weight values in the top percentile.

><<<insert table>>>

The last 200KG value MIGHT be a real measurement - so we would apply judgement to determine whether this is actually an outlier or not.

Now that we've identified these large outliers, what we'll do next is to remove. However before we proceed we first inspect the first percentile with the smallest values.

#### Small Outliers

We apply the ordering window functions here once more. This time around we will sort from smallest to largest so we will remove all of the `DESC` parts in our query to end up with the following:

```sql
WITH percentile_values AS (
  SELECT
    measure_value,
    NTILE(100) OVER (
      ORDER BY
        measure_value
    ) AS percentile
FROM health.user_logs
WHERE measure = 'weight'
)
SELECT
  measure_value,
  ROW_NUMBER() OVER (ORDER BY measure_value) as row_number_order,
  RANK() OVER (ORDER BY measure_value) as rank_order,
  DENSE_RANK() OVER (ORDER BY measure_value) as dense_rank_order
FROM percentile_values
WHERE percentile = 1
ORDER BY measure_value;
```

  <<< insert table >>>

To interprete these results - it seems like there are a few 0KG values as well as a few VERY small values.

Again, this will be a judgement call as to whether to or not to keep or remove some of these values.

For the simplicity of the project, we will remove just the 0KG values on the assumption that it is would have been an error.

#### Removing Outliers

Here comes the fun part - removing these outliers! To do this - we can simply use `WHERE` filter with some SQL conditions.

We create a temporary `clean_weight_logs` table with our outliers removed by applying strict inequality conditions.

We achieve this task by doing the following query:

1. Create a temporary table using a `CREATE TABLE TEMP <> AS` statement

```sql
DROP TABLE IF EXISTS clean_weight_logs;
CREATE TEMP TABLE clean_weight_logs AS (
  SELECT *
  FROM health.user_logs
  WHERE measure = 'weight'
    AND measure_value > 0
    AND measure_value < 201
);
```

The query above returns nothing because it is a temp table.

2. Calculate summary statistics on this new temp table

Now we re-use the same code we used for the original summary statistics and compare this to the original output

```sql
SELECT
  ROUND(MIN(measure_value), 2) AS minimum_value,
  ROUND(MAX(measure_value), 2) AS maximum_value,
  ROUND(AVG(measure_value), 2) AS mean_value,
  ROUND(
    -- this function actually returns a float which is incompatible with ROUND!
    -- we use this cast function to convert the output type to NUMERIC
    CAST(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY measure_value) AS NUMERIC),
    2
  ) AS median_value,
  ROUND(
    MODE() WITHIN GROUP (ORDER BY measure_value),
    2
  ) AS mode_value,
  ROUND(STDDEV(measure_value), 2) AS standard_deviation,
  ROUND(VARIANCE(measure_value), 2) AS variance_value
FROM clean_weight_logs;
```

  > <<<insert table >>>

3. Show the new cumulative distribution function with treated data

```sql
WITH percentile_values AS (
  SELECT
    measure_value,
    NTILE(100) OVER (
      ORDER BY
        measure_value
    ) AS percentile
  FROM clean_weight_logs
)
SELECT
  percentile,
  MIN(measure_value) AS floor_value,
  MAX(measure_value) AS ceiling_value,
  COUNT(*) AS percentile_counts
FROM percentile_values
GROUP BY percentile
ORDER BY percentile;
```

<<< insert table >>>

### Health Analytics Mini Case Study

We just received an urgent request from the General Manager of Analytics at Health Co requesting our assistance with their analysis of `health.user_logs` dataset

The Health Co analytics team have shared with us their SQL script - they unfortunately ran into a few bugs that they couldn’t fix!

We’ve been asked to quickly debug their SQL script and use the resulting query outputs to quickly answer a few questions that the GM has requested for a board meeting about their active users.

#### Business Questions

Here are a couple of business questions that we need to help the GM answer:

  1. How many unique users exist in the logs dataset?
  2. How many total measurements do we have per user on average?
  3. What about the median number of measurements per user?
  4. How many users have 3 or more measurements?
  5. How many users have 1,000 or more measurements?

Looking at the logs data - what is the number of the active user base who:
  6. Have logged blood glucose measurements?
  7. Have at least 2 types of measurements?
  8. Have all 3 measures - blood glucose, weight and blood pressure?

For users that have blood pressure measurements:
  9. What is the median systolic/diastolic blood pressure values?

```sql
-- 1. How many unique users exist in the logs dataset?
SELECT
  COUNT (DISTINCT id)
FROM health.user_logs;

-- for questions 2-8 we created a temporary table
CREATE TEMP TABLE user_measure_count AS (
SELECT
    id,
    COUNT(*) AS measure_count,
    COUNT(DISTINCT measure) as unique_measures
FROM health.user_logs
GROUP BY 1
)

-- 2. How many total measurements do we have per user on average?
SELECT
  ROUND(MEAN(measure_count))
FROM user_measure_count;

-- 3. What about the median number of measurements per user?
SELECT
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY measure_count) AS median_number_measurements
FROM user_measure_count;

-- 4. How many users have 3 or more measurements?
SELECT
  COUNT(id)
FROM user_measure_count
WHERE measure_count >= 3


```
