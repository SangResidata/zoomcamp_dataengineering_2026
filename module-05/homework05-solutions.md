# Module 5 Homework: Data Platforms with Bruin

In this homework, we'll use Bruin to build a complete data pipeline, from ingestion to data reporting.

## Setup

1. Install Bruin CLI: `curl -LsSf https://getbruin.com/install/cli | sh`
2. Initialize the zoomcamp template: `bruin init zoomcamp my-pipeline`
3. Configure your `.bruin.yml` with a DuckDB connection
4. Follow the tutorial in the [main module README](..\05-data-platforms\README.md)

After completing the setup, you should have a working NYC taxi data pipeline.

---

### Question 1. Bruin Pipeline Structure

In a Bruin project, what are the required files/directories?

- `bruin.yml` and `assets/`
- `.bruin.yml` and `pipeline.yml` (assets can be anywhere)
- `.bruin.yml` and `pipeline/` with `pipeline.yml` and `assets/`
- `pipeline.yml` and `assets/` only

---

#### **Answer of Question 1**

The answer is **`.bruin.yml` and `pipeline/` with `pipeline.yml` and `assets/`**

**Explanation**

A Bruin project requires a specific hierarchical structure to function correctly:
- Project Root: At the root level, a Bruin project must contain a `.bruin.yml` file, which defines environments, connections, and secrets. Additionally, the root directory must be initialized so the CLI can navigate the file tree.
- Pipeline Level: Each project contains one or more **pipelines**, each separated into its own folder. Inside this folder, a `pipeline.yml` file is required to define the pipeline's name, schedule, connections, and variables.
- Asset Level: Within each **pipeline** folder, there is a dedicated `assets/` directory. This directory contains the individual asset files (SQL, Python, YAML, or R) that perform the actual data tasks.

---


### Question 2. Materialization Strategies

You're building a pipeline that processes NYC taxi data organized by month based on `pickup_datetime`. Which incremental strategy is best for processing a specific interval period by deleting and inserting data for that time period?

- `append` - always add new rows
- `replace` - truncate and rebuild entirely
- `time_interval` - incremental based on a time column
- `view` - create a virtual table only

---

#### **Answer of Question 2**

The answer is **`time_interval` - incremental based on a time column**

**Explanation**

In the NYC taxi data case, using the `pickup_datetime` as the incremental key ensures that only the data within the `start_date` and `end_date` of the current run is affected.

Comparison:
- `append` simply adds new rows without removing old ones.
- `replace` (or standard table materialization) typically drops and recreates the entire table every time, which is not incremental.
- `time_interval` executes a transaction that first deletes the existing data from the destination table for the specific time interval being processed (using an incremental key like `pickup_datetime`) and then reinserts the new data for that time frame.
- `view` creates a virtual table that does not physically store data, so it doesn't involve delete/insert operations.

---

### Question 3. Pipeline Variables

You have the following variable defined in `pipeline.yml`:

```yaml
variables:
  taxi_types:
    type: array
    items:
      type: string
    default: ["yellow", "green"]
```

How do you override this when running the pipeline to only process yellow taxis?

- `bruin run --taxi-types yellow`
- `bruin run --var taxi_types=yellow`
- `bruin run --var 'taxi_types=["yellow"]'`
- `bruin run --set taxi_types=["yellow"]`

---

#### **Answer of Question 3**

The answer is **`bruin run --var 'taxi_types=["yellow"]'`**

**Explanation**

The concept is `--var KEY=VALUE`
- **The `--var` Flag**: Bruin uses the `variable` tag (represented as `--var` in the CLI) to override custom variables defined in the `pipeline.yml` during runtime.
- **Handling Arrays**: Since the `taxi_types` variable is specifically defined as an **array type** in the configuration, the override value must be provided in a format that Bruin can interpret as a list. Wrapping the array in single quotes (e.g., `'taxi_types=["yellow"]'`) ensures the shell passes the entire bracketed expression to the Bruin CLI correctly.

---

### Question 4. Running with Dependencies

You've modified the `ingestion/trips.py` asset and want to run it plus all downstream assets. Which command should you use?

- `bruin run ingestion.trips --all`
- `bruin run ingestion/trips.py --downstream`
- `bruin run pipeline/trips.py --recursive`
- `bruin run --select ingestion.trips+`

---

#### **Answer of Question 4**

The answer is **`bruin run ingestion/trips.py --downstream`**

**Explanation**

Based on the `run` [Docs](https://getbruin.com/docs/bruin/commands/run.html), add `--downstream` flag will run all downstream asset.

---

### Question 5. Quality Checks

You want to ensure the `pickup_datetime` column in your trips table never has NULL values. Which quality check should you add to your asset definition?

- `name: unique`
- `name: not_null`
- `name: positive`
- `name: accepted_values, value: [not_null]`

---

#### **Answer of Question 5**

The answer is **`name: not_null`**

**Explanation**

Bruin includes built-in quality checks that we can add directly to our asset definition to ensure data integrity before it reaches our final reports.

Specifically:
- **`not_null`**: This check is used to ensure that a column does not contain any NULL values. For example, in the NYC taxi data pipeline, you would apply this to the pickup_datetime column to ensure every trip has a valid timestamp.
- **`unique`**: This check ensures that all values in a column are distinct, which is helpful for ID columns but does not specifically prevent NULLs.

**Automatic Execution**: Once these checks are defined in our YAML configuration, Bruin automatically runs them as soon as the asset finishes its execution. If a check fails (e.g., if a NULL is found in a not_null column), it will be flagged in the logs, allowing you to catch the issue early.

---

### Question 6. Lineage and Dependencies

After building your pipeline, you want to visualize the dependency graph between assets. Which Bruin command should you use?

- `bruin graph`
- `bruin dependencies`
- `bruin lineage`
- `bruin show`

---

#### **Answer of Question 6**

The answer is **`bruin lineage`**

**Explanation**
Based on `lineage` [Docs](https://getbruin.com/docs/bruin/commands/lineage.html), this command helps you understand how a specific asset fits into your pipeline by showing its dependencies.

---

### Question 7. First-Time Run

You're running a Bruin pipeline for the first time on a new DuckDB database. What flag should you use to ensure tables are created from scratch?

- `--create`
- `--init`
- `--full-refresh`
- `--truncate`

---

#### **Answer of Question 7**

The answer is **`--full-refresh`**

**Explanation**

When the `full refresh` tag is passed during a bruin run, the orchestrator will drop the existing table and recreate it using the results of the query. Its function is to truncate and rebuild tables from scratch.

---
