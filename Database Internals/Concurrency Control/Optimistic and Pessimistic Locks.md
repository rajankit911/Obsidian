---
tags:
  - database-internals
  - concurrency-control
  - locking
  - transactions
date_created: 2026-08-23
last_modified: 2026-08-23
---

# 🔐 Optimistic vs Pessimistic Locking

> [!info] **Overview**
> Two fundamental strategies for handling **concurrent access** to shared data in databases and distributed systems. They differ in their **assumptions about conflict frequency** and when they enforce isolation.

---

## 🧠 Definitions

### 🔴 Pessimistic Locking

> [!info] **Definition**
> Assumes that **conflicts are likely**. Locks the resource **before** reading or writing it, preventing other transactions from accessing it until the lock is released.

- Based on the "expect the worst" philosophy.
- Uses actual **database-level locks** (row locks, table locks, etc.).
- The transaction holds the lock for its **entire duration**.

```
T1 acquires lock → T1 reads/writes → T1 commits → lock released
                    ↑
              T2 is BLOCKED here until T1 releases
```

### 🟢 Optimistic Locking

> [!info] **Definition**
> Assumes that **conflicts are rare**. Does **not** lock the resource upfront. Instead, it proceeds freely and **validates at commit time** whether a conflict occurred. If a conflict is found, the transaction is **rolled back and retried**.

- Based on the "expect the best" philosophy.
- Typically uses a **version number** or **timestamp** column to detect conflicts.
- No actual lock is held during the read phase.

```
T1 reads (version=5) → T1 modifies locally → T1 commits
                                                  ↓
                                        Check: is version still 5?
                                        ✅ Yes → Commit, set version=6
                                        ❌ No  → Rollback & retry
```

---

## ⚙️ How to Achieve It

### 🔴 Pessimistic Locking

> [!example] **SQL: Exclusive Lock with `SELECT FOR UPDATE`**
> ```sql
> BEGIN;
>
> -- Acquires an exclusive row-level lock
> SELECT * FROM accounts WHERE id = 42 FOR UPDATE;
>
> -- No other transaction can read/write this row until COMMIT
> UPDATE accounts SET balance = balance - 100 WHERE id = 42;
>
> COMMIT;
> ```

> [!example] **Java: Pessimistic Lock with JPA**
> ```java
> @Transactional
> public void deductBalance(Long accountId, BigDecimal amount) {
>     Account account = entityManager.find(
>         Account.class,
>         accountId,
>         LockModeType.PESSIMISTIC_WRITE  // acquires SELECT FOR UPDATE
>     );
>     account.setBalance(account.getBalance().subtract(amount));
>     // lock released on transaction commit
> }
> ```

**Common Lock Modes:**

| Mode | SQL Equivalent | Behaviour |
|------|---------------|-----------|
| `PESSIMISTIC_READ` | `SELECT FOR SHARE` | Prevents writes; allows concurrent reads |
| `PESSIMISTIC_WRITE` | `SELECT FOR UPDATE` | Prevents all concurrent reads & writes |

---

### 🟢 Optimistic Locking

> [!example] **Schema: Add a `version` column**
> ```sql
> ALTER TABLE accounts ADD COLUMN version INT NOT NULL DEFAULT 0;
> ```

> [!example] **SQL: Manual optimistic check**
> ```sql
> -- Step 1: Read with version
> SELECT id, balance, version FROM accounts WHERE id = 42;
> -- Returns: balance=500, version=5
>
> -- Step 2: Update only if version hasn't changed
> UPDATE accounts
>    SET balance = 400, version = version + 1
>  WHERE id = 42 AND version = 5;
>
> -- If affected rows = 0 → conflict detected → retry
> ```

> [!example] **Java: Optimistic Lock with JPA `@Version`**
> ```java
> @Entity
> public class Account {
>     @Id
>     private Long id;
>
>     private BigDecimal balance;
>
>     @Version  // JPA manages version automatically
>     private int version;
> }
>
> @Transactional
> public void deductBalance(Long accountId, BigDecimal amount) {
>     Account account = accountRepository.findById(accountId).orElseThrow();
>     account.setBalance(account.getBalance().subtract(amount));
>     // JPA will throw OptimisticLockException on version mismatch at commit
> }
> ```

> [!note] **Retry on conflict**
> ```java
> @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
> @Transactional
> public void deductBalanceWithRetry(Long accountId, BigDecimal amount) {
>     // ... same logic as above
> }
> ```

**Conflict Detection Strategies:**

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Version number** | Integer incremented on each write | Simple, widely supported |
| **Timestamp** | `last_modified` timestamp compared | When audit trail matters |
| **Hash / Checksum** | Hash of the row content compared | When no schema change is possible |

---

## 🧭 When to Use Them

### ✅ Use **Pessimistic Locking** when…

> [!note]
> - **Conflicts are frequent** — e.g., a hot row like a bank account balance being updated by many concurrent users.
> - **Operations are short and fast** — the lock hold time is minimal, so blocking is acceptable.
> - **Cost of retry is high** — rolling back and redoing expensive work (e.g., complex computations or external calls) is unacceptable.
> - **Data integrity is non-negotiable and immediate** — financial transactions, inventory decrement (avoid overselling).
> - **Working with external systems** — if the transaction involves a non-retryable side effect (e.g., sending an email), failing late is worse than blocking early.

```
Good fit: flight seat reservation, bank transfers, flash sale inventory
```

---

### ✅ Use **Optimistic Locking** when…

> [!note]
> - **Conflicts are rare** — most transactions work on different rows or at non-overlapping times.
> - **Reads vastly outnumber writes** — e.g., a CMS where many users read articles but few edit simultaneously.
> - **Scalability matters** — no lock contention means higher throughput and no deadlocks.
> - **Long-running transactions** — holding a pessimistic lock for seconds would be too disruptive; optimistic lock lets you read freely.
> - **Distributed / microservices systems** — where cross-service locks are impractical or impossible.

```
Good fit: collaborative document editing (with retry), user profile updates,
          shopping cart modifications, REST APIs with ETags
```

---

## 🚫 When NOT to Use Them

### ❌ Avoid **Pessimistic Locking** when…

> [!warning]
> - **Conflicts are rare** — you're paying the performance cost of locking for no real benefit.
> - **Transactions are long-running** — holding locks for a long time causes thread starvation and cascading slowdowns.
> - **High read-to-write ratio** — blocking readers with `FOR UPDATE` where a shared read lock (or no lock) would suffice.
> - **Distributed systems** — distributed locks are complex, have failure modes, and can become a single point of failure.
> - **Risk of Deadlocks** — if multiple transactions lock resources in different orders, deadlocks will occur.

---

### ❌ Avoid **Optimistic Locking** when…

> [!warning]
> - **Conflicts are frequent** — high retry rates waste CPU, increase latency, and degrade UX under load.
> - **Operations have side effects** — if a transaction sends an email or charges a card mid-way, rolling back is painful or impossible.
> - **Retries are expensive** — e.g., recalculating a complex report on every conflict is wasteful.
> - **Starvation risk** — under heavy contention, a transaction may retry indefinitely and never succeed ("livelock").

---

## 📊 TL;DR – What to Use When?

| Factor | Pessimistic | Optimistic |
|--------|-------------|------------|
| Conflict frequency | High | Low |
| Transaction duration | Short | Can be long |
| Read/Write ratio | Write-heavy | Read-heavy |
| Side effects | OK (block early) | Risky (fail late) |
| Scalability | Lower (lock contention) | Higher (no locks) |
| Deadlock risk | ⚠️ Yes | ✅ None |
| Retry overhead | ✅ None | ⚠️ Yes |
| Distributed systems | ❌ Complex | ✅ Natural fit |
| Use case examples | Bank transfers, inventory | Profile edits, CMS, REST APIs |

---

## 🔗 Related Topics

- [[Two-Phase Locking (2PL)]]
- [[MVCC – Multi-Version Concurrency Control]]
- [[Transaction Isolation Levels]]
- [[Deadlocks and Prevention]]
