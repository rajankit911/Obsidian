---
tags:
  - database-internals
  - concurrency-control
  - mvcc
  - isolation-levels
  - transactions
date_created: 2026-08-24
last_modified: 2026-08-24
---

# 🕰️ MVCC – Multi-Version Concurrency Control

> [!info] **The Core Idea**
> Instead of locking a row so only one transaction can touch it at a time, MVCC keeps **multiple versions** of the same row alive simultaneously. Each transaction sees a **consistent snapshot** of the database as it existed at a specific point in time.
>
> **The golden rule of MVCC:**
> > 🔵 **Readers never block Writers. Writers never block Readers.**

---

## 🧱 How MVCC Stores Versions

Every row in an MVCC database carries hidden metadata columns tracking its lifetime:

| Hidden Column | Meaning |
|---------------|---------|
| `xmin` | Transaction ID (TxID) that **created** this version |
| `xmax` | TxID that **deleted or updated** this version (`0` = still alive) |
| `ctid` | Physical location of this row version (PostgreSQL) |

> [!example] **Visualising a version chain**
>
> ```
> Initial state — Alice's salary is 50,000
> ┌──────────────────────────────────────────┐
> │  xmin=10  xmax=0   salary=50000  (LIVE)  │  ← version V1
> └──────────────────────────────────────────┘
>
> T2 (TxID=20) runs: UPDATE salary = 60000
> ┌────────────────────────────────────────────┐
> │  xmin=10  xmax=20  salary=50000  (EXPIRED) │  ← version V1 (old)
> └────────────────────────────────────────────┘
> ┌──────────────────────────────────────────┐
> │  xmin=20  xmax=0   salary=60000  (LIVE)  │  ← version V2 (new)
> └──────────────────────────────────────────┘
> ```
>
> Old version V1 is **not deleted immediately** — it stays until no transaction needs it anymore (cleaned up by VACUUM in PostgreSQL).

```mermaid
flowchart LR
    V1["V1\nxmin=10 xmax=20\nsalary=50000\n❌ Expired"]
    V2["V2\nxmin=20 xmax=35\nsalary=60000\n❌ Expired"]
    V3["V3\nxmin=35 xmax=0\nsalary=75000\n✅ Live"]

    V1 -->|"UPDATE by T20"| V2
    V2 -->|"UPDATE by T35"| V3
```

### 👁️ Visibility Rule

> [!info] **A version V is visible to transaction T if:**
> 1. `V.xmin` is **committed** and `V.xmin < T.snapshot_time` *(version was created before my snapshot)*
> 2. `V.xmax` is **0** (never deleted) **OR** `V.xmax` is not yet committed at T's snapshot time *(version wasn't deleted before my snapshot)*

```mermaid
flowchart TB
    A{{"Is xmin committed\nbefore my snapshot?"}}
    A -- No --> X1["❌ Not visible\n(creator not committed yet)"]
    A -- Yes --> B{{"Is xmax = 0\nor xmax NOT committed\nbefore my snapshot?"}}
    B -- No --> X2["❌ Not visible\n(row was deleted before me)"]
    B -- Yes --> OK["✅ VISIBLE"]
```

---

## 🎯 How MVCC Works Under Each Isolation Level

The **key variable** that changes across isolation levels is:
> 🔑 **When is the snapshot taken?**

```mermaid
flowchart LR
    RC["Read Committed\n📸 Snapshot per STATEMENT"]
    RR["Repeatable Read\n📸 Snapshot per TRANSACTION"]
    S["Serializable\n📸 Snapshot per TRANSACTION\n+ conflict detection"]

    RC --> RR --> S
```

---

### 🟡 Read Uncommitted

> [!note]
> In pure MVCC databases (e.g., **PostgreSQL**), Read Uncommitted behaves **identically to Read Committed** — because MVCC *structurally cannot* serve uncommitted versions (there's no "dirty" pointer to follow). Every version you read has a committed `xmin`.
>
> MySQL InnoDB technically can serve dirty reads here, but this level is almost never used in practice.

| Anomaly | Status |
|---------|--------|
| Dirty Read | ✅ Prevented (by MVCC structure) |
| Non-Repeatable Read | ❌ Possible |
| Phantom Read | ❌ Possible |
| Write Skew | ❌ Possible |

---

### 🟠 Read Committed

> [!info] **Snapshot taken at the start of each STATEMENT.**
> Every new query inside the same transaction gets a **fresh snapshot**, so it can see data committed by other transactions between statements.

```mermaid
sequenceDiagram
    participant T1
    participant DB
    participant T2

    T1->>DB: BEGIN
    T2->>DB: BEGIN
    T1->>DB: SELECT salary FROM emp WHERE id=1
    DB-->>T1: 50000 (snapshot S1 taken here)

    T2->>DB: UPDATE emp SET salary=60000 WHERE id=1
    T2->>DB: COMMIT

    T1->>DB: SELECT salary FROM emp WHERE id=1
    Note over T1,DB: ⚠️ New snapshot S2 taken!
    DB-->>T1: 60000 ← sees T2's committed change
    T1->>DB: COMMIT
```

> [!warning] **Non-Repeatable Read** — T1 reads `50000` then `60000` in the same transaction. The same query yields different results.

> [!example] **When Read Committed is the right choice**
> ```sql
> -- Session 1 (report query that runs for 10 seconds)
> BEGIN;
> SELECT COUNT(*) FROM orders WHERE status = 'pending';  -- snapshot taken HERE
> -- ... 5 seconds later ...
> SELECT SUM(amount) FROM orders WHERE status = 'pending'; -- NEW snapshot
> -- Session 2 commits new orders between these two queries
> -- The two queries may be inconsistent — acceptable for dashboards, not for accounting
> COMMIT;
> ```

---

### 🟢 Repeatable Read

> [!info] **Snapshot taken once at the start of the TRANSACTION.**
> All reads throughout the transaction see the **same frozen snapshot**, regardless of what other transactions commit in the meantime.

```mermaid
sequenceDiagram
    participant T1
    participant DB
    participant T2

    T1->>DB: BEGIN
    DB-->>T1: 📸 Snapshot S taken (TxID=100)
    T2->>DB: BEGIN

    T1->>DB: SELECT salary FROM emp WHERE id=1
    DB-->>T1: 50000

    T2->>DB: UPDATE emp SET salary=60000 WHERE id=1
    T2->>DB: COMMIT

    T1->>DB: SELECT salary FROM emp WHERE id=1
    Note over T1,DB: Still using snapshot S (TxID=100)
    DB-->>T1: 50000 ✅ Same result — repeatable!
    T1->>DB: COMMIT
```

> [!note] **PostgreSQL goes further than the SQL standard here.**
> The SQL standard says Repeatable Read still allows Phantom Reads. But PostgreSQL's snapshot-based RR **also prevents phantom reads** — because the snapshot was frozen, newly inserted rows (with `xmin > snapshot`) are simply invisible.

| Anomaly | SQL Standard RR | PostgreSQL RR |
|---------|----------------|--------------|
| Dirty Read | ✅ Prevented | ✅ Prevented |
| Non-Repeatable Read | ✅ Prevented | ✅ Prevented |
| Phantom Read | ❌ Possible | ✅ Prevented |
| Write Skew | ❌ Possible | ❌ Possible |

#### ⚠️ Write Skew — The Remaining Anomaly at RR

> [!warning] **Write Skew** is a subtle anomaly that MVCC Repeatable Read cannot prevent.
>
> ```
> Invariant: At least one doctor must be on call at all times.
> Current state: Alice (on_call=true), Bob (on_call=true)
>
> T1 (Alice going off call):           T2 (Bob going off call):
>   SELECT COUNT(*) WHERE on_call=true   SELECT COUNT(*) WHERE on_call=true
>   → sees 2 ✅ safe to proceed          → sees 2 ✅ safe to proceed
>   UPDATE alice SET on_call=false       UPDATE bob SET on_call=false
>   COMMIT                               COMMIT
>
> Result: BOTH doctors off call — invariant violated! 💥
> ```
>
> Each transaction read a consistent snapshot and made a locally-valid decision. But the **combined effect** broke the invariant. Neither saw the other's write.

```mermaid
flowchart LR
    subgraph T1["T1 – Alice"]
        A1["READ: 2 doctors on call\n✅ OK to go off call"]
        A2["WRITE: alice.on_call = false"]
        A1 --> A2
    end

    subgraph T2["T2 – Bob"]
        B1["READ: 2 doctors on call\n✅ OK to go off call"]
        B2["WRITE: bob.on_call = false"]
        B1 --> B2
    end

    T1 & T2 -->|"Concurrent — neither sees the other's write"| Result["💥 0 doctors on call\nInvariant violated"]
```

---

### 🔴 Serializable (SSI — Serializable Snapshot Isolation)

> [!info] **Snapshot taken at transaction start + runtime conflict detection.**
> PostgreSQL implements **SSI (Serializable Snapshot Isolation)** — a non-blocking algorithm that detects dangerous "rw-anti-dependency" cycles between transactions and aborts one if a serialization conflict is found.
>
> MySQL InnoDB uses **locking** (not SSI) at Serializable — it adds gap locks and next-key locks, effectively blocking concurrent writes to prevent phantoms.

```mermaid
flowchart TB
    subgraph SSI["PostgreSQL SSI — Detecting Conflicts"]
        T1S["T1 reads X\n(depends on X's committed value)"]
        T2S["T2 writes X\n(creates rw-anti-dependency on T1)"]
        T2R["T2 reads Y"]
        T1W["T1 writes Y\n(creates rw-anti-dependency on T2)"]

        T1S -->|"rw-dep"| T2S
        T2R -->|"rw-dep"| T1W

        Cycle{{"Dangerous cycle\ndetected!"}}
        T2S --> Cycle
        T1W --> Cycle
        Cycle -->|"Abort one txn"| Abort["❌ Serialization failure\nRetry transaction"]
    end
```

> [!example] **The doctor on-call scenario — now fixed at Serializable**
> ```sql
> BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
>
> -- T1: Alice checking doctors on call
> SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- reads: 2
> UPDATE doctors SET on_call = false WHERE name = 'Alice';
>
> COMMIT;
> -- If T2 ran concurrently and created a conflict cycle,
> -- PostgreSQL detects it and aborts one with:
> -- ERROR: could not serialize access due to concurrent update
> ```

| Anomaly | Serializable (SSI) |
|---------|-------------------|
| Dirty Read | ✅ Prevented |
| Non-Repeatable Read | ✅ Prevented |
| Phantom Read | ✅ Prevented |
| Write Skew | ✅ Prevented |

---

## 🧹 MVCC Housekeeping — Version Cleanup

Keeping old versions has a cost. Databases must clean them up when no active transaction needs them anymore.

```mermaid
flowchart LR
    OldV["Old row version\nxmin=10, xmax=20\n(dead tuple)"]
    ActiveTx{{"Any active transaction\nwith snapshot ≤ 20?"}}
    Vacuum["🧹 VACUUM marks it\nfor reuse"]
    Keep["⏳ Keep alive\n(still needed)"]

    OldV --> ActiveTx
    ActiveTx -- No --> Vacuum
    ActiveTx -- Yes --> Keep
```

| Database | Cleanup Mechanism | Notes |
|----------|------------------|-------|
| **PostgreSQL** | `VACUUM` / `autovacuum` | Dead tuples reclaimed; table bloat if vacuum lags |
| **MySQL InnoDB** | Purge thread | Undo log segments purged in background |
| **Oracle** | Undo tablespace | Circular reuse; old reads fail with `ORA-01555` if undo is overwritten |

> [!warning] **Long-running transactions are MVCC's enemy.**
> A transaction that stays open for hours forces the DB to keep **all versions** created during that window alive — causing table bloat, undo log growth, and performance degradation.

---

## 📊 TL;DR — MVCC Across Isolation Levels

| Isolation Level | Snapshot Taken | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
|----------------|----------------|-----------|-------------------|-------------|------------|
| Read Uncommitted | Per statement (PG: same as RC) | ✅ | ❌ | ❌ | ❌ |
| Read Committed | Per **statement** | ✅ | ❌ | ❌ | ❌ |
| Repeatable Read | Per **transaction** | ✅ | ✅ | ✅ (PG) / ❌ (standard) | ❌ |
| Serializable | Per transaction + SSI | ✅ | ✅ | ✅ | ✅ |

> [!tip] **Quick rule of thumb**
> - Default for most apps → **Read Committed** (PostgreSQL default)
> - Financial / analytical queries needing consistent view → **Repeatable Read**
> - Critical invariants across multiple rows → **Serializable**

---

## 🔗 Related Topics

- [[Optimistic and Pessimistic Locks]]
- [[Lock-Based Concurrency Control]]
- [[Transaction Isolation Levels]]
- [[Deadlocks and Prevention]]
- [[Write-Ahead Logging (WAL)]]
