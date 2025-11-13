# 🦾 Apache Kafka + ZooKeeper Cluster Setup Guide (Manual - Ubuntu/Linux)

This guide walks you through setting up a **Kafka Cluster (3 Brokers)** and a **ZooKeeper Cluster (3 Nodes)** manually on Ubuntu or Linux servers (also works on AWS EC2).

---

## 🧠 Overview

### Architecture
```
+-----------------------+
| ZooKeeper Cluster     |
| (3 Nodes)             |
+-----------------------+
          |
+-----------------------+
| Kafka Cluster         |
| (3 Brokers)           |
+-----------------------+
```

Kafka brokers use ZooKeeper to:

- Store metadata (topic, partition, broker info)
- Handle leader election and coordination

---

## 🪜 STEP 1 — Prepare Your Machines

You need 6 servers total (or fewer if testing locally):

| Role | Hostname | Port |
|------|-----------|------|
| Zookeeper 1 | zk1 | 2181 |
| Zookeeper 2 | zk2 | 2181 |
| Zookeeper 3 | zk3 | 2181 |
| Kafka Broker 1 | kafka1 | 9092 |
| Kafka Broker 2 | kafka2 | 9092 |
| Kafka Broker 3 | kafka3 | 9092 |

💡 You can simulate all of them on one machine using different ports for local testing.

---

## 🪜 STEP 2 — Install Java

Kafka requires Java.

```bash
sudo apt update
sudo apt install openjdk-11-jdk -y
java -version
```

You should see output like:
```
openjdk version "11.0.x"
```

---

## 🪜 STEP 3 — Download Kafka

Run this on **each Kafka and ZooKeeper node**:

```bash
cd /opt
sudo wget https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz
sudo tar -xvzf kafka_2.13-3.7.0.tgz
sudo mv kafka_2.13-3.7.0 kafka
```

Now Kafka is installed at `/opt/kafka`.

---

## 🪜 STEP 4 — Configure ZooKeeper Cluster

Go to the ZooKeeper config directory:

```bash
cd /opt/kafka/config
sudo nano zookeeper.properties
```

### 🧩 zk1 — `/opt/kafka/config/zookeeper.properties`
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

Repeat the same file on **zk2** and **zk3**.

Create the **myid** file on each ZooKeeper node:

```bash
echo 1 | sudo tee /tmp/zookeeper/myid    # on zk1
echo 2 | sudo tee /tmp/zookeeper/myid    # on zk2
echo 3 | sudo tee /tmp/zookeeper/myid    # on zk3
```

---

## 🪜 STEP 5 — Start ZooKeeper Cluster

Start ZooKeeper on each node:

```bash
cd /opt/kafka
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
```

✅ Verify ZooKeeper:
```bash
ps -ef | grep zookeeper
```

Logs are stored at:
```
/opt/kafka/logs/zookeeper.out
```

---

## 🪜 STEP 6 — Configure Kafka Brokers

Edit Kafka config for each broker:

```bash
cd /opt/kafka/config
sudo nano server.properties
```

### 🧩 Broker 1 — server.properties
```properties
broker.id=1
listeners=PLAINTEXT://kafka1:9092
log.dirs=/tmp/kafka-logs
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
```

### 🧩 Broker 2 — server.properties
```properties
broker.id=2
listeners=PLAINTEXT://kafka2:9092
log.dirs=/tmp/kafka-logs
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
```

### 🧩 Broker 3 — server.properties
```properties
broker.id=3
listeners=PLAINTEXT://kafka3:9092
log.dirs=/tmp/kafka-logs
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
```

---

## 🪜 STEP 7 — Start Kafka Brokers

Start Kafka one by one on each Kafka node:

```bash
cd /opt/kafka
bin/kafka-server-start.sh -daemon config/server.properties
```

✅ Verify Kafka processes:
```bash
ps -ef | grep kafka
```

Logs:
```
/opt/kafka/logs/server.log
```

---

## 🪜 STEP 8 — Verify Cluster Setup

On any Kafka node, run:

```bash
cd /opt/kafka
bin/zookeeper-shell.sh zk1:2181 ls /brokers/ids
```

Expected output:
```
[1, 2, 3]
```

That means your **Kafka cluster (3 brokers)** is successfully connected to **ZooKeeper cluster (3 nodes)** ✅

---

## 🪜 STEP 9 — Test Kafka

### 🔹 Create a topic
```bash
bin/kafka-topics.sh --create --topic test-topic --bootstrap-server kafka1:9092 --replication-factor 3 --partitions 3
```

### 🔹 List topics
```bash
bin/kafka-topics.sh --list --bootstrap-server kafka1:9092
```

### 🔹 Start a producer
```bash
bin/kafka-console-producer.sh --topic test-topic --bootstrap-server kafka1:9092
```

(Type some messages)

### 🔹 Start a consumer (in another terminal)
```bash
bin/kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server kafka2:9092
```

✅ If you see your producer messages in the consumer, the cluster works perfectly.

---

## 🧠 STEP 10 — Understanding Cluster Working

When a topic is created:

- Kafka creates multiple partitions.
- Partitions are distributed across brokers.
- ZooKeeper stores metadata like:
  - Broker ownership of partitions
  - Leader/Follower info

If any broker fails → ZooKeeper elects a new leader for affected partitions.

This ensures **fault tolerance and high availability**.

---

## ⚙️ Stop Kafka & ZooKeeper

### Stop Kafka:
```bash
bin/kafka-server-stop.sh
```

### Stop ZooKeeper:
```bash
bin/zookeeper-server-stop.sh
```

---

## ✅ Final Summary

| Component | Quantity | Purpose |
|------------|-----------|----------|
| ZooKeeper | 3 Nodes | Metadata, leader election |
| Kafka Brokers | 3 Nodes | Message storage & replication |
| Producer / Consumer | Clients | Send and read messages |
| Kafka ↔ ZooKeeper | Connection | Coordination & cluster state |

---

## 🧩 Real-Time Example — Amazon Order System

| Role | Analogy |
|------|----------|
| ZooKeeper | Team leader managing delivery centers |
| Kafka Brokers | Delivery centers storing orders |
| Producer | Website adding new orders |
| Consumer | Billing system reading orders |

If one broker (center) fails → ZooKeeper assigns another broker → **system keeps running** 🚀
