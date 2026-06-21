---
tags:
  - redis
  - database-internals
  - production-setup
  - devops
  - architecture
date_created: 2026-06-09
last_modified: 2026-06-09
---

# Redis Sentinel Deployment & Cost Optimization

A high-availability Redis Sentinel setup requires at least **3 Sentinel instances** to form a quorum and prevent split-brain scenarios. 

Running 3 dedicated VMs solely for Sentinel processes is expensive and unnecessary, as the Sentinel process is extremely lightweight (consuming ~10-15MB RAM and negligible CPU). You can co-locate Sentinel processes using the following patterns:

---

## Pattern A: Co-location with Redis Database Nodes (Minimum 3 VMs)

If your database layer already runs **3 database nodes** (e.g., 1 Master + 2 Slaves), you can run one Sentinel process on each database VM.

### Topology Layout
- **VM 1**: Redis Master + Sentinel 1 (IP `10.0.0.10`)
- **VM 2**: Redis Slave 1 + Sentinel 2 (IP `10.0.0.11`)
- **VM 3**: Redis Slave 2 + Sentinel 3 (IP `10.0.0.12`)

### Cost Impact
- **Total VMs**: **3 VMs**
- **Pros**: Zero extra infrastructure cost; clean 1-to-1 mapping between Redis instances and Sentinel processes.
- **Cons**: Requires at least 2 replicas to reach the 3-VM minimum.

---

## Pattern B: Co-location with Application Servers (Minimum 2 DB VMs)

If you have a **2-node** Redis setup (1 Master + 1 Slave) due to budget constraints, running Sentinel on only those 2 database servers is unsafe. If a network partition occurs between them, neither node can achieve the majority quorum of 2 to elect a new Master. 

To solve this, run the Sentinel processes on your **Application/Web server instances**:

### Topology Layout
```
   ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
   │   App Server 1    │    │   App Server 2    │    │   App Server 3    │
   │  (App + Sentinel1)│    │  (App + Sentinel2)│    │  (App + Sentinel3)│
   └─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
             │                        │                        │
             └────────────────────────┼────────────────────────┘
                                      │ (Monitoring & Control)
                                      ▼
                        ┌───────────────────────────┐
                        │   Database Tier (2 VMs)   │
                        │ ┌──────────────┐          │
                        │ │ Redis Master │ (10.0.0.8)│
                        │ └──────┬───────┘          │
                        │        │ (Replication)    │
                        │        ▼                  │
                        │ ┌──────────────┐          │
                        │ │ Redis Slave  │ (10.0.0.9)│
                        │ └──────────────┘          │
                        └───────────────────────────┘
```

- **VM 1 (DB Tier)**: Redis Master (IP `10.0.0.8`)
- **VM 2 (DB Tier)**: Redis Slave (IP `10.0.0.9`)
- **VM 3 (App Tier)**: Application Client + Sentinel 1 (IP `10.0.0.21`)
- **VM 4 (App Tier)**: Application Client + Sentinel 2 (IP `10.0.0.22`)
- **VM 5 (App Tier)**: Application Client + Sentinel 3 (IP `10.0.0.23`)

### Cost Impact
- **Total Database VMs**: **2 VMs**
- **Pros**: Saves the cost of 1 database VM; ensures high availability even with just 1 Master and 1 Slave.
- **Cons**: Sentinel configuration and processes are coupled with the application server lifecycle.

---

## ⚙️ Basic Sentinel Configuration (`sentinel.conf`)

This configuration should be deployed on all 3 hosts running Sentinel.

```ini
# Port on which Sentinel runs (default is 26379)
port 26379

# Run Sentinel in non-daemon mode or daemon mode depending on service manager
daemonize yes

# Monitor a master named "mymaster" at 10.0.0.10:6379 with a quorum of 2
# Format: sentinel monitor <master-name> <ip> <port> <quorum>
sentinel monitor mymaster 10.0.0.10 6379 2

# Authenticate with the master if it requires a password
sentinel auth-pass mymaster StrongMasterPassword123!

# Time in milliseconds that the master can be unresponsive before failover is triggered
sentinel down-after-milliseconds mymaster 30000

# Timeout in milliseconds for failover operations
sentinel failover-timeout mymaster 180000

# Number of replicas that can be reconfigured to sync with the new master simultaneously
sentinel parallel-syncs mymaster 1
```

---
_End of Sentinel Deployment notes._
