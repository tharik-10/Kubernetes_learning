# 🧠 Apache Kafka with ZooKeeper — Concepts, Configuration & Use Cases

## 📘 Overview

Apache Kafka is a **distributed streaming platform** designed for **high-throughput, fault-tolerant, real-time data streaming**.

ZooKeeper acts as the **coordination service** that manages and synchronizes Kafka brokers, helping them form a cluster.

Kafka + ZooKeeper combination ensures **high availability, reliability, and scalability**.

---

## ⚙️ Architecture Overview

### 🔹 ZooKeeper Cluster
- Stores metadata about Kafka cluster (brokers, topics, partitions)
- Handles **leader election** for brokers and partitions
- Ensures **synchronization** among brokers

### 🔹 Kafka Cluster
- Consists of **multiple brokers** (servers)
- Each broker stores **topics** divided into **partitions**
- Provides **replication** and **load balancing**

### 🔹 High-Level Flow

```
Producer → Kafka Broker(s) → ZooKeeper (metadata management)
Consumer ← Kafka Broker(s)
```

---

## 🧩 Key Components

| Component | Description |
|------------|--------------|
| **Producer** | Publishes messages (records) to Kafka topics |
| **Consumer** | Subscribes and reads data from topics |
| **Broker** | Kafka server that stores and serves messages |
| **Topic** | Logical channel for message streams |
| **Partition** | Unit of parallelism within a topic |
| **Replication Factor** | Number of copies of partition data for fault tolerance |
| **ZooKeeper** | Manages broker registration, configuration, and leader election |

---

## 🧰 ZooKeeper Configuration (`zookeeper.properties`)

Located at: `/opt/kafka/config/zookeeper.properties`

### 🧾 Common Configuration Parameters

| Parameter | Description | Example |
|------------|--------------|----------|
| `dataDir` | Directory where ZooKeeper stores data and logs | `/tmp/zookeeper` |
| `clientPort` | Port where clients connect to ZooKeeper | `2181` |
| `tickTime` | Basic time unit in milliseconds for ZooKeeper heartbeat | `2000` |
| `initLimit` | Number of ticks for followers to connect to leader | `5` |
| `syncLimit` | Ticks allowed for followers to sync with leader | `2` |
| `server.X` | Specifies each ZooKeeper node with host and ports | `server.1=zk1:2888:3888` |

Each ZooKeeper node also needs an ID file:
```bash
echo 1 > /tmp/zookeeper/myid    # zk1
echo 2 > /tmp/zookeeper/myid    # zk2
echo 3 > /tmp/zookeeper/myid    # zk3
```

---

## ⚙️ Kafka Configuration (`server.properties`)

Located at: `/opt/kafka/config/server.properties`

### 🧾 Key Parameters Explained

| Parameter | Description | Example |
|------------|--------------|----------|
| `broker.id` | Unique ID for each Kafka broker | `1`, `2`, `3` |
| `listeners` | Defines how the broker listens for connections | `PLAINTEXT://kafka1:9092` |
| `log.dirs` | Directory for storing Kafka logs (topic data) | `/tmp/kafka-logs` |
| `zookeeper.connect` | ZooKeeper connection string for cluster coordination | `zk1:2181,zk2:2181,zk3:2181` |
| `num.partitions` | Default partitions per topic | `3` |
| `log.retention.hours` | Duration to retain messages | `168` (1 week) |
| `auto.create.topics.enable` | Automatically create topics when referenced | `true` |
| `offsets.topic.replication.factor` | Replication for internal offset topics | `3` |

---

## 🏗️ Kafka + ZooKeeper Cluster Example

### Cluster Components

| Role | Hostname | Port |
|------|-----------|------|
| ZooKeeper 1 | zk1 | 2181 |
| ZooKeeper 2 | zk2 | 2181 |
| ZooKeeper 3 | zk3 | 2181 |
| Kafka Broker 1 | kafka1 | 9092 |
| Kafka Broker 2 | kafka2 | 9092 |
| Kafka Broker 3 | kafka3 | 9092 |

### ZooKeeper Configuration Example (`zookeeper.properties`)

```properties
dataDir=/tmp/zookeeper
clientPort=2181
tickTime=2000
initLimit=5
syncLimit=2
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

### Kafka Configuration Example (`server.properties`)

```properties
broker.id=1
listeners=PLAINTEXT://kafka1:9092
log.dirs=/tmp/kafka-logs
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
num.partitions=3
log.retention.hours=168
auto.create.topics.enable=true
```

---

## 🔄 Cluster Workflow

1. **ZooKeeper starts** and forms quorum (majority nodes connected)
2. **Kafka brokers connect** to ZooKeeper and register themselves
3. ZooKeeper assigns **controller** broker (leader)
4. Producers send data to **topic partitions**
5. Kafka replicates data across brokers for reliability
6. Consumers read messages from topics
7. If a broker fails → ZooKeeper triggers leader re-election automatically

---

## 🔍 Verification Commands

### ✅ Check running brokers
```bash
bin/zookeeper-shell.sh zk1:2181 ls /brokers/ids
```

Output:
```
[1, 2, 3]
```

### ✅ List topics
```bash
bin/kafka-topics.sh --list --bootstrap-server kafka1:9092
```

### ✅ Describe topic details
```bash
bin/kafka-topics.sh --describe --topic test-topic --bootstrap-server kafka1:9092
```

---

## 💡 Real-World Use Cases

| Use Case | Description |
|-----------|-------------|
| **Log Aggregation** | Collect and centralize application logs from multiple sources |
| **Real-Time Analytics** | Stream data for fraud detection, metrics, or dashboards |
| **Messaging Queue Replacement** | Replace RabbitMQ/ActiveMQ for scalable event streaming |
| **Microservices Communication** | Enable decoupled, asynchronous communication |
| **IoT Data Streaming** | Collect sensor or telemetry data at scale |
| **ETL Pipelines** | Move data between systems with transformations |

---

## 🧠 Advantages of Using ZooKeeper with Kafka

- Centralized broker metadata management  
- Reliable leader election  
- Simplified cluster coordination  
- Strong fault tolerance  
- Ensures consistent configuration across brokers  

---

## ⚙️ Operational Best Practices

1. **Use 3–5 ZooKeeper nodes** for quorum-based stability  
2. **Keep ZooKeeper and Kafka logs separate** for clarity  
3. **Avoid placing all brokers on the same machine**  
4. **Use DNS hostnames** (not IPs) for broker and ZK config  
5. **Monitor** ZooKeeper latency and Kafka metrics regularly  
6. **Backup** `/tmp/zookeeper` and Kafka logs for disaster recovery  

---

## 🧾 Summary

| Component | Count | Purpose |
|------------|--------|----------|
| ZooKeeper Nodes | 3 | Coordination & Metadata |
| Kafka Brokers | 3 | Message storage & replication |
| Topics | Many | Logical channels for data |
| Producers | Many | Data generators |
| Consumers | Many | Data processors |

---

## 🏁 Example Workflow

```
1️⃣ ZooKeeper starts → forms quorum
2️⃣ Kafka brokers start → register in ZooKeeper
3️⃣ Controller elected → manages cluster metadata
4️⃣ Producer publishes → broker stores in partitions
5️⃣ Consumers subscribe → read from brokers
6️⃣ Broker down? → ZooKeeper triggers new leader election
```

---

## 🧩 Real Example: E-Commerce System

| Component | Role |
|------------|------|
| Producer | Website adding new orders |
| Kafka Topic | `orders` topic stores events |
| Consumers | Billing, Notification, Inventory microservices |
| ZooKeeper | Manages broker coordination and leader election |

If one broker fails → ZooKeeper reassigns partition leadership → system keeps running smoothly ✅

---

## 🧱 Conclusion

Kafka + ZooKeeper provides:
- **High Availability**
- **Scalability**
- **Strong coordination**
- **Guaranteed message delivery**

While new versions of Kafka can run **without ZooKeeper (KRaft)**, the ZooKeeper-based setup remains **the foundation for understanding distributed Kafka clusters**.

---
