# ClickHouse Cluster Configuration: 3 Nodes, 2 Shards, 2 Replicas

In this tutorial, we'll walk through the process of configuring a ClickHouse cluster with 3 nodes, 3 shards, and 2 replicas per shard. We'll also cover how to create distributed tables on the replicated local tables.

## 1. Server Configuration

First, let's update the server configuration to reflect our new cluster structure. We'll need to modify the `remote_servers` section in the ClickHouse configuration file on each node.

```xml
<clickhouse>
  <remote_servers replace="true">
    <cluster_2S_2R> <!-- Cluster Name -->
      <!-- Shard 1 -->
      <shard>
        <internal_replication>true</internal_replication>
        <replica>
          <host>chnode1</host>
          <port>9000</port>
        </replica>
        <replica>
          <host>chnode3</host>
          <port>9000</port>
        </replica>
      </shard>

      <!-- Shard 2 -->
      <shard>
        <internal_replication>true</internal_replication>
        <replica>
          <host>chnode2</host>
          <port>9000</port>
        </replica>
        <replica>
          <host>chnode4</host>
          <port>9000</port>
        </replica>
      </shard>
    </cluster_2S_2R>
  </remote_servers>
</clickhouse>
```

This configuration defines a cluster named `cluster_2S_2R` with 3 shards and 2 replicas per shard.

#### Important Tip

> *for every replica node, you need a separate*

## 2. Macros Configuration

Next, we need to update the macros configuration on each node to reflect its role in the cluster. Here's how the macros should be set for each node:

### Node 1 (chnode1)

```xml
<clickhouse>
  <macros>
    <shard>1</shard>
    <replica>replica_1</replica>
  </macros>
</clickhouse>
```

### Node 2 (chnode2)

```xml
<clickhouse>
  <macros>
    <shard>2</shard>
    <replica>replica_1</replica>
  </macros>
</clickhouse>
```

### Node 3 (chnode3)

```xml
<clickhouse>
  <macros>
    <shard>1</shard>
    <replica>replica_2</replica>
  </macros>
</clickhouse>
```

## Node 4 (chnode4)

```xml
<clickhouse>
  <macros>
    <shard>2</shard>
    <replica>replica_2</replica>
  </macros>
</clickhouse>
```

## 3. DDL Queries

Now, let's update our DDL queries to work with the new cluster configuration:

```sql
-- Create the database
CREATE DATABASE IF NOT EXISTS rides ON CLUSTER 'cluster_2S_2R';

-- Create the local table on each node
CREATE TABLE rides.trips_local ON CLUSTER 'cluster_2S_2R'
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
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/rides/trips_local', '{replica}')
PARTITION BY toYYYYMM(tpep_pickup_datetime)
ORDER BY (tpep_pickup_datetime, tpep_dropoff_datetime, PULocationID, DOLocationID)
SETTINGS index_granularity = 8192;

-- Create the distributed table

CREATE TABLE rides.trips_distributed ON CLUSTER 'cluster_2S_2R'
AS rides.trips_local
ENGINE = Distributed('cluster_2S_2R', rides, trips_local, toMonth(tpep_pickup_datetime) % 2);

SELECT 
    hostName() AS host,
    name AS table_name,
    engine,
    data_paths,
    metadata_path
FROM system.tables
WHERE database = 'rides' AND name = 'trips_distributed';
```

In this updated DDL:

1. We've changed the cluster name to `cluster_2S_2R`.

2. We've renamed `trips_part` to `trips_local` to better reflect its role as a local table.

3. We've changed the engine of the local table to `ReplicatedMergeTree` to enable replication.

4. We've updated the distributed table to use the new local table name and a random distribution of data across shards.

### **ReplicatedMergeTree**

   The `ReplicatedMergeTree` engine is a specialized version of the `MergeTree` engine used in ClickHouse for **replication**. When dealing with a **replicated cluster**, each replica within a shard needs to synchronize its data with the other replicas. The `ReplicatedMergeTree` engine achieves this by using a **ZooKeeper-compatible service** for coordination, which could be **Zookeeper** or **ClickHouse Keeper**.

   In your case, you're using **ClickHouse Keeper**, which is a built-in alternative to Zookeeper, designed to manage metadata and replication processes for ClickHouse clusters.

### Syntax of `ReplicatedMergeTree`

   Here’s a basic breakdown of the syntax:

```sql
ReplicatedMergeTree('/clickhouse/tables/{shard}/rides/trips_local', '{replica}')
```

   This tells ClickHouse how to manage replication for each table replica within a shard.

#### Parameters:

1. **`/clickhouse/tables/{shard}/rides/trips_local`**:
   
   - This is the **ZooKeeper (or ClickHouse Keeper) path** that identifies the table. Each shard will substitute `{shard}` with its own shard number to ensure the paths for the different shards are unique.
   - Example for **Shard 1**: `/clickhouse/tables/1/rides/trips_local`
   - This path is used by ClickHouse Keeper to store the metadata for the replicated tables in the cluster.

2. **`{replica}`**:
   
   - This is the **replica identifier** that is automatically replaced by the replica's value from the `macros.xml` configuration on each node. It ensures each replica within the shard gets its own unique identifier.
   - The value of `{replica}` will correspond to the `<replica>` macro in the **`macros.xml`** file. So, on one node, it will be `replica_1`, and on the other replica in the shard, it will be `replica_2`.
   
   ### Why Use `ReplicatedMergeTree`?
- **Automatic Replication**: Data written to one replica in a shard is automatically replicated to other replicas in the same shard.

- **High Availability**: If one replica goes down, another replica in the same shard can handle queries, ensuring fault tolerance.

- **Consistency**: Using ClickHouse Keeper, the replication process is coordinated across the cluster to ensure consistency between replicas.
  
  ### Using ClickHouse Keeper Instead of Zookeeper

- **ClickHouse Keeper** is a built-in lightweight alternative to Zookeeper for managing metadata, replication, and distributed table coordination.

- The configuration and functionality are nearly identical to using Zookeeper, but since it’s part of ClickHouse, it’s easier to set up and manage for ClickHouse-specific use cases.
  
  You don't need to make any changes to the `ReplicatedMergeTree` syntax if you're using ClickHouse Keeper instead of Zookeeper. The replication path and configuration are the same.

## 4. Data Insertion

To insert data into the cluster, you can use the distributed table:

- go into the container shell and run the `clickhouse-client` , then : 

```sql
INSERT INTO rides.trips_distributed
FROM INFILE '/var/lib/clickhouse/user_files/nyc_taxi/*.parquet'
FORMAT Parquet;
```

This will distribute the data across all shards in the cluster.

stop `chnode2` and insert some sample data : (use any other nodes for executing this query)

```sql
INSERT INTO rides.trips_distributed
(
    VendorID,
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    passenger_count,
    trip_distance,
    RatecodeID,
    store_and_fwd_flag,
    PULocationID,
    DOLocationID,
    payment_type,
    fare_amount,
    extra,
    mta_tax,
    tip_amount,
    tolls_amount,
    improvement_surcharge,
    total_amount,
    congestion_surcharge,
    Airport_fee
) VALUES
(1, '2023-09-27 08:00:00', '2023-09-27 08:30:00', 2, 5.2, 1, 'N', 161, 237, 1, 20.5, 0.5, 0.5, 4.0, 0, 0.3, 25.8, 2.5, 0),
(2, '2023-09-27 09:15:00', '2023-09-27 09:45:00', 1, 3.8, 1, 'N', 234, 161, 2, 16.0, 0.5, 0.5, 0, 0, 0.3, 17.3, 2.5, 0),
(1, '2023-09-27 10:30:00', '2023-09-27 11:15:00', 3, 10.5, 2, 'N', 132, 230, 1, 45.0, 1.0, 0.5, 9.0, 5.5, 0.3, 61.3, 2.5, 1.25),
(2, '2023-09-27 12:00:00', '2023-09-27 12:20:00', 1, 2.1, 1, 'N', 163, 229, 3, 11.5, 0.5, 0.5, 2.0, 0, 0.3, 14.8, 2.5, 0),
(1, '2023-09-27 14:45:00', '2023-09-27 15:30:00', 4, 8.7, 1, 'N', 237, 161, 1, 35.0, 1.0, 0.5, 7.0, 0, 0.3, 43.8, 2.5, 0);
```

then query the records count on the `node4`:

```sql
select count(*)
from  rides.trips_local tl  ;
```

bring up the node2 again and run the above query against it. it worked!

## Checking the cluster

Certainly! To check if every table has a mirror (replica) in your ClickHouse cluster, you can use system tables and SQL queries. I'll provide you with a query and explanation on how to verify the replication status of your tables.

Here's a SQL query you can use to check if every table has a mirror: (this command have to be run on every node)

```sql
SELECT 
    database,
    table,
    is_leader,
    total_replicas,
    active_replicas,
    is_readonly
FROM system.replicas
WHERE database NOT IN ('system', 'information_schema', 'INFORMATION_SCHEMA')
ORDER BY database, table;
```

This query will give you information about all replicated tables in your cluster. Here's what each column means:

- `database`: The name of the database containing the table
- `table`: The name of the table
- `is_leader`: Indicates if this replica is the leader (1) or not (0)
- `total_replicas`: The total number of replicas for this table
- `active_replicas`: The number of currently active replicas
- `is_readonly`: Indicates if the table is in read-only mode (1) or not (0)

To interpret the results:

1. If `total_replicas` is 2 for all tables, it means each table has a mirror as expected in your 3S_2R setup.
2. Ideally, `active_replicas` should be equal to `total_replicas` for all tables, indicating all replicas are up and running.
3. For each table, you should see two rows (one for each replica), with one having `is_leader = 1` and the other `is_leader = 0`.

If you want to focus only on tables that might not have proper replication, you can modify the query like this (local view, run it against each node separately):

```sql
SELECT 
    database,
    table,
    is_leader,
    total_replicas,
    active_replicas,
    is_readonly
FROM system.replicas
WHERE database NOT IN ('system', 'information_schema', 'INFORMATION_SCHEMA')
  AND (total_replicas != 2 OR active_replicas != 2)
ORDER BY database, table;
```

This will show only tables where the total number of replicas or active replicas is not 2, which would indicate a potential issue with replication.

To run these queries:

1. Connect to any node in your ClickHouse cluster using the clickhouse-client or your preferred SQL client.
2. Execute the query.
3. Analyze the results to ensure all your tables have the expected replication setup.

Remember, you should run this check on all nodes in your cluster to get a complete picture of the replication status. If you find any discrepancies, you may need to investigate further and potentially fix the replication for those specific tables.

Is there anything else you'd like to know about checking table replication in ClickHouse?

## Conclusion

With these changes, you now have a ClickHouse cluster with 3 nodes, 3 shards, and 2 replicas per shard. The data is distributed across all shards and replicated for fault tolerance. 

Remember to restart your ClickHouse servers after making configuration changes, and always test your setup thoroughly before using it in a production environment.
