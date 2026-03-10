# Module 6 Homework

In this homework we'll put what we learned about Spark in practice.

For this homework we will be using the Yellow 2025-11 data from the official website:

```bash
wget https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2025-11.parquet
```


## Question 1: Install Spark and PySpark

- Install Spark
- Run PySpark
- Create a local spark session
- Execute spark.version.

What's the output?

> [!NOTE]
> To install PySpark follow this [guide](https://github.com/DataTalksClub/data-engineering-zoomcamp/blob/main/06-batch/setup/pyspark.md)

---

### **Answer of Question 1**

The answer is **Spark 4.4.1**

**Explanation**

Follow these steps:
1. Installation and Prerequisites: Based on the course material, to install Spark and PySpark and retrieve the version, you can check it [here](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/06-batch/setup).
    
    For me, I am using docker and operate everything inside it. You can check and follow the step by step from my `experiment` folder [here](../06-batch/experiment/README.md)
2. Creating a Local Spark Session
To run PySpark and initialize a session, you can use the following Python code in a Jupyter notebook or script:
    ```python
    from pyspark.sql import SparkSession

    # Create a local spark session
    spark = SparkSession.builder \
        .master("local[*]") \
        .appName('test') \
        .getOrCreate()
    ```
    The `.master("local[*]")` argument specifies that Spark should run locally using all available CPU cores.

3. Execution and Output
Once the session is created, you execute the following command to check the version:
    ```python
    spark.version
    ```

---

## Question 2: Yellow November 2025

Read the November 2025 Yellow into a Spark Dataframe.

Repartition the Dataframe to 4 partitions and save it to parquet.

What is the average size of the Parquet (ending with .parquet extension) Files that were created (in MB)? Select the answer which most closely matches.

- 6MB
- 25MB
- 75MB
- 100MB

---

### **Answer of Question 2**

The answer is **25MB**

**Explanation**

Follow these steps:
1. Read the Data: You can read the CSV data into a Spark Dataframe using the `spark.read.csv` method, typically specifying `header=True` and `inferSchema=True` to correctly identify columns and data types.
2. Repartition: To change the number of partitions to 4, use the repartition() method: `df = df.repartition(4)`. This distributes the data across the cluster (or local threads) to enable parallel processing.
3. Save to Parquet: Use the `write.parquet()` method to save the resulting Dataframe. This will create a directory containing the partition files, each ending with the `.snappy.parquet` extension.

Check on my [`homework_06.ipynb`](../06-batch/experiment/src/homework_06.ipynb).

---

## Question 3: Count records

How many taxi trips were there on the 15th of November?

Consider only trips that started on the 15th of November.

- 62,610
- 102,340
- 162,604
- 225,768

---

### **Answer of Question 3**

The answer is **162,604**

**Explanation**

Calculation Method:
1. Using PySpark Dataframe API: This uses the `to_date()` function and the `filter()` transformation.
2. Using Spark SQL: This follows the standard SQL practice to query data like a relational table.

Check on my [`homework_06.ipynb`](../06-batch/experiment/src/homework_06.ipynb).

---

## Question 4: Longest trip

What is the length of the longest trip in the dataset in hours?

- 22.7
- 58.2
- 90.6
- 134.5

---

### **Answer of Question 4**

The answer is **90.6**

**Explanation**

Methodology using Spark
1. Calculate Duration: We would create a new column representing the difference between the drop-off and pickup times (columns `tpep_dropoff_datetime` and `tpep_pickup_datetime`).
2. Conversion to Hours: Using PySpark functions, we would calculate the difference in seconds (often by casting to a long/timestamp or using `unix_timestamp`) and divide by 3600.
3. Find the Maximum: We would then apply the `max()` action, which is described in the sources as an operation that "triggers" the execution of the transformation graph

Check on my [`homework_06.ipynb`](../06-batch/experiment/src/homework_06.ipynb).

---

## Question 5: User Interface

Spark's User Interface which shows the application's dashboard runs on which local port?

- 80
- 443
- 4040
- 8080

---

### **Answer of Question 5**

The answer is **4040**

**Explanation**

Spark's User Interface (UI), which serves as the application's dashboard to monitor jobs and progress, runs by default on local port `4040` and you can access the Spark GUI at the URL http://localhost:4040.

---

## Question 6: Least frequent pickup location zone

Load the zone lookup data into a temp view in Spark:

```bash
wget https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv
```

Using the zone lookup data and the Yellow November 2025 data, what is the name of the LEAST frequent pickup location Zone?

- Governor's Island/Ellis Island/Liberty Island
- Arden Heights
- Rikers Island
- Jamaica Bay

---

### **Answer of Question 6**

The answer is **Governor's Island/Ellis Island/Liberty Island**

**Explanation**

Methodology using Spark
We can solve this by following these steps:
1. Load the additional Zone Lookup CSV using `spark.read.csv(..., header=True, inferSchema=True)`.
2. Join Trips with Zone Data on the location ID and count by zone.
3. Identify the Minimum with sort the resulting DataFrame by the count in ascending order using `orderBy()`. The first record will represent the least frequent pickup location.

Actually there are three zones with the same trip count amount of 1:
- Governor's Island/Ellis Island/Liberty Island
- Arden Heights
- Eltingville/Annadale/Prince's Bay

Check on my [`homework_06.ipynb`](../06-batch/experiment/src/homework_06.ipynb).

---
