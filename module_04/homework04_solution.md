# Module 4 Homework: Analytics Engineering with dbt

In this homework, we'll use the dbt project in `04-analytics-engineering/taxi_rides_ny/` to transform NYC taxi data and answer questions by querying the models.  

Goal: Transforming the data loaded in DWH into Analytical Views developing a dbt project.  

This module focuses on yellow and green taxi data (2019-2020). While Module 3 may have included FHV data, it is not used in this dbt project.  

  

## Setup

1. Set up your dbt project following the [setup guide](../../../04-analytics-engineering/setup/)
2. Load the Green and Yellow taxi data for 2019-2020 into your warehouse
3. Run `dbt build --target prod` to create all models and run tests

> **Note:** By default, dbt uses the `dev` target. You must use `--target prod` to build the models in the production dataset, which is required for the homework queries below.

After a successful build, you should have models like `fct_trips`, `dim_zones`, and `fct_monthly_zone_revenue` in your warehouse.

---

### Question 1. dbt Lineage and Execution

Given a dbt project with the following structure:

```
models/
├── staging/
│   ├── stg_green_tripdata.sql
│   └── stg_yellow_tripdata.sql
└── intermediate/
    └── int_trips_unioned.sql (depends on stg_green_tripdata & stg_yellow_tripdata)
```

If you run `dbt run --select int_trips_unioned`, what models will be built?

- `stg_green_tripdata`, `stg_yellow_tripdata`, and `int_trips_unioned` (upstream dependencies)
- Any model with upstream and downstream dependencies to `int_trips_unioned`
- `int_trips_unioned` only
- `int_trips_unioned`, `int_trips`, and `fct_trips` (downstream dependencies)

---

#### **Answer of Question 1**

The answer is **`int_trips_unioned` only**

**Explanation**

When you use the **`--select`** (or shorthand **`-s`**) flag in dbt without any additional operators, dbt will **exclusively run the specific model** you have named. 

In your project structure, even though `int_trips_unioned.sql` depends on `stg_green_tripdata` and `stg_yellow_tripdata`, simply selecting the intermediate model will not trigger its parents to run. 

To build other models in the lineage, you would need to use the **`+` operator**, which works as follows:

*   **Upstream Dependencies (`+model`):** If you ran `dbt run --select +int_trips_unioned`, dbt would build the intermediate model **and everything it needs to run** (its parents: `stg_green_tripdata` and `stg_yellow_tripdata`).
*   **Downstream Dependencies (`model+`):** If you ran `dbt run --select int_trips_unioned+`, dbt would build the intermediate model **and anything else that depends on it** (downstream models like `fct_trips`).
*   **Both (`+model+`):** Using a double plus sign (or `+` on both sides) allows you to build all upstream and downstream dependencies for that specific model. 

---

### Question 2. dbt Tests

You've configured a generic test like this in your `schema.yml`:

```yaml
columns:
  - name: payment_type
    data_tests:
      - accepted_values:
          arguments:
            values: [1, 2, 3, 4, 5]
            quote: false
```

Your model `fct_trips` has been running successfully for months. A new value `6` now appears in the source data.

What happens when you run `dbt test --select fct_trips`?

- dbt will skip the test because the model didn't change
- dbt will fail the test, returning a non-zero exit code
- dbt will pass the test with a warning about the new value
- dbt will update the configuration to include the new value

---

#### **Answer of Question 2**

The answer is **dbt will fail the test, returning a non-zero exit code**

**Explanation**

Based on the sources, here is why this happens:

*   **How Generic Tests Work:** Generic tests like `accepted_values` are defined in your `.yml` files to ensure data quality. When you run a test, dbt executes a query designed to find rows that violate your specified logic.
*   **The Definition of Failure:** In dbt, a test is essentially an assertion in SQL format. If the test query returns **more than zero rows**—meaning it found data that does not meet your criteria (like the new value `6` which is not in the accepted list)—the test is considered a **failure**.
*   **Execution Result:** When a test fails, dbt will report the error in the terminal and return a **non-zero exit code**. This is a critical feature for automation, as it can be used in CI/CD pipelines to prevent bad data from being merged or promoted.

**Why the other options are incorrect:**
*   **dbt will skip the test:** dbt tests the data currently residing in your warehouse, not just new or changed code. Even if the model hasn't changed, the test will run against the new data.
*   **dbt will pass with a warning:** By default, dbt tests are configured to fail on errors. While you can manually configure a test to only issue a "warning" severity, the standard behavior for a violation is a failure. 
*   **dbt will update the configuration:** dbt does not automatically modify your source code or configuration files based on the data it finds; it is a tool for transforming and testing data according to the rules **you** define.

---

### Question 3. Counting Records in `fct_monthly_zone_revenue`

After running your dbt project, query the `fct_monthly_zone_revenue` model.

What is the count of records in the `fct_monthly_zone_revenue` model?

- 12,998
- 14,120
- 12,184
- 15,421

---

#### **Answer of Question 3**
```sql
-- Count records
SELECT COUNT(*) as total_records
FROM dbt_prod.fct_monthly_zone_revenue;
```

The answer is **12,184**

---

### Question 4. Best Performing Zone for Green Taxis (2020)

Using the `fct_monthly_zone_revenue` table, find the pickup zone with the **highest total revenue** (`revenue_monthly_total_amount`) for **Green** taxi trips in 2020.

Which zone had the highest revenue?

- East Harlem North
- Morningside Heights
- East Harlem South
- Washington Heights South

---

#### **Answer of Question 4**
```sql
-- Best performing zone for Green taxis in 2020

with green_2020_revenue as (
    select 
        zone,
        borough,
        sum(revenue_monthly_total_amount) as total_revenue,
        sum(total_monthly_trips) as total_trips,
        avg(revenue_monthly_total_amount) as avg_monthly_revenue
    from {{ ref('fct_monthly_zone_revenue') }}
    where service_type = 'Green'
      and revenue_year = 2020
    group by zone, borough
)

select 
    zone,
    borough,
    round(total_revenue::numeric, 2) as total_revenue,
    total_trips,
    round(avg_monthly_revenue::numeric, 2) as avg_monthly_revenue,
    rank() over (order by total_revenue desc) as revenue_rank
from green_2020_revenue
order by total_revenue desc
limit 5
```
or

```sql
SELECT 
    zone,
    borough,
    SUM(revenue_monthly_total_amount) as total_revenue,
    SUM(total_monthly_trips) as total_trips,
    ROUND(AVG(revenue_monthly_total_amount)::numeric, 2) as avg_monthly_revenue
FROM dbt_prod.fct_monthly_zone_revenue
WHERE service_type = 'Green'
  AND revenue_year = 2020
GROUP BY zone, borough
ORDER BY total_revenue DESC
LIMIT 10;
```

The answer is **East Harlem North**

---

### Question 5. Green Taxi Trip Counts (October 2019)

Using the `fct_monthly_zone_revenue` table, what is the **total number of trips** (`total_monthly_trips`) for Green taxis in October 2019?

- 500,234
- 350,891
- 384,624
- 421,509


---

#### **Answer of Question 5**
```sql
SELECT 
    revenue_month,
    SUM(total_monthly_trips) as total_trips_october_2019
FROM dbt_prod.fct_monthly_zone_revenue
WHERE service_type = 'Green'
  AND revenue_year = 2019
  AND revenue_month_num = 10
GROUP BY revenue_month;
```
or 

```sql
SELECT 
    zone,
    SUM(revenue_monthly_total_amount) as total_revenue
FROM dbt_prod.fct_monthly_zone_revenue
WHERE service_type = 'Green'
  AND revenue_year = 2020
GROUP BY zone
ORDER BY total_revenue DESC
LIMIT 5;
```

The answer is **384,624**

---

### Question 6. Build a Staging Model for FHV Data

Create a staging model for the **For-Hire Vehicle (FHV)** trip data for 2019.

1. Load the [FHV trip data for 2019](https://github.com/DataTalksClub/nyc-tlc-data/releases/tag/fhv) into your data warehouse
2. Create a staging model `stg_fhv_tripdata` with these requirements:
   - Filter out records where `dispatching_base_num IS NULL`
   - Rename fields to match your project's naming conventions (e.g., `PUlocationID` → `pickup_location_id`)

What is the count of records in `stg_fhv_tripdata`?

- 42,084,899
- 43,244,693
- 22,998,722
- 44,112,187

---

#### **Answer of Question 6**
```sql
-- FHV record count after filtering

select 
    count(*) as total_records,
    count(distinct dispatching_base_num) as unique_bases,
    min(pickup_datetime) as earliest_pickup,
    max(pickup_datetime) as latest_pickup,
    count(distinct pickup_datetime::date) as distinct_days
from dbt_prod.stg_fhv_tripdata;
```

The answer is **43,244,693**

---
