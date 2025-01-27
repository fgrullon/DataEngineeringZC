---
# NYC Taxi Data Ingestion and Analysis

## Overview

In this project we set up a Dockerized PostgreSQL database for NYC taxi data, creating a data pipeline for later analysis and deployment the infrastructure using Terraform.
---

## Docker Setup

### PostgreSQL

Start a PostgreSQL in a docker container:

```bash
docker run -it \
    -e POSTGRES_USER="root" \
    -e POSTGRES_PASSWORD="grullon" \
    -e POSTGRES_DB="ny_taxi" \
    -v $(pwd)/ny_taxi_postgres_data:/var/lib/postgressql/data \
    -p 5432:5432 \
    --network=frank_default \
    --name pg-database \
    postgres:13
```

### pgAdmin Container

Launch a pgAdmin container with the following command:

```bash
docker run -it \
    -e PGADMIN_DEFAULT_EMAIL="frank@grullon.com" \
    -e PGADMIN_DEFAULT_PASSWORD="root" \
    -p 8080:80 \
    --network=frank_default \
    --name pgadmin \
    dpage/pgadmin4
```

---

## Data Pipeline

### Building Docker Image

Build a Docker image for the data ingestion pipeline:

```bash
docker build -t taxi_ingest:v001 .
```

### Running Data Pipeline Container

Run the data ingestion pipeline container:

URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz"

````bash
docker run -it \
    --network=frank_default \
    taxi_ingest:v001 \
    --user=root \
    --password=root \
    --host=pgdatabase \
    --port=5432 \
    --db=ny_taxi \
    --table_name=yellow_taxi_data \
    --url=${URL}

---

## Docker Compose

Docker Compose is used to orchestrate multiple containers. It provides a way to define and run multi-container Docker applications. In this project, it is used to define and run the entire stack in a more organized manner.

### Deploying the Stack

To deploy the entire stack using Docker Compose:

```bash
docker-compose up
````

In detached mode:

```bash
docker-compose up -d
```

To stop the containers:

```bash
docker-compose down
```

---

## Terraform Setup

1. Install Terraform and add it to your system path.
2. Initialize Terraform:

   ```bash
   terraform init
   ```

3. View the planned changes:

   ```bash
   terraform plan
   ```

4. Deploy the architecture:

   ```bash
   terraform apply
   ```

5. To destroy the infrastructure:

   ```bash
   terraform destroy
   ```

## Analysis Queries

### Question 3: Trip Segmentation Count

This query retrieves the count of taxi trips in October 2019, segmented by the distance of the trip.

```sql
SELECT
	sum(case when trip_distance <= 1 then 1 else 0 end) as one_mile,
	sum(case when trip_distance > 1  and trip_distance <= 3 then 1 else 0 end) as one_three_mile,
	sum(case when trip_distance > 3  and trip_distance <= 7 then 1 else 0 end) as three_seven_mile,
	sum(case when trip_distance > 7  and trip_distance <= 10 then 1 else 0 end) as seven_ten_mile,
	sum(case when trip_distance > 10 then 1 else 0 end) as over_ten_mile
FROM public.green_taxi_data_2019
where lpep_pickup_datetime >= '2019-10-01'
and lpep_dropoff_datetime < '2019-11-01';
```

### Question 4: Longest trip

This query retrieves the longest trip distance in one day.

```sql
SELECT
    lpep_pickup_datetime,
    lpep_dropoff_datetime,
	trip_distance
FROM public.green_taxi_data_2019
where  date(lpep_pickup_datetime) = date(lpep_dropoff_datetime)
order by trip_distance desc limit 10;
```

### Question 5: Biggest pickup zones

This query provides the top three pickup zones with over 13,000 in total amount focusing on pickups on October 18, 2019.

```sql
SELECT
	count(trip_distance) syna,
	t2."Zone",
	sum(total_amount)
	FROM public.green_taxi_data_2019 t1
	join public.green_taxi_data_zone t2 on t1."PULocationID" = t2."LocationID"
	where  date(lpep_pickup_datetime) = '2019-10-18'
	group by t2."Zone"
	having sum(total_amount) >= 13000
	order by 3 desc;
```

### Question 6: Largest tip

This query retrieves the zone with the largest tip amount for trips that originated in East Harlem North on October 2019.

```sql
SELECT
	t2."Zone",
	tip_amount,
	t3."Zone"
FROM public.green_taxi_data_2019 t1
join public.green_taxi_data_zone t2 on t1."PULocationID" = t2."LocationID"
join public.green_taxi_data_zone t3 on t1."DOLocationID" = t3."LocationID"
where  date(lpep_pickup_datetime) >= '2019-10-01' and date(lpep_dropoff_datetime) <= '2019-10-31'
and t2."Zone" = 'East Harlem North'
order by tip_amount desc limit 1;
```

---
