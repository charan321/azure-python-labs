# Real Time Transactional and Analytical Processing on Azure Database for PostgreSQL - Hyperscale (Citus)

Azure Database for PostgreSQL is a fully managed database-as-a-service based on the open-source Postgres relational database engine. The Hyperscale (Citus) deployment option enables you to scale queries horizontally- across multiple machines, to serve applications that require greater scale and performance. Citus transforms Postgres into a distributed database with features like [co-location](https://docs.citusdata.com/en/stable/get_started/concepts.html#co-location), a distributed SQL engine, reference tables, distributed tables and many more. The combination of [parallelism](https://docs.citusdata.com/en/stable/get_started/concepts.html#parallelism), keeping more data in memory, and higher I/O bandwidth can lead to significant performance improvements

With the latest release, Citus 10 is now available in preview on Azure Hyperscale (Citus) with new capabilities like Columnar Storage, sharding on a single node Postgres machine, Joins between Local PostgreSQL & Citus tables and much more. With Basic Tier, you can now build applications that are scale ready from day one.

In this lab, we will learn about some of the superpowers that Citus brings in to the table by distributing data across multiple nodes. We will explore:

- How to create an Azure Database for PostgreSQL-Hyperscale (Citus) using Azure Portal
- Concepts of Sharding on Hyperscale (Citus) Basic Tier
- Creating schemas and ingesting data into an Hyperscale (Citus) instance
- Using Columnar Storage to reduce storage cost and speedup analytical queries
- Scaling the Hyperscale (Citus)-Basic Tier to Standard Tier
- Rebalancing the data and capturing performance improvements

To test the new features of Citus you can either use:

- [Citus 10 Open Source](https://www.citusdata.com/download/) or;
- [Hyperscale (Citus) on Azure Database for PostgreSQL](https://docs.microsoft.com/azure/postgresql/hyperscale-overview)

**Note:** You can even run Citus on [Docker](https://docs.citusdata.com/en/v10.0/installation/single_node_docker.html). But please note that the docker image is intended to be used for development or testing purposes only and not for production workloads.


# run PostgreSQL with Citus on port 5500
docker run -d --name citus -p 5500:5432 -e POSTGRES_PASSWORD=mypassword citusdata/citus
```

## Prerequisites

- Azure Subscription (e.g. [Free](https://aka.ms/azure-free-account) or [Student](https://aka.ms/azure-student-account))
- An **Azure Database for PostgreSQL-Hyperscale Server-Basic Tier** (Detailed steps are listed [here](https://docs.microsoft.com/azure/postgresql/quickstart-create-hyperscale-basic-tier)). For this lab, we will start with Azure Basic Tier- run queries & capture performance benchmarks and later scale it to Standard Tier to see the performance improvements introduced by horizantal scaling of nodes.
- You will also need [psql](https://www.postgresql.org/download/) (Ver 11 is recommended), which is included in [Azure Cloud Shell](https://docs.microsoft.com/en-ca/azure/cloud-shell/overview).
- [Optional] If you want you can also run Citus open source on your laptop as a single Docker container!
    ```bash
    # run PostgreSQL with Citus on port 5500
    docker run -d --name citus -p 5500:5432 -e POSTGRES_PASSWORD=mypassword citusdata/citus
    ```
   
## Connecting to the Hyperscale (Citus) Database
You can create the tables by using standard PostgreSQL CREATE TABLE commands as shown below:

```sql
-- re-initializing database
DROP OWNED BY citus;

CREATE SCHEMA IF NOT EXISTS covid19;

SET search_path='covid19';

-- Sequences

CREATE SEQUENCE covid19.area_reference_id_seq
    INCREMENT 1
    START 1
    MINVALUE 1
    MAXVALUE 2147483647
    CACHE 1;

CREATE SEQUENCE covid19.metric_reference_id_seq
    INCREMENT 1
    START 1
    MINVALUE 1
    MAXVALUE 2147483647
    CACHE 1;

CREATE SEQUENCE covid19.release_reference_id_seq
    INCREMENT 1
    START 1
    MINVALUE 1
    MAXVALUE 2147483647
    CACHE 1;

-- Tables

CREATE TABLE covid19.area_reference
(
    id integer NOT NULL DEFAULT nextval('area_reference_id_seq'::regclass),
    area_type character varying(15) COLLATE pg_catalog."default" NOT NULL,
    area_code character varying(12) COLLATE pg_catalog."default" NOT NULL,
    area_name character varying(120) COLLATE pg_catalog."default" NOT NULL,
    unique_ref character varying(26) COLLATE pg_catalog."default" NOT NULL DEFAULT "substring"(((now())::character varying)::text, 0, 26),
    CONSTRAINT area_reference_pkey PRIMARY KEY (area_type, area_code),
    CONSTRAINT area_reference_id_key UNIQUE (id),
    CONSTRAINT unq_area_reference_ref UNIQUE (unique_ref)
);


CREATE TABLE covid19.metric_reference
(
    id integer NOT NULL DEFAULT nextval('metric_reference_id_seq'::regclass),
    metric character varying(120) COLLATE pg_catalog."default" NOT NULL,
    released boolean NOT NULL DEFAULT false,
    metric_name character varying(150) COLLATE pg_catalog."default",
    source_metric boolean NOT NULL DEFAULT false,
    CONSTRAINT metric_reference_pkey PRIMARY KEY (id),
    CONSTRAINT metric_reference_metric_key UNIQUE (metric)
);
```
Output:
```text
citus=> SELECT pg_size_pretty(citus_total_relation_size('time_series_250421_to_290421') + citus_total_relation_size('time_series_300421_to_040521'));
 pg_size_pretty
----------------
 153 MB
(1 row)

citus=> SELECT pg_size_pretty(citus_total_relation_size('time_series_250421_to_290421'));
 pg_size_pretty
----------------
 121 MB
(1 row)
```


With time as data grows, Hyperscale (Citus) gives you a flexibility to compress your old partitions to save storage cost just by running below simple command that uses table access method to compress the data:

```sql

Can you see the benefit of using **Columnar** storage- we got a compression ratio of about 5x for `time_series_250421_to_290421` partition. Another important aspect to notice here is that, the relation `time_series` now has both columnar storage as well as row-based storage. This is what we call as **HTAP**-(Hybrid Transactional/Analytical Processing) wherein the same database can be used for both analytical and transactional workloads.

We see that relation `time_series` has a attribute called `payload` of `jsonb` type which stores time-series metrics related to Covid-19 in UK. It also stores Foreign Keys to other tables like `area_reference`, `metric_reference` and `release_reference`. We can use this dataset to identify the no. of Covid-19 tests done in an area on a given date:

```sql
SELECT 
area_code,
area_name,
date,
MAX((payload -> 'value')::INT) AS Tests_Conducted
FROM covid19.time_series AS ts
JOIN covid19.area_reference AS ar ON ar.id = ts.area_id
JOIN covid19.metric_reference AS mr ON mr.id = ts.metric_id
JOIN covid19.release_reference AS rr ON rr.id = ts.release_id
WHERE date = '2021-04-27' 
AND metric = 'newVirusTestsRollingSum'
AND (payload -> 'value')  NOTNULL 
GROUP BY area_code, area_name, date;
```
Output:
```text
 area_code |    area_name     |    date    | tests_conducted
-----------+------------------+------------+-----------------
 S92000003 | Scotland         | 2021-04-27 |          127272
 N92000002 | Northern Ireland | 2021-04-27 |           77090
 W92000004 | Wales            | 2021-04-27 |           66246
 E92000001 | England          | 2021-04-27 |         6680473
 K02000001 | United Kingdom   | 2021-04-27 |         6951081
(5 rows)

Time: 615.612 ms
```

That was quick, isn't it - that too when we are using Citus on single node machine. You can imagine the performance we will get when we will add more nodes ot the cluster.If we look at the query above, we will observe that the query ran efficiently because we have distributed our tables such that the data is [co-located](https://docs.citusdata.com/en/stable/get_started/concepts.html#co-location) with minimal cross-shard operations.

Let's run another query that will generate stats for total no. of first dose vaccinations given across various areas in UK.

```sql
SELECT 
area_type,
area_code,
MAX(date) AS date,
MAX((payload -> 'value')::FLOAT) AS first_dose
FROM (
	SELECT *
	FROM covid19.time_series AS tm
	JOIN covid19.release_reference AS rr ON rr.id = release_id
	JOIN covid19.metric_reference AS mr ON mr.id = metric_id
	JOIN covid19.area_reference AS ar ON ar.id = tm.area_id
	 ) AS ts
WHERE date > (now() - INTERVAL '30 days')
AND metric = 'cumPeopleVaccinatedFirstDoseByPublishDate'
AND (payload -> 'value') NOTNULL
GROUP BY area_type, area_code;
```
Output:
```text
 area_type | area_code |    date    | first_dose
-----------+-----------+------------+------------
 nation    | E92000001 | 2021-05-03 |   29025049
 overview  | K02000001 | 2021-05-03 |   34667904
 nation    | S92000003 | 2021-05-03 |    2833761
 nation    | W92000004 | 2021-05-03 |    1864400
 nation    | N92000002 | 2021-05-03 |     944694
(5 rows)

Time: 1106.661 ms (00:01.107)
```

Let's try to see how a transactional query will perform on the same cluster.

```sql
Output:
![image](https://user-images.githubusercontent.com/41684987/117805620-ccfdc400-b276-11eb-9d2d-7592b05378c0.png)

Output:
![image](https://user-images.githubusercontent.com/41684987/117832491-283daf80-b293-11eb-810b-9e66e090127e.png)

That was quick, isn't it - that too when we are using Citus on single node machine. You can imagine the performance we will get when we will add more nodes ot the cluster.If we look at the query above, we will observe that the query ran efficiently because we have distributed our tables such that the data is [co-located](https://docs.citusdata.com/en/stable/get_started/concepts.html#co-location) with minimal cross-shard operations.

Let's run another query that will generate stats for total no. of first dose vaccinations given across various areas in UK.

Output:
![image](https://user-images.githubusercontent.com/41684987/117937719-ef015000-b323-11eb-8249-b34b65b977a5.png)

So we see that with Hyperscale (Citus)- you can run both transactional and analytical workloads on the same machine.
Now that we are familiar with columnar and how to query data on Hyperscale (Citus), lets move on to explore another important (infact most important) capability of Hyperscale (Citus):

**The Power of Horizontal Scaling**

For this, I would request you to goto the [Azure portal](https://portal.azure.com/) again, select your Azure Database for PostgreSQL-Hyperscale (Citus) server and under **Compute + storage** section upgrade from **Basic** tier to **Standard** & increase **Worker node count** to **4** nodes as shown in screenshot below. 

![image](https://user-images.githubusercontent.com/41684987/117833371-ebbe8380-b293-11eb-82ee-77a0243dd4e3.png)

For this query as well, we see similar improvements in the overall run time. 
As you can see, we've got perfectly normal SQL running in a distributed environment with no changes to our actual queries. This is a very powerful tool for scaling PostgreSQL to any size you need without dealing with the traditional complexity of distributed systems.

## Next steps

If you do not want to keep and continue to be billed for the Azure database for Postgres-Hyperscale (Citus) server that we provisioned at the beginning of the lab, you can [delete](https://docs.microsoft.com/azure/postgresql/howto-hyperscale-read-replicas-portal#:~:text=To%20delete%20a%20server%20group,Select%20Delete.) it via the Azure Portal.

You have successfully completed this lab. If you are interested in learning more about Hyperscale (Citus) please refer to our [Quickstart](https://docs.microsoft.com/azure/postgresql/hyperscale/) guide.
