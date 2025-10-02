# ClickHouse Cluster Setup Guide with ZooKeeper

## Architecture Overview

### Recommended: Separate ZooKeeper Ensemble
- **ClickHouse nodes**: 3 LXC containers
- **ZooKeeper nodes**: 3 LXC containers (minimum 3 for quorum)
- Total: 6 LXC containers

### Alternative: Co-located Setup
- 3 LXC containers running both ClickHouse and ZooKeeper
- Suitable for development/testing only

---

## Prerequisites

Assign static IPs to your LXC containers:

### Separate Setup:
- ClickHouse nodes: `10.0.0.11`, `10.0.0.12`, `10.0.0.13`
- ZooKeeper nodes: `10.0.0.21`, `10.0.0.22`, `10.0.0.23`

### Co-located Setup:
- Combined nodes: `10.0.0.11`, `10.0.0.12`, `10.0.0.13`

---

## Part 1: Separate ZooKeeper Ensemble (Recommended)

### Step 1: Create LXC Containers

```bash
# Create ZooKeeper containers
for i in 1 2 3; do
    pct create 20$i local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
        --hostname zk0$i \
        --memory 2048 \
        --cores 2 \
        --net0 name=eth0,bridge=vmbr0,ip=10.0.0.2$i/24,gw=10.0.0.1 \
        --rootfs local-lvm:8
    pct start 20$i
done

# Create ClickHouse containers
for i in 1 2 3; do
    pct create 10$i local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
        --hostname ch0$i \
        --memory 4096 \
        --cores 4 \
        --net0 name=eth0,bridge=vmbr0,ip=10.0.0.1$i/24,gw=10.0.0.1 \
        --rootfs local-lvm:20
    pct start 10$i
done
```

### Step 2: Install ZooKeeper on ZooKeeper Nodes

Run on **each ZooKeeper node** (zk01, zk02, zk03):

```bash
# Update system
apt update && apt upgrade -y

# Install ZooKeeper
apt install -y zookeeperd

# Stop the service to configure
systemctl stop zookeeper
```

#### Configure ZooKeeper

Create/edit `/etc/zookeeper/conf/zoo.cfg`:

```bash
cat > /etc/zookeeper/conf/zoo.cfg << 'EOF'
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=60

# ZooKeeper ensemble
server.1=10.0.0.21:2888:3888
server.2=10.0.0.22:2888:3888
server.3=10.0.0.23:2888:3888

# Performance tuning
autopurge.snapRetainCount=3
autopurge.purgeInterval=1
EOF
```

#### Set ZooKeeper ID

On **zk01** (10.0.0.21):
```bash
echo "1" > /var/lib/zookeeper/myid
```

On **zk02** (10.0.0.22):
```bash
echo "2" > /var/lib/zookeeper/myid
```

On **zk03** (10.0.0.23):
```bash
echo "3" > /var/lib/zookeeper/myid
```

#### Start ZooKeeper

```bash
systemctl start zookeeper
systemctl enable zookeeper

# Verify status
systemctl status zookeeper
echo stat | nc localhost 2181
```

### Step 3: Install ClickHouse on ClickHouse Nodes

Run on **each ClickHouse node** (ch01, ch02, ch03):

```bash
# Install prerequisites
apt update && apt upgrade -y
apt install -y apt-transport-https ca-certificates dirmngr

# Add ClickHouse repository
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754
echo "deb https://packages.clickhouse.com/deb stable main" | tee /etc/apt/sources.list.d/clickhouse.list

# Install ClickHouse
apt update
apt install -y clickhouse-server clickhouse-client

# Don't start yet - need to configure first
systemctl stop clickhouse-server
```

### Step 4: Configure ClickHouse Cluster

On **each ClickHouse node**, create `/etc/clickhouse-server/config.d/cluster.xml`:

```xml
<clickhouse>
    <!-- Listen on all interfaces -->
    <listen_host>::</listen_host>
    <listen_host>0.0.0.0</listen_host>

    <!-- ZooKeeper configuration -->
    <zookeeper>
        <node>
            <host>10.0.0.21</host>
            <port>2181</port>
        </node>
        <node>
            <host>10.0.0.22</host>
            <port>2181</port>
        </node>
        <node>
            <host>10.0.0.23</host>
            <port>2181</port>
        </node>
    </zookeeper>

    <!-- Macros - CHANGE FOR EACH NODE -->
    <macros>
        <shard>01</shard>
        <replica>replica_01</replica>
        <cluster>my_cluster</cluster>
    </macros>

    <!-- Remote servers (cluster definition) -->
    <remote_servers>
        <my_cluster>
            <!-- Shard 1 -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>10.0.0.11</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>10.0.0.12</host>
                    <port>9000</port>
                </replica>
            </shard>
            <!-- Shard 2 -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>10.0.0.13</host>
                    <port>9000</port>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>
</clickhouse>
```

**Important**: Update the `<macros>` section for each node:

**ch01** (10.0.0.11):
```xml
<macros>
    <shard>01</shard>
    <replica>replica_01</replica>
    <cluster>my_cluster</cluster>
</macros>
```

**ch02** (10.0.0.12):
```xml
<macros>
    <shard>01</shard>
    <replica>replica_02</replica>
    <cluster>my_cluster</cluster>
</macros>
```

**ch03** (10.0.0.13):
```xml
<macros>
    <shard>02</shard>
    <replica>replica_01</replica>
    <cluster>my_cluster</cluster>
</macros>
```

### Step 5: Start ClickHouse

On **each ClickHouse node**:

```bash
systemctl start clickhouse-server
systemctl enable clickhouse-server
systemctl status clickhouse-server
```

### Step 6: Verify Cluster

On **any ClickHouse node**:

```sql
clickhouse-client

-- Check cluster configuration
SELECT * FROM system.clusters WHERE cluster = 'my_cluster';

-- Check ZooKeeper connection
SELECT * FROM system.zookeeper WHERE path = '/';

-- Create a replicated table
CREATE TABLE test_replicated ON CLUSTER my_cluster
(
    id UInt64,
    name String,
    created DateTime
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/test_replicated', '{replica}')
PARTITION BY toYYYYMM(created)
ORDER BY id;

-- Create distributed table
CREATE TABLE test_distributed ON CLUSTER my_cluster AS test_replicated
ENGINE = Distributed(my_cluster, default, test_replicated, rand());

-- Test insert
INSERT INTO test_distributed VALUES (1, 'test', now());

-- Verify data on all nodes
SELECT * FROM test_distributed;
```

---

## Part 2: Co-located Setup (Development Only)

### Step 1: Create LXC Containers

```bash
for i in 1 2 3; do
    pct create 10$i local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
        --hostname node0$i \
        --memory 4096 \
        --cores 4 \
        --net0 name=eth0,bridge=vmbr0,ip=10.0.0.1$i/24,gw=10.0.0.1 \
        --rootfs local-lvm:20
    pct start 10$i
done
```

### Step 2: Install Both Services

Run on **each node**:

```bash
# Update system
apt update && apt upgrade -y

# Install ZooKeeper
apt install -y zookeeperd

# Install ClickHouse
apt install -y apt-transport-https ca-certificates dirmngr
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754
echo "deb https://packages.clickhouse.com/deb stable main" | tee /etc/apt/sources.list.d/clickhouse.list
apt update
apt install -y clickhouse-server clickhouse-client

# Stop services for configuration
systemctl stop zookeeper clickhouse-server
```

### Step 3: Configure ZooKeeper

Create `/etc/zookeeper/conf/zoo.cfg` on each node:

```bash
cat > /etc/zookeeper/conf/zoo.cfg << 'EOF'
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=60

server.1=10.0.0.11:2888:3888
server.2=10.0.0.12:2888:3888
server.3=10.0.0.13:2888:3888

autopurge.snapRetainCount=3
autopurge.purgeInterval=1
EOF
```

Set myid on each node:
- Node 1: `echo "1" > /var/lib/zookeeper/myid`
- Node 2: `echo "2" > /var/lib/zookeeper/myid`
- Node 3: `echo "3" > /var/lib/zookeeper/myid`

### Step 4: Configure ClickHouse

Use the same ClickHouse configuration from Part 1, Step 4, but with localhost ZooKeeper:

```xml
<zookeeper>
    <node>
        <host>10.0.0.11</host>
        <port>2181</port>
    </node>
    <node>
        <host>10.0.0.12</host>
        <port>2181</port>
    </node>
    <node>
        <host>10.0.0.13</host>
        <port>2181</port>
    </node>
</zookeeper>
```

### Step 5: Start Services

On each node:

```bash
systemctl start zookeeper
systemctl enable zookeeper
systemctl start clickhouse-server
systemctl enable clickhouse-server
```

---

## Troubleshooting

### Check ZooKeeper Status
```bash
echo stat | nc localhost 2181
echo mntr | nc localhost 2181
```

### Check ClickHouse Logs
```bash
tail -f /var/log/clickhouse-server/clickhouse-server.log
tail -f /var/log/clickhouse-server/clickhouse-server.err.log
```

### Test ZooKeeper from ClickHouse
```sql
SELECT * FROM system.zookeeper WHERE path = '/';
```

### Check Cluster Status
```sql
SELECT * FROM system.clusters;
SELECT * FROM system.replicas;
```

### Firewall Rules (if needed)
```bash
# ClickHouse
ufw allow 9000/tcp  # Native protocol
ufw allow 8123/tcp  # HTTP interface

# ZooKeeper
ufw allow 2181/tcp  # Client port
ufw allow 2888/tcp  # Follower port
ufw allow 3888/tcp  # Election port
```

---

## Production Recommendations

1. **Use separate ZooKeeper ensemble** - better isolation and scalability
2. **Minimum 3 ZooKeeper nodes** - required for fault tolerance
3. **Use odd number of ZooKeeper nodes** - 3, 5, or 7 for proper quorum
4. **Monitor resources** - ZooKeeper is lightweight but ClickHouse needs RAM
5. **Backup regularly** - both ClickHouse data and ZooKeeper snapshots
6. **Use ReplicatedMergeTree** - for data replication across nodes
7. **Consider ClickHouse Keeper** - native replacement for ZooKeeper (ClickHouse 22.8+)

---

## Next Steps

1. Set up monitoring (Prometheus + Grafana)
2. Configure backup strategy
3. Implement security (authentication, encryption)
4. Tune performance settings based on workload
5. Consider upgrading to ClickHouse Keeper for simpler management