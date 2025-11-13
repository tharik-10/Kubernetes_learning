# 🦾 Apache Kafka + ZooKeeper Cluster Setup on AWS EC2 (Private VPC)

This guide provides a complete setup for a **Kafka Cluster (3 Brokers)** and a **ZooKeeper Cluster (3 Nodes)** running on **AWS EC2 instances** using **private IPs** for communication.

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

Kafka brokers use ZooKeeper for:
- Metadata storage (topics, partitions, broker info)
- Leader election and coordination

---

## 🪜 STEP 1 — Launch EC2 Instances

You need **6 EC2 instances** (t2.micro or t3.small are fine for testing).

| Role | Hostname | Ports |
|------|-----------|--------|
| ZooKeeper 1 | zk1 | 2181, 2888, 3888 |
| ZooKeeper 2 | zk2 | 2181, 2888, 3888 |
| ZooKeeper 3 | zk3 | 2181, 2888, 3888 |
| Kafka Broker 1 | kafka1 | 9092 |
| Kafka Broker 2 | kafka2 | 9092 |
| Kafka Broker 3 | kafka3 | 9092 |

✅ **All instances must be in the same VPC and Security Group.**

---

## 🧱 STEP 2 — Configure Security Group

Open these ports **in the EC2 Security Group**:

| Port | Purpose |
|------|----------|
| 22 | SSH access |
| 2181 | ZooKeeper client port |
| 2888 | ZooKeeper quorum port |
| 3888 | ZooKeeper election port |
| 9092 | Kafka broker port |

👉 Allow inbound traffic **only from the internal private IP range** of your VPC (not `0.0.0.0/0`).

---

## 🧭 STEP 3 — Set Hostnames on Each Instance

Example for zk1:
```bash
sudo hostnamectl set-hostname zk1
```

Repeat for zk2, zk3, kafka1, kafka2, kafka3.

✅ Verify with:
```bash
hostname
```

---

## 🧩 STEP 4 — Update `/etc/hosts` File

On **every instance**, edit the hosts file:

```bash
sudo nano /etc/hosts
```

Add all internal private IPs (replace with your actual EC2 IPs):

```
172.31.10.10 zk1
172.31.10.11 zk2
172.31.10.12 zk3
172.31.10.20 kafka1
172.31.10.21 kafka2
172.31.10.22 kafka3
```

✅ Verify connectivity:
```bash
ping zk1
ping kafka1
```

---

## 🪜 STEP 5 — Install Java

Kafka requires Java.

```bash
sudo apt update
sudo apt install openjdk-11-jdk -y
java -version
```

Expected output:
```
openjdk version "11.0.x"
```

---

## 🪜 STEP 6 — Download Kafka

Run this on **each ZooKeeper and Kafka node**:

```bash
cd /opt
sudo wget https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz
sudo tar -xvzf kafka_2.13-3.7.0.tgz
sudo mv kafka_2.13-3.7.0 kafka
```

Kafka is now installed at `/opt/kafka`.

---

## 🪜 STEP 7 — Configure ZooKeeper Cluster

On each ZooKeeper node, edit the configuration:

```bash
cd /opt/kafka/config
sudo nano zookeeper.properties
```

Example (same for all zk nodes):

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

Then, assign each ZooKeeper node a unique ID:

```bash
echo 1 | sudo tee /tmp/zookeeper/myid    # on zk1
echo 2 | sudo tee /tmp/zookeeper/myid    # on zk2
echo 3 | sudo tee /tmp/zookeeper/myid    # on zk3
```

---

## 🪜 STEP 8 — Start ZooKeeper Cluster

On all 3 ZooKeeper nodes:

```bash
cd /opt/kafka
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
```

✅ Verify ZooKeeper:
```bash
ps -ef | grep zookeeper
```

Logs:
```
/opt/kafka/logs/zookeeper.out
```

---

## 🪜 STEP 9 — Configure Kafka Brokers

On each Kafka node, edit:
```bash
cd /opt/kafka/config
sudo nano server.properties
```

### 🧩 Broker 1 — kafka1
```properties
broker.id=1
listeners=PLAINTEXT://kafka1:9092
advertised.listeners=PLAINTEXT://172.31.10.20:9092
log.dirs=/tmp/kafka-logs
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
```

### 🧩 Broker 2 — kafka2
```properties
broker.id=2
listeners=PLAINTEXT://kafka2:9092
advertised.listeners=PLAINTEXT://172.31.10.21:9092
log.dirs=/tmp/kafka-logs
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
```

### 🧩 Broker 3 — kafka3
```properties
broker.id=3
listeners=PLAINTEXT://kafka3:9092
advertised.listeners=PLAINTEXT://172.31.10.22:9092
log.dirs=/tmp/kafka-logs
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
```

---

## 🪜 STEP 10 — Start Kafka Brokers

On each Kafka node:

```bash
cd /opt/kafka
bin/kafka-server-start.sh -daemon config/server.properties
```

✅ Verify Kafka is running:
```bash
ps -ef | grep kafka
```

Logs:
```
/opt/kafka/logs/server.log
```

---

## 🪜 STEP 11 — Verify Cluster Setup

On any Kafka node:

```bash
cd /opt/kafka
bin/zookeeper-shell.sh zk1:2181 ls /brokers/ids
```

Expected output:
```
[1, 2, 3]
```

✅ That means your Kafka cluster (3 brokers) is registered with ZooKeeper (3 nodes).

---

## 🪜 STEP 12 — Test Kafka Functionality

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

Type a few messages.

### 🔹 Start a consumer (in another terminal)
```bash
bin/kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server kafka2:9092
```

✅ If messages appear in the consumer window, your cluster works perfectly.

---

## 🧠 STEP 13 — Understanding Cluster Operation

- Topics are split into multiple partitions.
- Kafka distributes these partitions across brokers.
- ZooKeeper tracks broker IDs, partition leaders, and replicas.
- If a broker fails, ZooKeeper helps elect a new leader.

This provides **fault tolerance and high availability**.

---

## ⚙️ STEP 14 — Stopping Services

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
| Kafka ↔ ZooKeeper | Connection | Cluster coordination |

---

## 🧩 Real-Time Example — Amazon Order System

| Role | Analogy |
|------|----------|
| ZooKeeper | Team leader managing delivery centers |
| Kafka Brokers | Delivery centers storing orders |
| Producer | Website adding new orders |
| Consumer | Billing system reading orders |

If one broker (center) fails, ZooKeeper assigns another → **continuous uptime ensured** 🚀
