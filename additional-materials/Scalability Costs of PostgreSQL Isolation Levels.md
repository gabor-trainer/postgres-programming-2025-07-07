# Scalability Costs of PostgreSQL Isolation Levels
## 1. **Read Committed vs. Repeatable Read**

### **How They Differ Internally**

* **Read Committed**: Each query uses a **new snapshot**, so reads always see the latest committed data. Minimal tracking required.
* **Repeatable Read**: A **single snapshot is fixed at transaction start**, and all reads see only that consistent view. This requires tracking of the snapshot throughout the transaction.

### **Scalability Cost**

| Dimension                   | Cost Going from Read Committed to Repeatable Read                                                                                       |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **Snapshot Duration**       | Long-lived snapshots can cause **bloat**: vacuum can't clean old row versions still visible to the snapshot.                            |
| **Memory Use**              | Long transactions hold more catalog and visibility data in memory due to snapshot retention.                                            |
| **Tuple Visibility Checks** | More tuples are scanned because older snapshots may see tuples invisible to newer snapshots.                                            |
| **Writer Contention**       | Writers may need to maintain more tuple versions for longer periods (due to xmin visibility), increasing heap size and vacuum pressure. |
| **Throughput Impact**       | For workloads with many concurrent writes and long transactions, throughput can degrade due to tuple bloat and cache pressure.          |
| **Vacuum Lag**              | Vacuum cannot reclaim space if any long-running Repeatable Read transaction could still see those rows.                                 |

**Summary**: The cost is **moderate** and mostly comes from **longer-lived snapshots**, which delay vacuum and increase memory and CPU usage for tuple visibility checks.

---

## 2. **Repeatable Read vs. Serializable**

### **How They Differ Internally**

* Both use one snapshot per transaction.
* **Serializable** adds **Serializable Snapshot Isolation (SSI)**:

  * Tracks **read/write dependencies**.
  * Uses **predicate locks** (SIReadLock) for **non-blocking conflict tracking**.
  * Can **abort transactions at commit** time if a conflict cycle is detected.

### **Scalability Cost**

| Dimension                 | Cost Going from Repeatable Read to Serializable                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Transaction Overhead**  | SSI must track every read and write (via predicate locks). This adds CPU and memory overhead per transaction.                    |
| **Memory Usage**          | Each transaction may need to maintain a list of read and write sets. High concurrency → large tracking metadata.                 |
| **Abort Rates**           | When many concurrent transactions interact, **false positives** in SSI’s conflict detection can cause **increased rollbacks**.   |
| **Concurrency Impact**    | Increased transaction contention and aborts in high-throughput OLTP systems. Fewer transactions may complete successfully.       |
| **Index Usage**           | Reads on non-indexed columns create wider predicate locks (e.g., page or relation locks), increasing false conflict probability. |
| **Lock Manager Pressure** | SIReadLocks are tracked in `pg_locks` but do not block. However, their volume grows with transaction complexity.                 |
| **CPU Load**              | More CPU is used for tracking dependency graphs and checking for conflict cycles.                                                |

**Summary**: The cost is **high**, especially in **write-heavy** or **high-concurrency** environments. The overhead from tracking and aborting transactions to ensure serializability can degrade performance if not carefully managed.

---

## Final Comparison Table

| Isolation Level     | Read Method       | Write Behavior                | Snapshot Scope  | Conflict Detection | Abort Risk | Scalability Cost |
| ------------------- | ----------------- | ----------------------------- | --------------- | ------------------ | ---------- | ---------------- |
| **Read Committed**  | Latest committed  | No anomalies prevented        | Per-statement   | None               | None       | **Lowest**       |
| **Repeatable Read** | Fixed at tx start | Prevents non-repeatable reads | Per-transaction | None               | None       | **Moderate**     |
| **Serializable**    | Fixed at tx start | Prevents all anomalies        | Per-transaction | SSI (background)   | Possible   | **High**         |

---

## Practical Recommendations

* **Read Committed** is suitable for high-throughput, low-conflict OLTP workloads (e.g. logging, metrics ingestion).
* **Repeatable Read** balances consistency and performance for most applications that need consistent reads within a transaction. Always validate the need for long-lived transactions.
* **Serializable** should only be used when **true serializability is required**, such as in **financial systems** or **complex business rules**—but and always validate the need. Expect significantly reduced scalability in high concurrency.

Let me know if you'd like benchmark data or tuning strategies to mitigate these costs.
