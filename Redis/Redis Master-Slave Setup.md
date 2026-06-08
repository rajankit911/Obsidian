---
tags:
  - redis
  - database-internals
  - production-setup
  - devops
date_created: 2026-06-09
last_modified: 2026-06-09
---

# Redis Master-Slave Production Setup Guide

This guide walks through configuring a secure, production-grade Redis Master-Slave (Replication) setup using two separate servers.

---

## 📋 Topology Overview
- **Master Node**: IP `10.0.0.10` (Port `6379`) — Handles Writes & Reads
- **Slave Node**: IP `10.0.0.11` (Port `6379`) — Handles Reads Only (Replicates from Master)

---

## 1️⃣ Operating System Tuning (On Both Servers)
Before configuring Redis, optimize the host Linux kernel settings to prevent memory and connection bottlenecking.

### A. Overcommit Memory
Redis uses a background process (`fork()`) to save snapshots (RDB files) or rewrite AOF files. Set `overcommit_memory` to `1` so the OS doesn't deny memory allocation requests when memory runs low.
```bash
# Apply immediately
sudo sysctl vm.overcommit_memory=1

# Make permanent in /etc/sysctl.conf
echo 'vm.overcommit_memory = 1' | sudo tee -a /etc/sysctl.conf
```

### B. Disable Transparent Huge Pages (THP)
THP can cause significant latency spikes and high memory usage during background writes.
```bash
# Disable immediately
sudo echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Configure to disable at boot in /etc/rc.local or via systemd service
```

### C. TCP Backlog (somaxconn)
Raise the limit of queued connections to prevent connection drops during traffic spikes.
```bash
# Apply immediately
sudo sysctl -w net.core.somaxconn=1024

# Make permanent in /etc/sysctl.conf
echo 'net.core.somaxconn = 1024' | sudo tee -a /etc/sysctl.conf
```

---

## 2️⃣ Master Node Configuration (`redis.conf` on Master)

Locate the `redis.conf` file (typically in `/etc/redis/redis.conf`). Make the following adjustments:

```ini
# --- NETWORK ---
# Bind to localhost and the private network interface
bind 127.0.0.1 10.0.0.10
port 6379
protected-mode yes

# --- SECURITY ---
# Require clients to authenticate
requirepass StrongMasterPassword123!
# Auth password for the master in case it replicates from a promoted slave later
masterauth StrongMasterPassword123!

# --- PERSISTENCE ---
# Use AOF (Append Only File) for production resilience
appendonly yes
appendfsync everysec

# --- RESOURCE LIMITS ---
# Set maximum memory limit (adjust based on system RAM, e.g., leaving 20-30% for OS/Forking)
maxmemory 8gb
maxmemory-policy volatile-lru
```

---

## 3️⃣ Slave Node Configuration (`redis.conf` on Slave)

Locate the `redis.conf` file on the slave server (`10.0.0.11`) and adjust the settings:

```ini
# --- NETWORK ---
bind 127.0.0.1 10.0.0.11
port 6379
protected-mode yes

# --- REPLICATION RELATIONSHIP ---
# Define the master node to replicate from
replicaof 10.0.0.10 6379

# --- SECURITY & AUTHENTICATION ---
# Password needed to authenticate against the master node
masterauth StrongMasterPassword123!
# Local password for clients accessing this slave node (usually identical to master)
requirepass StrongMasterPassword123!

# --- REPLICA READ-ONLY ---
# Force replicas to be read-only to prevent drift
replica-read-only yes

# --- PERSISTENCE ---
# Replicas can also run AOF to offload IO workload from the Master
appendonly yes
appendfsync everysec

# --- RESOURCE LIMITS ---
maxmemory 8gb
maxmemory-policy volatile-lru
```

---

## 4️⃣ Service Control

Start or restart Redis on both servers to apply the configuration:

```bash
# Restart the Master
sudo systemctl restart redis-server

# Restart the Slave
sudo systemctl restart redis-server
```

---

## 5️⃣ Verification

Validate the replication status using `redis-cli` from the terminal.

### A. Check Master Status
Connect to the Master node:
```bash
redis-cli -h 10.0.0.10 -a StrongMasterPassword123! info replication
```
**Expected Output:**
```ini
# Replication
role:master
connected_slaves:1
slave0:ip=10.0.0.11,port=6379,state=online,offset=20394,lag=0
...
```

### B. Check Slave Status
Connect to the Slave node:
```bash
redis-cli -h 10.0.0.11 -a StrongMasterPassword123! info replication
```
**Expected Output:**
```ini
# Replication
role:slave
master_host:10.0.0.10
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
...
```

### C. Write & Read Test
1. **Write to Master**:
   ```bash
   redis-cli -h 10.0.0.10 -a StrongMasterPassword123! set testkey "Hello Replication"
   ```
2. **Read from Slave**:
   ```bash
   redis-cli -h 10.0.0.11 -a StrongMasterPassword123! get testkey
   # Output should be: "Hello Replication"
   ```
3. **Write Test on Slave (Should fail)**:
   ```bash
   redis-cli -h 10.0.0.11 -a StrongMasterPassword123! set testkey "Fail"
   # Output should be: (error) READONLY You can't write against a read only replica.
   ```

---

## 🛡️ Production Best Practices & Security

> [!WARNING]
> **Never run Redis bound to public interfaces without a firewall.** Use Linux `iptables` or cloud security groups to allow traffic on port `6379` strictly between your application servers and the Redis servers.

### 1. Rename Dangerous Commands
Disable or rename commands that can crash your Redis cluster or leak data:
```ini
# In redis.conf on both Master and Slave:
rename-command FLUSHALL ""
rename-command FLUSHDB  ""
rename-command CONFIG   "SECURE_CONFIG_34891"
rename-command KEYS     ""
```

### 2. Client Eviction Tuning
If the memory threshold is reached, make sure your eviction policy (`maxmemory-policy`) matches your workload:
- `volatile-lru`: Evicts the least recently used keys with an expire set (recommended for caches).
- `noeviction`: Returns errors when memory is full (recommended for persistent DB use).

---
_End of setup guide._
