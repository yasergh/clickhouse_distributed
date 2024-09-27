# [ Setting up a local 3-node ClickHouse cluster](https://medium.com/@suffyan.asad1/beginners-guide-to-clickhouse-introduction-features-and-getting-started-55315107399a)

# Introduction

ClickHouse, an open-source Online Analytical Processing (OLAP) database, stands out for its remarkable speed and exceptional performance in various warehousing and analytics scenarios such as analytical and time-series data.

![img](https://miro.medium.com/v2/resize:fit:875/1*Ji0gBOUm1BvvqZUK1VBZWg.png)

The first step in experiencing ClickHouse is to set-up a ClickHouse cluster locally [Ref.](https://clickhouse.com/docs/en/architecture/horizontal-scaling) For running the code examples of this article, I have created a three node, single replica cluster using Docker and docker compose. So first, make sure Docker and Docker Composed are installed on your system.

## Setting up the cluster

Next, clone the following repository:

```bash
git clone https://github.com/SA01/clickhouse-docker-compose-cluster
```

Or alternatively, download and extract the code.

**Starting 3 node ClickHouse cluster**

Next, in the terminal, navigate to the folder containing the cloned repository, and start the cluster using the following command:

<iframe src="https://medium.com/media/e0c1c11aa72d432c453cb1d23e97de08" allowfullscreen="" frameborder="0" height="58" width="680" title="ch2.bash" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 57.9875px; left: 0px;"></iframe>

`docker compose up` 

![img](https://miro.medium.com/v2/resize:fit:875/1*Z4Okv57eQI5MfD5mi12o0w.png)

Run the `docker ps` command to see all three nodes running:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\2eSQPVCq-CqE5txD0bqH-w.png)

**This local cluster has 3 nodes, each with 4GB RAM and 2 CPUs**. It is possible to modify this code to run 2 nodes or 1 node. However, **keep in mind that it is recommended to run ClickHouse Keeper on an odd number of nodes.**

So, if there are 2 ClickHouse nodes, a third node should run only the ClickHouse Keeper. Additionally, in production environments, it is recommended to run ClickHouse Keeper on separate nodes.

![](https://clickhouse.com/docs/assets/images/scaling-out-1-c60c14b5c018fea791fa965c8beeef20.png)

This link contains documentation about horizontally scaling ClickHouse:

## [Scaling out | ClickHouse Docs](https://clickhouse.com/docs/en/architecture/horizontal-scaling)

This page in ClickHouse documentation provides details about ClickHouse cluster configuration:

## Docker-Compose Recipes

this Github repository contains many example ClickHouse configurations that can be used as reference:

[Docker-Compose Recipes - Github](https://github.com/ClickHouse/examples/blob/main/docker-compose-recipes/README.md)

## Connecting to the cluster

Many database clients have ClickHouse connectivity built-in. I prefer using the community edition of DBeaver as database client. 

To connect to ClickHouse’s first node, select to create new connection in DBeaver, and choose the ClickHouse driver. Because we are using the latest ClickHouse version, which at the time of writing this article, is 24.5.3.5, do not use the legacy ClickHouse driver.

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\6OjtpQWk0AO6y8D7JMc6Eg.png)

New Connection interface in DBeaver

Next, set the host to `localhost` and port to `8123` to connect to the first node. Username is `default` and no password:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\-npR_h8goYWmudc7AJxeVg.png)

Connection details to connect to ClickHouse node 1

Next, select Test Connection to make sure it can connect:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\xsPCREj0L0GpvY0yw-bmaQ.png)

Connection successful

Similarly, create 2 more connections, second one to node 2 (port: `8124` keeping the rest same) and third to node 3 (port: `8125` keeping the rest same). Make sure you can connect to all 3 nodes:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\G3d5wBL2pvB5iC06I9UpsA.png)

Connection successful to all 3 nodes

On any node, run the following command:

<iframe src="https://medium.com/media/96782278c8dc6e0c8fd7ef089e4d1d81" allowfullscreen="" frameborder="0" height="58" width="680" title="ch3.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 57.9875px; left: 0px;"></iframe>

Command to print ClickHouse version : `SELECT version();`

It should print the running ClickHouse version. Next, run the SQL code:

```sql
select *
from system.clusters; 
```

<iframe src="https://medium.com/media/57a30aad460bd2cc83253c327d7bd54a" allowfullscreen="" frameborder="0" height="58" width="680" title="ch4.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 57.9875px; left: 0px;"></iframe>

Print ClickHouse cluster details

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\QRSRHXgcysK5CweFPDyNOA.png)

***To shut down the setup, press Ctrl + C on the terminal window or tab where the `docker compose up` command was executed.*** This will terminate the nodes. 

Next, use the `docker compose down` command to clean up the nodes and start fresh if needed. It's better to delete the `clickhouse-server.log` and `clickhouse-server.err.log` files in the `logs/logs_1`, `logs/logs_2`, and `logs/logs_3` folders to isolate logs for each run. These logs are also helpful in diagnosing any issues you may come across.

In production environments, ***it is recommended not to enable the `default` user.*** Instead, create users with appropriate permissions according to their needs and use cases. Please visit this page in the ClickHouse documentation to set up users and relevant settings:

##### [Users and Roles Settings | ClickHouse Docs](https://clickhouse.com/docs/en/operations/settings/settings-users)

We have not loaded any data into ClickHouse at this time, as loading data will be covered later in the article.

# ## MergeTree (and ReplicatedMergeTree) engine family

MergeTree engine family is the most robust and widely used engine family, it is designed for handling high data volume and high volume data ingestion. In this section, we’ll work with MergeTree tables.

For detailed documentation of MergeTree, please visit the links:

#### [MergeTree Engine Family | ClickHouse Docs](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family)

**The data in the MergeTree tables is sorted by the sorting key columns**(or expressions), ***into blocks called granules of 8192 rows*** (this is a configurable parameter with a default value of 8192, which is suitable for most use-cases). If the sorting key contains more than columns or expressions, the data is first sorted by the first column in the sorting key, and within each value of the first column, it is sorted by the second key, and so on. Each granule has a mark, which stores the values of the key columns in for that block.

***Primary key*** is optional, and can be provided if it is different from the sorting key. If no primary key is provided, the sorting key also becomes the primary key of the table.

*The data is sorted by sorting key, and the sorting key marks (values of sorting key columns at the start of each granule) for granules are used to find the rows queried by the query filters*. Therefore, the sorting key should be selected according to the usage patterns of common queries for fast data extraction. ***This way, the MergeTree family engines are required to be tuned for queries that run on the table, otherwise, a lot of data (or all data) will have to be scanned to find the required rows.***

An important aspect of the MergeTree family is the underlying storage of data, and how sorting key and marks work in determining the data to be scanned, and how query filters work. 

The data used for demonstration is the popular TLC Trip Record dataset. The data contains Yellow and Green taxi trip records in New York City, containing pickup and drop off locations, fare, tip, distance etc.

The dataset can be obtained from:

##### [TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

www.nyc.govFrom here, download the Yellow Taxi trip records for year 2023, 2022 and 2021. They are parquet files. In the repository folder, place them in `data/rides_2023`, `data/rides_2022` and `data/rides_2021` folders:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\eArm2T7xsktqHrBhA5Jl6g.png)

**Data Files**

The data folder has been mounted on all nodes of the ClickHouse cluster, to the `/var/lib/clickhouse/user_files` folder. Connect your terminal to node 1 by running the following commands:

<iframe src="https://medium.com/media/560940bbf7baa6a531740bd1caf39255" allowfullscreen="" frameborder="0" height="102" width="680" title="ch6.bash" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 102px; left: 0px;"></iframe>

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\QSKH-077XwckKynFNP-l7Q.png)

You can use the `docker ps` command to list all the running nodes of the cluster and to get their names for the `docker exec` command.

## Creating the rides database, and trips table

Next, lets create the table, and load data into it, but first, lets create a database to hold the tables. Run the following SQL Code from DBeaver (or any client you are using to connect to ClickHouse) on node 1 (or any node).

<iframe src="https://medium.com/media/800c36713ade9bb031a98e942a2eca02" allowfullscreen="" frameborder="0" height="58" width="680" title="ch7.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 57.9875px; left: 0px;"></iframe>

```sql
CREATE database rides ON CLUSTER 'cluster_3S_1R';
```

After that, hit refresh on all 3 nodes of the cluster, and the newly created database should be visible on all 3 nodes:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\Jl_YDk7nLO77-dy88FNbNQ.png)

Rides database accessible on all 3 nodes.

Next, on the first node (or any other one), create the table:

<iframe src="https://medium.com/media/76d8f488f5e521b9e7f45b03577693e8" allowfullscreen="" frameborder="0" height="608" width="680" title="ch8.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 608px; left: 0px;"></iframe>

```sql
CREATE TABLE rides.trips ON CLUSTER 'cluster_3S_1R'
(
    VendorID Int32,
    tpep_pickup_datetime DateTime64(6),
    tpep_dropoff_datetime DateTime64(6),
    passenger_count Nullable(Int64),
    trip_distance Nullable(Float64),
    RatecodeID Nullable(Int64),
    store_and_fwd_flag Nullable(String),
    PULocationID Int32,
    DOLocationID Int32,
    payment_type Nullable(Int64),
    fare_amount Nullable(Float64),
    extra Nullable(Float64),
    mta_tax Nullable(Float64),
    tip_amount Nullable(Float64),
    tolls_amount Nullable(Float64),
    improvement_surcharge Nullable(Float64),
    total_amount Nullable(Float64),
    congestion_surcharge Nullable(Float64),
    Airport_fee Nullable(Float64)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(tpep_pickup_datetime)
ORDER BY (tpep_pickup_datetime, tpep_dropoff_datetime, PULocationID, DOLocationID)
SETTINGS index_granularity = 8192;
```

The `trips` table gets created on the 3 nodes of the database. For now, lets insert data into the table in the first row. Run the following SQL code:

<iframe src="https://medium.com/media/ce2907bfeb11eece77a90899fca753e6" allowfullscreen="" frameborder="0" height="75" width="680" title="ch9.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 75px; left: 0px;"></iframe>

```sql
INSERT INTO rides.trips FROM INFILE '/var/lib/clickhouse/user_files/rides_2023/*.parquet' FORMAT Parquet;
```

Running this query from the client gives the following error:

<iframe src="https://medium.com/media/a26a1c53ce814dabf76a69d7272509ce" allowfullscreen="" frameborder="0" height="97" width="680" title="ch10.txt" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 97px; left: 0px;"></iframe>

```bash
SQL Error [78] [07000]: Code: 78. DB::Exception: Query has infile and was send directly to server. (UNKNOWN_TYPE_OF_QUERY) (version 24.5.3.5 (official build))
, server ClickHouseNode [uri=http://localhost:8123/default, options={use_server_time_zone=false,use_time_zone=false}]
```

This query will have to be executed from the ClickHouse client from the node itself. Connect the terminal to node 1, and run the following commands:

<iframe src="https://medium.com/media/fade06fa1e0d2b73780fd0035336d48c" allowfullscreen="" frameborder="0" height="80" width="680" title="ch11.bash" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 80px; left: 0px;"></iframe>

```BASH
SQL Error [78] [07000]: Code: 78. DB::Exception: Query has infile and was send directly to server. (UNKNOWN_TYPE_OF_QUERY) (version 24.5.3.5 (official build))
, server ClickHouseNode [uri=http://localhost:8123/default, options={use_server_time_zone=false,use_time_zone=false}]
```

This, of course requires ClickHouse client to be present on the node. It should be installed and available on all nodes of the example cluster, otherwise, it can be installed. [Link](https://clickhouse.com/docs/en/interfaces/cli):

Run the same insert query on the ClickHouse client:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\Sc02HSlJZWiZFqgboFOktQ.png)

Note: Please note that insertion may fail if each node has less than 4GB RAM. If you are using a computer that doesn’t have sufficient RAM, the data can be inserted file by file, instead of using `*.parquet` in the insertion query, or a cluster with 2 nodes can be used instead.

The set contains 38.31 million rows. Lets run a row count, and total fare aggregation. It can be run from the client now. Run the following two queries on the first row:

<iframe src="https://medium.com/media/1e66e7b32afaadfe67ef9ea3994458d1" allowfullscreen="" frameborder="0" height="190" width="680" title="ch12.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 190px; left: 0px;"></iframe>

```SQL
SELECT sum(total_amount), avg(total_amount), count(1) AS num_rows
FROM rides.trips t;

explain (
    SELECT sum(total_amount), avg(total_amount), count(1)
    FROM rides.trips t
)
```

These give the following results:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\rY8eG2GhFQW1fOEpVknBZg.png)

The result was obtained very quickly, on my computer, it took a little over 0.5 seconds to sum, average and count all 38 million rows. Lets see the query plan:

<iframe src="https://medium.com/media/3310b9f8a4a6964f8207073378db408f" allowfullscreen="" frameborder="0" height="168" width="680" title="ch12.txt" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 168px; left: 0px;"></iframe>

```sql
explain                                                                       |
------------------------------------------------------------------------------+
Expression ((Project names + Projection))                                     |
  Aggregating                                                                 |
    Expression ((Before GROUP BY + Change column names to column identifiers))|
      ReadFromMergeTree (rides.trips)                                         |
```

This query plan shows simple scan of the entire dataset, followed by the aggregate operations. **Next, lets filter by `tpep_pickup_datetime` which is one of the ordering columns:**

<iframe src="https://medium.com/media/4c6753c91241484835240a28cf62b655" allowfullscreen="" frameborder="0" height="119" width="680" title="ch13.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 119px; left: 0px;"></iframe>

```sql
SELECT sum(total_amount), avg(total_amount), count(1) AS num_rows
FROM rides.trips t
WHERE tpep_pickup_datetime >= '2023-05-01 00:00:00' AND tpep_pickup_datetime < '2023-06-01 00:00:00';
```

This query executes very fast, even faster than the previous one. Execution times vary a lot but are consistently faster than the previous query on the whole set. On my computer, the runs were faster than 0.1 second, and although it used over 3 million rows in the final result, it was really fast.

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\FLBwzLrSVBXfYNe43Xk0hA.png)

For comparison, lets **filter on a column that is not part of the sorting key**, i.e. `passenger_count` and filter on that:

<iframe src="https://medium.com/media/e3e416256655e0afe44420f5004b6c6c" allowfullscreen="" frameborder="0" height="102" width="680" title="ch14.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 102px; left: 0px;"></iframe>

```sql
SELECT sum(total_amount), avg(total_amount), count(1) AS num_rows
FROM rides.trips t
WHERE passenger_count = 6
```

**This query is slower**, as the entire dataset is scanned to create that result. Although the number of rows returned by the filter is significantly smaller, the system cannot take advantage of the filter as the data is not ordered by the `passenger_count` column. On my computer, this query takes 0.3 to 0.7 seconds, which is similar to the time taken by the whole data aggregation example query.

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\HXMawDzuN3uHQxuV86ktxg.png)

Execution time of the query filtering data on passenger_count column

**This demonstrates the advantage of optimizing tables for frequent queries, as less data scanned results in faster query execution over large datasets.**

Next, lets look at materialized views, and the SummingMergeTree engine.

## ## Materialized Views

Materialized views in ClickHouse are pre-populated views that make faster query execution possible by shifting complex operations to `insert` process instead of the queries to retrieve data. Materialized views are refreshed in real time when new data is inserted in source table(s). ClickHouse documentation describes Materialized Views as:

> *Materialized views allow users to shift the cost of computation from query time to insert time, resulting in faster* `***SELECT\***` *queries.*
> 
> *Unlike in transactional databases like Postgres, a ClickHouse materialized view is just a trigger that runs a query on blocks of data as they are inserted into a table. The result of this query is inserted into a second “target” table.*

and

> *Materialized views in ClickHouse are updated in real time as data flows into the table they are based on, functioning more like continually updating indexes.*

Link to documentation:

#### [Materialized View | ClickHouse Docs](https://clickhouse.com/docs/en/materialized-view)

The primary benefit of using materialized views in ClickHouse is the shift of complex processing to the insertion process, making each query simpler and faster if directed to the views. This not only enhances performance but also reduces the burden on the system during query execution. By pre-computing and storing the results of complex queries, materialized views can significantly reduce the time required to retrieve data, making them ideal for real-time analytics and reporting.

![img](https://miro.medium.com/v2/resize:fit:875/1*5udgIcKTNay0EBuVYa-pOA.png)

Shift of costly computations to insertion process. Image has been taken from the ClickHouse documentation at: https://clickhouse.com/docs/en/materialized-view

Here is the syntax of MaterializedView in the CREATE VIEW documentation:

<iframe src="https://medium.com/media/c447eede0d1b19a9265e4cfe345343e0" allowfullscreen="" frameborder="0" height="119" width="680" title="ch15.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 119px; left: 0px;"></iframe>

```sql
CREATE MATERIALIZED VIEW [IF NOT EXISTS] [db.]table_name [ON CLUSTER] [TO[db.]name] [ENGINE = engine] [POPULATE] 
[DEFINER = { user | CURRENT_USER }] [SQL SECURITY { DEFINER | INVOKER | NONE }] 
AS SELECT ...
```

The `POPULATE` flag populates the view with existing data in the source table.** If it is not specified, any new data that gets inserted in the table is inserted in the views, but not the data that exists before the got was created**. However, the documentation advises against using it, as any data being inserted in the table at the time the view is getting created is not inserted in the view.

Additionally, as seen in the syntax, it is possible to specify an engine, which can be different from the table engine of the source table. Choosing the right table engine is the key to derive maximum performance from tables and views.

Lets see this in action, create a view that summarizes rides by date of pickup, vendor ID, and pickup location. It’ll use the `SummingMergeTree` engine to sum the `passenger_count`, `trip_distance`, `total_amount` and `congestion_surcharge` columns:

<iframe src="https://medium.com/media/8df7374db6b106a4c5c416ddf60fc32c" allowfullscreen="" frameborder="0" height="141" width="680" title="ch16.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 141px; left: 0px;"></iframe>

```sql
CREATE MATERIALIZED VIEW rides.trips_by_origin ENGINE SummingMergeTree() ORDER BY (VendorID, pickup_date, PULocationID) POPULATE
AS 
SELECT VendorID, PULocationID, toDate(tpep_pickup_datetime) AS pickup_date, passenger_count, trip_distance, fare_amount, total_amount, congestion_surcharge
FROM rides.trips;
```

**is this can be run on the node2 or node3?**

Here, we have defined the Sorting Key as `VendorID, pickup_date, PULocationID`. The rest of the columns get summed. As `POPULATE` flag is present, the data in the table is also inserted in the view. It takes a few seconds to create the view on my computer because it is running the aggregation on all the data in the table. Once complete, it can be queried:

<iframe src="https://medium.com/media/1eff648dbc481c7e466a367175d030e1" allowfullscreen="" frameborder="0" height="102" width="680" title="ch17.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 102px; left: 0px;"></iframe>

Selecting a few rows from the view

![img](https://miro.medium.com/v2/resize:fit:875/1*UsJDBKClEIh0giX5TauvGA.png)

Next, lets run two queries, one on the view and the second one on the table:

<iframe src="https://medium.com/media/d355d99b96e292071284193fdbcc803e" allowfullscreen="" frameborder="0" height="229" width="680" title="ch18.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 229px; left: 0px;"></iframe>

```sql
SELECT *
FROM rides.trips_by_origin
WHERE VendorID = 1 and PULocationID = 1 and pickup_date = '2023-01-01';


SELECT VendorID, PULocationID, tpep_pickup_datetime, passenger_count, trip_distance, fare_amount, total_amount, congestion_surcharge
FROM rides.trips t 
WHERE VendorID = 1 and PULocationID = 1 and toDate(tpep_pickup_datetime) = '2023-01-01';
```

Here, the first query filters rows with `VendorID` = 1 and `PULocationID` = 1 and `pickup_date` = 2023-01-01. The result is only one row, because the SummingMergeTree engine merges all the rows for each value of the Sorting Key into one row:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\k_Wqbk7QP1BZQVjTIOL7cA.png)

Only one row is returned by running the query on the view

Whereas the second query returns all the rows from the table:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\9anhHOD-YNfEdWVO-dtuGg.png)

One row in the view contains aggregate of multiple rows in the table

**Note that we haven’t used Group By expression in the definition of the materialized view, but due to the use of SummingMergeTree engine, all the rows with the same value of the Sorting Key get collapsed into a single row**, and the numerical columns are summed in this new row. This can be easily verified from the data above. If MergeTree engine is used instead of SummingMergeTree, the view will contain all the rows:

<iframe src="https://medium.com/media/a206b97a8acd1bfa3fad36eb39acf2bf" allowfullscreen="" frameborder="0" height="273" width="680" title="ch19.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 273px; left: 0px;"></iframe>

```sql
CREATE MATERIALIZED VIEW rides.trips_by_origin_test ENGINE MergeTree() ORDER BY (VendorID, pickup_date, PULocationID) POPULATE
AS 
SELECT VendorID, PULocationID, toDate(tpep_pickup_datetime) AS pickup_date, passenger_count, trip_distance, fare_amount, total_amount, congestion_surcharge
FROM rides.trips;

SELECT VendorID, PULocationID, pickup_date, passenger_count, trip_distance, fare_amount, total_amount, congestion_surcharge
FROM rides.trips_by_origin_test t 
WHERE VendorID = 1 and PULocationID = 1 and pickup_date = '2023-01-01';

DROP VIEW rides.trips_by_origin_test;
```

The above query returns the following result:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\QeDDvOJSggjnme7IIzN8yA.png)

If MergeTree engine is used in the view, the rows are not aggregated

Additionally, views can be dropped by using the `DROP VIEW` query.

Next, lets examine the rows in the view by month and year:

<iframe src="https://medium.com/media/66191b12b968852cd45ac574d9ede0c8" allowfullscreen="" frameborder="0" height="124" width="680" title="ch20.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 124px; left: 0px;"></iframe>

```sql
SELECT toMonth(pickup_date) mnth, toYear(pickup_date) AS yr, count(1)
FROM rides.trips_by_origin tbo 
GROUP BY mnth, yr
ORDER BY mnth, yr;
```

# Mutations in ClickHouse

**In ClickHouse, UPDATE and DELETE operations are called mutations**. *Mutations are implemented to be asynchronous processes that run in the background.* Mutations are processed in the background, and their effect is not instantly visible because the requested operations do not get applied immediately. Rather, the asynchronous mutations process run in the background to perform the update or delete operations. It is possible to track progress of mutations, because the commands that issue mutations return immediately. We’ll see how to check progress and completion of mutations later in this section of the article. Mutations are executed with `ALTER TABLE` query clause, and [here is a full list of mutation operations](https://clickhouse.com/docs/en/sql-reference/statements/alter?source=post_page-----55315107399a--------------------------------#mutations)

An important thing to remember here is that ClickHouse is designed as a data warehouse, with the expectation that changes to existing data are infrequent, and in bulk. In general, the data stored in ClickHouse is not expected to change, and the insertions are in batches of large data.

Next, lets see update and delete operations in action:

## ALTER TABLE … UPDATE and DELETE

To update rows, the `ALTER TABLE <table name> UPDATE` query clause is used. Its syntax is:

<iframe src="https://medium.com/media/6b02ba36e1abcc365df7236d80de3d4f" allowfullscreen="" frameborder="0" height="75" width="680" title="ch22.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 75px; left: 0px;"></iframe>

```sql
ALTER TABLE [db.]table [ON CLUSTER cluster] UPDATE column1 = expr1 [, ...] [IN PARTITION partition_id] WHERE filter_expr
```

This operation is asynchronous by default, and it may not start or complete immediately. Additionally, it is not possible to update key columns. Update is a heavy operation and requires updating data across many parts. Lets try to update the number of passengers in each trip by adding 1 to it. The update query will be:

<iframe src="https://medium.com/media/7fdbb368d45aecdcdb63830b21d0cf54" allowfullscreen="" frameborder="0" height="58" width="680" title="ch23.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 57.9875px; left: 0px;"></iframe>

```sql
ALTER TABLE rides.trips UPDATE passenger_count = passenger_count + 1;
```

But, instead of performing the update, it returns with an error:

<iframe src="https://medium.com/media/06a728fa7cfdcc835f82cff03b521748" allowfullscreen="" frameborder="0" height="75" width="680" title="ch24.txt" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 75px; left: 0px;"></iframe>

```bash
SQL Error [62] [07000]: Code: 62. DB::Exception: Syntax error: failed at position 69 (end of query): . Expected one of: token, DoubleColon, Comma, IN PARTITION, WHERE. (SYNTAX_ERROR) 
```

This is because the `WHERE` clause is necessary in the update query syntax. Altering rows is a heavy operation and ClickHouse doesn’t allow it on all the data. Because the where clause is require, a hack is to execute it with a `WHERE TRUE` clause, but this should be done with a caution. The data is our example table is not that big, so lets run it:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\N5qh3y7BmLJFjVvxwLShyw.png)

Update clause returns immediately, and starts the asynchronous UPDATE operation

Status of mutations can be checked by querying the `system.mutations` table:

<iframe src="https://medium.com/media/98c0f47a1269ab8a37d5a04b29d6cb56" allowfullscreen="" frameborder="0" height="80" width="680" title="ch25.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 80px; left: 0px;"></iframe>

```sql
SELECT database, table, command, create_time, is_done
FROM system.mutations;
```

The query returns the following result:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\RN7czQE99sBZWnUCJovQmA.png)

Status of UPDATE operation

We can see in the query that the UPDATE operation is complete, as `is_done` is 1. It is 0 when mutation is in progress.

Next, lets update the pickup date for some rows:

<iframe src="https://medium.com/media/e629bb55bf80f62e6ce68abf96d6c223" allowfullscreen="" frameborder="0" height="75" width="680" title="ch26.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 75px; left: 0px;"></iframe>

```sql
ALTER TABLE rides.trips UPDATE tpep_pickup_datetime = toDateTime('2025-01-01 00:00:00') WHERE tpep_pickup_datetime < '2022-01-01';
```

This query results in the following error:

<iframe src="https://medium.com/media/2be1d8fdceb1762425e2c09b64111bde" allowfullscreen="" frameborder="0" height="75" width="680" title="ch27.txt" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 75px; left: 0px;"></iframe>

```sql
SQL Error [420] [07000]: Code: 420. DB::Exception: Cannot UPDATE key column `tpep_pickup_datetime`. (CANNOT_UPDATE_COLUMN) (version 24.5.3.5 (official build))
```

This update is not allowed because key columns cannot be updated.

Similar to update, delete operation is a mutation, and its syntax is:

<iframe src="https://medium.com/media/c8cf7744a95395d9b50bc077c479e873" allowfullscreen="" frameborder="0" height="58" width="680" title="ch28.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 57.9875px; left: 0px;"></iframe>

```sql
ALTER TABLE [db.]table [ON CLUSTER cluster] DELETE WHERE filter_expr
```

[Link to documentation](https://clickhouse.com/docs/en/sql-reference/statements/alter/delete)

Despite the fact that both UPDATE and DELETE are mutations and are performed asynchronously by default, it is possible to make the query wait till mutations are complete, and it is also possible to configured to wait till all replicas have been updated. This can be done using the `mutations_sync` setting.

Read more [here](https://clickhouse.com/docs/en/operations/settings/settings?source=post_page-----55315107399a--------------------------------#mutations_sync)

## Lightweight DELETE for MergeTree engine

A lightweight DELETE operation is available on table that use the `MergeTree` engine. It uses the `DELETE FROM` syntax, and it works by marking the rows as deleted, making them not appear in subsequent queries, and this is done immediately. The rows marked for deletion are then cleaned up during the next merge operation. Read about the lightweight delete operation, its limitations, pros and cons, and use-cases [here](https://clickhouse.com/docs/en/sql-reference/statements/delete)

## 

## Deleting bulk data by dropping partitions

It is recommended to drop data by dropping entire partitions. This is possible if the filter of the delete operation matches with the partition key of the table. This is another consideration while designing the schema for ClickHouse, bulk delete operations become very fast and simple if entire partitions can be dropped.

The example table `rides.trips` is partitioned by month year. In the create statement of the table, the partition key has been provided as `PARTITION BY toYYYYMM(tpep_pickup_datetime)`. Therefore, we can drop partitions for certain months, deleting all the data for that month. For example, if we want to drop the data for the month of June, 2023, the following query can achieve that:

<iframe src="https://medium.com/media/2a38d3e59e4f2da1b16e837181b75841" allowfullscreen="" frameborder="0" height="58" width="680" title="ch29.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 57.9875px; left: 0px;"></iframe>

```sql
ALTER TABLE rides.trips DROP PARTITION '202306';
```

Now, when the data is queried, it can be observed that the data for the month of June, 2023 is no longer present:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\ZOOpBYznwu_Tc1Iz985MCQ.png)

Summary after deleting the data

In addition to dropping partitions, it is possible to detach and attach partitions. So, to replace or update data in bulk, it is possible to create a temporary table, fill it with data.

Then, after performing validations, attach the new partitions to the final table, detaching partitions containing old data. Then, depending on the requirements, they can be dropped, or attached to a history table.

This process can be depicted in the following diagram:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\k523-hCKk5eTOzQfD4I7_w.png)

Replacement of partition with one containing new data

The process of data replacement by replacing partitions is covered in detail in my [article](https://medium.com/@suffyan.asad1/table-partitioning-in-postgresql-a-beginners-guide-4017635caea2):

Although the article focuses on PostgreSQL, the concepts are generic and applicable here. However, due to differences in database systems, the syntax and some mechanics may vary. Refer to ClickHouse documentation for implementation details.

Additionally, this article covers various mechanisms available in ClickHouse for updating and deleting data, each with their pros and cons and use-cases in detail, and I highly recommend reading it:

#### [Handling Updates and Deletes in ClickHouse]([Handling Updates and Deletes in ClickHouse](https://clickhouse.com/blog/handling-updates-and-deletes-in-clickhouse?source=post_page-----55315107399a--------------------------------))

One more article I highly recommend on partitions in ClickHouse:

#### [Using partitions in Clickhouse](https://medium.com/datadenys/using-partitions-in-clickhouse-3ea0decb89c4)

**In summary, updating and deleting data in ClickHouse involves understanding the asynchronous nature of mutations, the limitations on key columns, and the efficient strategies for bulk operations such as dropping partitions.** By leveraging these features and understanding their implications, you can effectively manage data in ClickHouse and optimize your database operations.

# Distributed table — using the DistributedTable engine

So far, all the tables created reside on one node of the cluster. In ClickHouse, it is possible to create tables, that span multiple nodes. Infact, it is possible to create tables that span all nodes, and all the parts of those tables are replicated on the replica nodes if they exist. Distributed tables can be created using the Distributed engine.

Distributed table doesn’t store any data, and it is a logical entity over a set of physical tables. Distributed table engine has been described in the ClickHouse documentation as:

> *Tables with Distributed engine do not store any data of their own, but allow distributed query processing on multiple servers.*

[Link](https://clickhouse.com/docs/en/engines/table-engines/special/distributed)

To demonstrate, we’ll create three tables, called `rides.trips_part` on all three nodes. The table on on node 1 will contain data for the year 2023, the one on node 2 will contain 2022, and the table on node 3 will contain year 2021:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\ox-8ZXGpj9yXkPi1D_qHSg.png)

Distributed table created using the rides tables on all three nodes.

First step is to create the table on all 3 nodes, and inserting data in them. Connect to one of the 3 nodes using `docker exec` command as demonstrated before. Use `docker ps` command to get the names of pods running the nodes. Then create the tables :

<iframe src="https://medium.com/media/f1ea6c732bf353057ccbff43b044b6a3" allowfullscreen="" frameborder="0" height="757" width="680" title="ch30.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 757px; left: 0px;"></iframe>

```sql
docker exec -it clickhouse-cluster-chnode1-1 /bin/bash
<on the pod>
clickhouse-client
CREATE database if not exists rides  ON CLUSTER 'cluster_3S_1R'

CREATE TABLE rides.trips_part ON CLUSTER 'cluster_3S_1R'
(
    VendorID Int32,
    tpep_pickup_datetime DateTime64(6),
    tpep_dropoff_datetime DateTime64(6),
    passenger_count Nullable(Int64),
    trip_distance Nullable(Float64),
    RatecodeID Nullable(Int64),
    store_and_fwd_flag Nullable(String),
    PULocationID Int32,
    DOLocationID Int32,
    payment_type Nullable(Int64),
    fare_amount Nullable(Float64),
    extra Nullable(Float64),
    mta_tax Nullable(Float64),
    tip_amount Nullable(Float64),
    tolls_amount Nullable(Float64),
    improvement_surcharge Nullable(Float64),
    total_amount Nullable(Float64),
    congestion_surcharge Nullable(Float64),
    Airport_fee Nullable(Float64)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(tpep_pickup_datetime)
ORDER BY (tpep_pickup_datetime, tpep_dropoff_datetime, PULocationID, DOLocationID)
SETTINGS index_granularity = 8192;
```

After running the commands, refresh the connections to all three nodes on DBeaver and the newly created tables should be visible:

![img](E:\NikAmooz\DE Advanced-01\7-DuckDB\Images\iSlStLbE_IF9p7odr9k2qw.png)

**Note:** It is possible to run the create table query with `on cluster <cluster_name>` clause to create the table on all three nodes. But data load will have to be done on each node individually. To get the cluster name, run `show clusters` command.

For more information on running distributed DDL queries on the entire cluster instead of individual nodes, please visit this section of the ClickHouse documentation:

###### [Distributed DDL Queries (ON CLUSTER Clause) | ClickHouse Docs](https://clickhouse.com/docs/en/sql-reference/distributed-ddl)

Next, create the distributed table:

```sql
CREATE TABLE rides.trips_distributed on cluster 'cluster_3S_1R' AS rides.trips_part 
ENGINE Distributed('cluster_3S_1R', rides, trips_part, toMonth(tpep_pickup_datetime) % 3);
```

#### Shard Key

- you can use `rand()` to insert rows in the underlying tables  randomly. 

- you can use `toYear(tpep_pickup_datetime) ` to save data for each year on a sigle node . 

Additionally, we have specified the optional sharding key as `toMonth(tpep_pickup_datetime) % 3`. This is a requirement to insert data into the distributed table, and will be covered later.

Here is the link to the documentation of Distributed table engine, containing syntax and other details about defining and using distributed tables in ClickHouse:

#### [Distributed Table Engine | ClickHouse Docs](https://clickhouse.com/docs/en/engines/table-engines/special/distributed)

Now let's polulate the distributed table. use the `clickhouse-client` on one of the 3-node containers : 

```sql
INSERT INTO rides.trips_distributed FROM INFILE '/var/lib/clickhouse/user_files/nyc_taxi/*.parquet' FORMAT Parquet;

select count(*)
from rides.trips_distributed; 

select count(*)
from rides.trips_part tp ; 


select DISTINCT toMonth(tpep_pickup_datetime) , count(*)
from rides.trips_part
group by 1;


SELECT
    partition,
    name,
    active,
    path
FROM system.parts
WHERE table = 'trips_distributed';

SELECT
    partition,
    name,
    active,
    path
FROM system.parts
WHERE table = 'trips_part';
```

Now, lets check the counts and other data:

<iframe src="https://medium.com/media/c4d16cbd5e7e6da227a27271f02c687f" allowfullscreen="" frameborder="0" height="190" width="680" title="ch34.sql" class="em n fe dz bh" scrolling="no" style="box-sizing: inherit; top: 0px; width: 680px; height: 190px; left: 0px;"></iframe>

```sql
select count(1) from rides.trips_distributed;
explain(select count(1) from rides.trips_distributed);

SELECT  toYear(tpep_pickup_datetime) AS yr, toMonth(tpep_pickup_datetime) mnth, count(1)
FROM rides.trips_distributed dist 
GROUP BY yr, mnth
ORDER BY yr, mnth;
```

The above query returns:

And the second query returns the following query plan:

![img](https://miro.medium.com/v2/resize:fit:530/1*aN4xZoqgHOjOz7TyM2y0fw.png)

Query plan of a query picking data from multiple nodes.

In the query plan, there is `ReadFromRemote (Read from remote replica)`, which indicates that some data has been obtained from other nodes. ClickHouse not only obtains but also partially processes data, including grouping and aggregations, on other nodes before sending them to the node on which the query got executed. This partial pre-processing is then used to assemble the final result.

Finally, the last query returns the following result:

![img](https://miro.medium.com/v2/resize:fit:454/1*YmP2oWwdjGoN2JMWCDVjkQ.png)

Result of the third query

Here, the data for all months of year 2021, 2022 and 2023, with the number of rows can be seen. There are some rows in each file that do not belong to these months, but overall, the result has been pulled from the `rides.trips_part` tables on all three nodes.

## Inserting data into a distributed table

ClickHouse documentation mentions two ways of inserting data into distributed tables:

1. Insert directly into distributed table. Because the distributed table itself is a virtual entity and holds no data, the data is sent to the respective nodes, but it must have a `sharding_key` specified.
2. Insert data directly into the underlying tables on the nodes of the cluster. This method has been described as the mode efficient and flexible method in the documentation.