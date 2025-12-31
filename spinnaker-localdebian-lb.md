# 🚀 Spinnaker on EC2 with ALB – Complete Step-by-Step Guide

This document describes how to deploy **Spinnaker on a single EC2 instance** using **Halyard (localdebian)** and expose it securely via an **AWS Application Load Balancer (ALB)**. It includes **system optimization**, **dual storage (S3 + Redis)**, **0.0.0.0 binding fixes**, and **Apache configuration for Deck UI**.

---

## 🏗️ Phase 1: AWS Infrastructure (Pre-Setup)

Before logging into the EC2 instance, configure AWS networking to allow Spinnaker traffic.

---

### 1. Target Groups (EC2 Console)

Create the following **Target Groups**:

| Name         | Port | Protocol | Health Check Path |
| ------------ | ---- | -------- | ----------------- |
| spin-deck-tg | 9000 | HTTP     | /                 |
| spin-gate-tg | 8084 | HTTP     | /health           |

---

### 2. ALB Listeners

Configure listeners on your **Application Load Balancer**:

* **Listener 1**: Port `9000` → Forward to `spin-deck-tg`
* **Listener 2**: Port `8084` → Forward to `spin-gate-tg`

---

### 3. EC2 Security Group

Add **Inbound Rules** to the Spinnaker EC2 instance Security Group:

| Type       | Port | Source                |
| ---------- | ---- | --------------------- |
| Custom TCP | 9000 | 0.0.0.0/0 (or ALB SG) |
| Custom TCP | 8084 | 0.0.0.0/0 (or ALB SG) |
| Custom TCP | 8080 | 0.0.0.0/0             |

---

## 🚀 Phase 2: System Optimization & Installation

Run all commands as the **ubuntu** user.

---

### 1. Create a Swap File (Prevents OOM / Killed Errors)

Spinnaker services are memory intensive. Swap avoids Java processes being killed during startup.

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify:

```bash
swapon --show
```

---

### 2. Install Halyard, Java 17, Redis & Utilities

```bash
sudo apt update && sudo apt install -y \
  openjdk-17-jdk \
  net-tools \
  apache2 \
  redis-server

curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh -y --user ubuntu

sleep 20  # Wait for Halyard daemon
```

Verify Halyard:

```bash
hal --version
```

---

## 📦 Phase 3: Dual Storage Configuration (S3 + Redis)

* **S3** → Long-term persistence (applications & pipelines)
* **Redis** → High-speed cache and state

---

### 1. Configure S3 Persistence

```bash
export MY_ACCESS_KEY="your_access_key"
export MY_SECRET_KEY="your_secret_key"
export MY_BUCKET="spinnaker-storage-$(date +%s)"

echo $MY_SECRET_KEY | hal config storage s3 edit \
  --access-key-id $MY_ACCESS_KEY \
  --secret-access-key \
  --region us-east-1 \
  --bucket $MY_BUCKET

hal config storage edit --type s3
```

---

### 2. Configure Redis Persistence (Primary Store)

> Redis provides the fastest performance but must persist to disk.

```bash
# Set Redis as primary storage
hal config storage edit --type redis

# Enable Redis disk persistence
sudo sed -i 's/appendonly no/appendonly yes/' /etc/redis/redis.conf
sudo systemctl restart redis-server
```

Verify:

```bash
redis-cli ping
```

---

## 🔒 Phase 4: Networking & 0.0.0.0 Binding

Ensures Spinnaker services are reachable externally via ALB.

---

### 1. Set External URLs

```bash
export ALB_DNS="your-alb-dns-name.amazonaws.com"

hal config version edit --version 2025.4.0
hal config deploy edit --type localdebian

hal config security ui edit \
  --override-base-url http://$ALB_DNS:9000

hal config security api edit \
  --override-base-url http://$ALB_DNS:8084

hal config security api edit \
  --cors-access-allowed-origin http://$ALB_DNS:9000
```

---

### 2. Force 0.0.0.0 Binding (Critical Fix)

By default, Spinnaker binds to `localhost`. This must be overridden.

```bash
mkdir -p ~/.hal/default/service-settings/

echo "host: 0.0.0.0" | tee ~/.hal/default/service-settings/gate.yml
echo "host: 0.0.0.0" | tee ~/.hal/default/service-settings/deck.yml
echo "host: 0.0.0.0" | tee ~/.hal/default/service-settings/front50.yml
```

---

## 🌐 Phase 5: Apache Configuration (Deck UI)

Deck is a static UI and must be served via Apache on port **9000**.

---

### Configure Apache

```bash
# Change Apache port
sudo sed -i 's/Listen 80/Listen 9000/' /etc/apache2/ports.conf

# Create Spinnaker site config
sudo tee /etc/apache2/sites-available/spinnaker.conf <<EOF
<VirtualHost *:9000>
  DocumentRoot /opt/deck/html/
  <Directory /opt/deck/html/>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
  </Directory>
</VirtualHost>
EOF

sudo a2dissite 000-default.conf || true
sudo a2ensite spinnaker.conf
sudo systemctl restart apache2
```

Verify:

```bash
netstat -tuln | grep 9000
```

---

## 🏁 Phase 6: Final Deployment

```bash
sudo hal deploy apply

# Restart critical services
sudo systemctl restart redis-server
sudo systemctl restart front50 gate
```

---

## 🩺 Troubleshooting Guide

| Issue           | Verification Command                   | Solution                      |                     |
| --------------- | -------------------------------------- | ----------------------------- | ------------------- |
| UI not loading  | `netstat -tuln                         | grep 9000`                    | Check Apache status |
| Gate down       | `curl -I http://localhost:8084/health` | Check `journalctl -u gate`    |                     |
| CORS error      | `hal config security api get`          | Match browser URL exactly     |                     |
| Data not saving | `redis-cli keys "front50:*"`           | Re-check Redis storage config |                     |

---

## ✅ Final Notes

* This setup is **ideal for learning, PoCs, and demos**
* For production, consider **EKS + external Redis + OAuth**
* Always monitor memory usage (`htop`) on small EC2 instances

---

🎯 **Spinnaker is now fully operational behind an AWS ALB on EC2!**

