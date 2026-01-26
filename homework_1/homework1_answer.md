# Module 1 Homework: Docker & SQL  

In this homework we'll prepare the environment and practice
Docker and SQL  

## Question 1. Understanding Docker images

Run docker with the `python:3.13` image. Use an entrypoint `bash` to interact with the container.

What's the version of `pip` in the image?

- 25.3
- 24.3.1
- 24.2.1
- 23.3.1

**Answer: 25.3**

**Explanation:**
Verified by running the `python:3.13` image and checking the pip version:
```bash
docker run --rm --entrypoint bash python:3.13 -c "pip --version"
```
Output:
(Pending output)


## Question 2. Understanding Docker networking and docker-compose

Given the following `docker-compose.yaml`, what is the `hostname` and `port` that pgadmin should use to connect to the postgres database?

```yaml
services:
  db:
    container_name: postgres
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'postgres'
      POSTGRES_DB: 'ny_taxi'
    ports:
      - '5433:5432'
    volumes:
      - vol-pgdata:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: "pgadmin@pgadmin.com"
      PGADMIN_DEFAULT_PASSWORD: "pgadmin"
    ports:
      - "8080:80"
    volumes:
      - vol-pgadmin_data:/var/lib/pgadmin

volumes:
  vol-pgdata:
    name: vol-pgdata
  vol-pgadmin_data:
    name: vol-pgadmin_data
```

- postgres:5433
- localhost:5432
- db:5433
- postgres:5432
- db:5432

If multiple answers are correct, select any 

## Prepare the Data

Download the green taxi trips data for November 2025:

```bash
wget https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet
```

You will also need the dataset with zones:

```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv
```
**Answer: db:5432**

**Explanation:**
In Docker Compose, services within the same network can verify/communicate with each other using the service name or container name as the hostname.
- **Hostname**: The service name is `db`, and the container name is explicitly set to `postgres`. So both `db` and `postgres` are valid hostnames for the internal network.
- **Port**: The internal port for the Postgres container is `5432` (standard Postgres port). The mapping `5433:5432` exposes the internal port 5432 to the host machine's port 5433. However, other containers in the same network (like `pgadmin`) communicate directly via the internal port.

Therefore, `pgadmin` should connect to the database using `db:5432` or `postgres:5432`.
The option `db:5432` is one of the correct choices.


## Question 3. Counting short trips

For the trips in November 2025 (lpep_pickup_datetime between '2025-11-01' and '2025-12-01', exclusive of the upper bound), how many trips had a `trip_distance` of less than or equal to 1 mile?

- 7,853
- 8,007
- 8,254
- 8,421

**Answer: 8007**

**Explanation:**
Query to count trips with distance <= 1 mile within the specified date range:
```sql
SELECT COUNT(*) 
FROM green_taxi_trips 
WHERE lpep_pickup_datetime >= '2025-11-01' AND lpep_pickup_datetime < '2025-12-01'
AND trip_distance <= 1;
```
Result: `8007`


## Question 4. Longest trip for each day

Which was the pick up day with the longest trip distance? Only consider trips with `trip_distance` less than 100 miles (to exclude data errors).

Use the pick up time for your calculations.

- 2025-11-14
- 2025-11-20
- 2025-11-23
- 2025-11-25

**Answer: 2025-11-14**

**Explanation:**
Query to find the day with the longest trip distance (filtering out likely outliers > 100 miles):
```sql
SELECT lpep_pickup_datetime::DATE AS pickup_day, MAX(trip_distance) as max_dist
FROM green_taxi_trips
WHERE trip_distance < 100
AND lpep_pickup_datetime >= '2025-11-01' AND lpep_pickup_datetime < '2025-12-01'
GROUP BY pickup_day
ORDER BY max_dist DESC
LIMIT 1;
```
Result: `2025-11-14` (Distance: 88.03)


## Question 5. Biggest pickup zone

Which was the pickup zone with the largest `total_amount` (sum of all trips) on November 18th, 2025?

- East Harlem North
- East Harlem South
- Morningside Heights
- Forest Hills

**Answer: East Harlem North**

**Explanation:**
Query to sum total amount by pickup zone for 2025-11-18:
```sql
SELECT z."Zone", SUM(t.total_amount) as total
FROM green_taxi_trips t
JOIN zones z ON t."PULocationID" = z."LocationID"
WHERE t.lpep_pickup_datetime::DATE = '2025-11-18'
GROUP BY z."Zone"
ORDER BY total DESC
LIMIT 1;
```
Result: `East Harlem North` (Total: 9281.92)


## Question 6. Largest tip

For the passengers picked up in the zone named "East Harlem North" in November 2025, which was the drop off zone that had the largest tip?

Note: it's `tip` , not `trip`. We need the name of the zone, not the ID.

- JFK Airport
- Yorkville West
- East Harlem North
- LaGuardia Airport

**Answer: Yorkville West**

**Explanation:**
Query to find the dropoff zone with the largest single tip from "East Harlem North":
```sql
SELECT z_do."Zone", MAX(t.tip_amount) as max_tip
FROM green_taxi_trips t
JOIN zones z_pu ON t."PULocationID" = z_pu."LocationID"
JOIN zones z_do ON t."DOLocationID" = z_do."LocationID"
WHERE z_pu."Zone" = 'East Harlem North'
AND t.lpep_pickup_datetime >= '2025-11-01' AND t.lpep_pickup_datetime < '2025-12-01'
GROUP BY z_do."Zone"
ORDER BY max_tip DESC
LIMIT 1;
```
Result: `Yorkville West` (Max Tip: 81.89)


## Terraform

In this section homework we'll prepare the environment by creating resources in GCP with Terraform.

In your VM on GCP/Laptop/GitHub Codespace install Terraform.
Copy the files from the course repo
[here](../../../01-docker-terraform/terraform/terraform) to your VM/Laptop/GitHub Codespace.

Modify the files as necessary to create a GCP Bucket and Big Query Dataset.


## Question 7. Terraform Workflow

Which of the following sequences, respectively, describes the workflow for:
1. Downloading the provider plugins and setting up backend,
2. Generating proposed changes and auto-executing the plan
3. Remove all resources managed by terraform`

Answers:
- terraform import, terraform apply -y, terraform destroy
- teraform init, terraform plan -auto-apply, terraform rm
- terraform init, terraform run -auto-approve, terraform destroy
- terraform init, terraform apply -auto-approve, terraform destroy
- terraform import, terraform apply -y, terraform rm

**Answer: terraform init, terraform apply -auto-approve, terraform destroy**

**Explanation:**
1.  **Downloading provider plugins and setting up backend**: This is handled by `terraform init`. It initializes the working directory containing Terraform configuration files.
2.  **Generating proposed changes and auto-executing the plan**: `terraform apply` generates a plan and executes it. The `-auto-approve` flag skips the interactive approval step, allowing it to execute immediately (auto-execute).
3.  **Remove all resources managed by terraform**: `terraform destroy` parses the state file and removes all resources created by the configuration.

## Submitting the solutions
*   Form for submitting: https://courses.datatalks.club/de-zoomcamp-2026/homework/hw1
