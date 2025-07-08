### Understanding and Applying Transaction Isolation Levels

**Objective:**
This lab provides a deep dive into PostgreSQL's **Transaction Isolation Levels**. You will understand what isolation levels are, explore common concurrency phenomena (Non-Repeatable Reads, Phantom Reads), and learn how to explicitly set and test different isolation levels (`READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`). A key focus will be on demonstrating a scenario where `REPEATABLE READ` is essential to prevent data inconsistencies that would occur at lower isolation levels, as well as showcasing PostgreSQL's specific implementation nuances regarding Phantom Reads.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
    *   *(If not installed, please refer to the installation instructions from the previous interaction or a reliable source for Northwind PostgreSQL setup.)*
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line, DBeaver, VS Code with PostgreSQL extension). You will need to open **two separate query sessions/windows** connected to the same `northwind` database to simulate concurrent transactions.
4.  **Basic SQL Knowledge:** Familiarity with `SELECT`, `UPDATE`, `BEGIN`, `COMMIT`, `ROLLBACK`.

**Environment Setup:**

1.  **Open Two SQL Client Sessions:** Connect both sessions to your `northwind` database. Label them clearly (e.g., "Session 1 - Manager Report" and "Session 2 - Inventory Update").
2.  **Verify Connection:** In both sessions, run `SELECT count(*) FROM customers;` to confirm connectivity.

---

### **8.1 Understanding Transaction Isolation Levels**

**What are Isolation Levels?**
In a multi-user database environment like Northwind, many transactions (sequences of database operations) can run concurrently. **Isolation** is one of the ACID properties of database transactions, ensuring that concurrent transactions execute independently without interfering with each other. An isolation level defines how much one transaction must be isolated from the actions of other concurrent transactions. Higher isolation levels mean more protection against concurrency anomalies but generally come with a performance cost (e.g., more locking, higher chance of transaction aborts).

**Concurrency Phenomena (Anomalies) to Prevent:**

1.  **Dirty Reads (Read Uncommitted):** A transaction reads data written by another concurrent transaction that has *not yet been committed*. If the uncommitted transaction later rolls back, the first transaction has read "dirty" or invalid data.
    *   **PostgreSQL Note:** PostgreSQL's `READ UNCOMMITTED` isolation level *behaves identically to `READ COMMITTED`* due to its Multi-Version Concurrency Control (MVCC) architecture. Therefore, dirty reads are **never** possible in PostgreSQL.

2.  **Non-Repeatable Reads (Read Committed):** A transaction reads a specific row twice, and between the two reads, another concurrent transaction *commits an update or deletion to that same row*. The second read retrieves different data (or no data) for the same row.
    *   **PostgreSQL Note:** This anomaly *is* possible at the default `READ COMMITTED` level.

3.  **Phantom Reads (Repeatable Read):** A transaction executes a query (e.g., `SELECT ... WHERE condition`) that retrieves a set of rows. Later, within the *same transaction*, it re-executes the *same query*, and the set of rows changes because another concurrent transaction *commits an insert or delete of rows that satisfy the `WHERE` clause*.
    *   **PostgreSQL Note:** This is where it gets subtle. In PostgreSQL's MVCC implementation:
        *   `REPEATABLE READ` *does* prevent phantom reads caused by *updates or deletions* of existing rows (these become non-repeatable reads).
        *   However, `REPEATABLE READ` *still allows* phantom reads caused by *newly inserted rows* matching the criteria. This is because `REPEATABLE READ` guarantees that data *already read* (or data that *could have been read* if you scanned the whole table) won't change from *other committed transactions*, but it doesn't prevent new rows from appearing if you re-scan (unless the new rows were inserted after your transaction started).
        *   To fully prevent all types of phantom reads (including inserts) and other complex concurrency anomalies, `SERIALIZABLE` is required.

**PostgreSQL's Standard Isolation Levels:**

*   **`READ COMMITTED` (Default):**
    *   Prevents: Dirty Reads.
    *   Allows: Non-Repeatable Reads, Phantom Reads.
    *   Behavior: A statement sees only rows committed before the statement began. Each new statement within the transaction sees the latest committed changes from other transactions.

*   **`REPEATABLE READ`:**
    *   Prevents: Dirty Reads, Non-Repeatable Reads.
    *   Allows: Phantom Reads (specifically, new inserts matching criteria in subsequent reads within the same transaction).
    *   Behavior: A transaction sees only rows committed before the transaction began. All subsequent `SELECT` statements within this transaction see the *exact same snapshot of data* as the first `SELECT` statement in that transaction. Concurrent `UPDATE`/`DELETE` operations that would affect rows read by a `REPEATABLE READ` transaction are blocked until the `REPEATABLE READ` transaction completes.

*   **`SERIALIZABLE`:**
    *   Prevents: Dirty Reads, Non-Repeatable Reads, Phantom Reads (all types), and other serialization anomalies (e.g., Write Skew).
    *   Behavior: Guarantees that concurrent transactions will produce the same result as if they were executed one after another (serially). This is achieved by detecting and aborting transactions that could lead to serialization anomalies. If such an anomaly is detected, one of the transactions will fail with a serialization_failure error (SQLSTATE `40001`), requiring the client to retry.

---

### **Lab Exercises:**

**Introduction to Tables Used:**
*   `products`: Main table for inventory and pricing.
*   `categories`: To filter products by category.

---

#### **Task 1: Demonstrate Non-Repeatable Read with `READ COMMITTED` (Default)**

**Description:**
Imagine a manager is trying to assess the potential revenue for 'Beverages' products. They first calculate the total stock, then later, the average unit price for these products. Another concurrent transaction updates a product's price in between their reads. At `READ COMMITTED` isolation, the manager's second read will show the *new*, committed price, leading to an inconsistent view of the data within their single transaction.

**Application Context:** Financial reporting, inventory assessment, or any complex calculation that relies on multiple reads of potentially changing data within a single logical operation. If the data changes mid-transaction, the final calculation could be flawed.

**Instructions:**
1.  **Session 1:** Run the initial setup and the first `SELECT`.
2.  **Session 2:** Run the `UPDATE` and `COMMIT`.
3.  **Session 1:** Run the second `SELECT`.

**SQL Query:**

**Session 1: Manager Report (READ COMMITTED - Default)**

```sql
-- Clean up any previous test state for product 1 (Chai) and product 2 (Chang)
UPDATE products SET unit_price = 18.00 WHERE product_id = 1;
UPDATE products SET units_in_stock = 17 WHERE product_id = 2;

-- Start Transaction 1
BEGIN;

-- Initial read: Get total units in stock for products in 'Beverages' (category_id = 1)
-- and their individual unit prices.
SELECT
    p.product_id,
    p.product_name,
    p.units_in_stock,
    p.unit_price
FROM
    products p
WHERE
    p.category_id = 1
ORDER BY p.product_id;

-- *** PAUSE HERE ***
-- Now switch to Session 2 and run its commands.
-- After Session 2 commits, switch back to Session 1.

-- Second read: After a concurrent update, re-read the data.
-- Specifically, check the average unit price for beverages.
SELECT
    AVG(p.unit_price) AS average_beverage_price
FROM
    products p
WHERE
    p.category_id = 1;

-- Compare the original unit_price for Product ID 1 (Chai) and Product ID 2 (Chang)
-- from your first select with the actual values in the database after the update.
SELECT
    p.product_id,
    p.product_name,
    p.unit_price
FROM
    products p
WHERE
    p.product_id IN (1, 2);

-- End Transaction 1
COMMIT;
```

**Session 2: Inventory Update**

```sql
-- Start Transaction 2
BEGIN;

-- Update the unit_price of 'Chai' (product_id 1) and 'Chang' (product_id 2) in 'Beverages'
UPDATE products SET unit_price = 20.00 WHERE product_id = 1;
UPDATE products SET units_in_stock = 20 WHERE product_id = 2; -- Change stock too

-- Commit Transaction 2
COMMIT;
```

**Expected Output/Explanation:**
*   **Session 1 - First `SELECT`:** Shows original prices for Chai (18.00) and Chang (19.00).
*   **Session 2 - `UPDATE` & `COMMIT`:** Changes prices.
*   **Session 1 - Second `SELECT` (AVG):** This will show the **new average price** based on the updated values (20.00 for Chai, 19.00 for Chang + others). This is a **Non-Repeatable Read** because the `AVG` calculation used a different price for Chai (and potentially stock for Chang) than what was observed in the first `SELECT` within the *same transaction*. The second explicit `SELECT p.product_id, p.product_name, p.unit_price` in Session 1 will clearly show the updated prices for products 1 and 2.

**Learning Point:**
This task demonstrates the **Non-Repeatable Read** anomaly. At `READ COMMITTED` (PostgreSQL's default), each `SELECT` statement within a transaction reads the latest *committed* version of data. If another transaction commits changes to rows already read by the current transaction, subsequent reads of those same rows will reflect the new committed values, leading to an inconsistent view within the transaction. This is problematic for reports or processes that rely on a stable view of data throughout their execution.

---

#### **Task 2: Solve Non-Repeatable Read with `REPEATABLE READ`**

**Description:**
We will repeat the scenario from Task 1, but this time, Session 1 will operate at `REPEATABLE READ` isolation. This level guarantees that once a row is read, any subsequent reads of that same row within the same transaction will always yield the same data, regardless of concurrent committed updates.

**Instructions:**
1.  **Session 1:** Set isolation level, then run initial `BEGIN` and `SELECT`.
2.  **Session 2:** Run `UPDATE` and `COMMIT`.
3.  **Session 1:** Run second `SELECT` and `COMMIT`. Observe the differences.

**SQL Query:**

**Session 1: Manager Report (REPEATABLE READ)**

```sql
-- Clean up any previous test state for product 1 (Chai) and product 2 (Chang)
UPDATE products SET unit_price = 18.00 WHERE product_id = 1;
UPDATE products SET units_in_stock = 17 WHERE product_id = 2;

-- Set isolation level to REPEATABLE READ
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Start Transaction 1
BEGIN;

-- Initial read: Get total units in stock for products in 'Beverages' (category_id = 1)
-- and their individual unit prices.
SELECT
    p.product_id,
    p.product_name,
    p.units_in_stock,
    p.unit_price
FROM
    products p
WHERE
    p.category_id = 1
ORDER BY p.product_id;

-- *** PAUSE HERE ***
-- Now switch to Session 2 and run its commands.
-- After Session 2 commits, switch back to Session 1.

-- Second read: After a concurrent update, re-read the data.
-- Specifically, check the average unit price for beverages.
SELECT
    AVG(p.unit_price) AS average_beverage_price
FROM
    products p
WHERE
    p.category_id = 1;

-- Compare the original unit_price for Product ID 1 (Chai) and Product ID 2 (Chang)
-- from your first select with the actual values in the database after the update.
SELECT
    p.product_id,
    p.product_name,
    p.unit_price
FROM
    products p
WHERE
    p.product_id IN (1, 2);

-- End Transaction 1
COMMIT;

-- Reset isolation level to default for future labs
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**Session 2: Inventory Update**

```sql
-- Start Transaction 2
BEGIN;

-- Update the unit_price of 'Chai' (product_id 1) and 'Chang' (product_id 2) in 'Beverages'
UPDATE products SET unit_price = 20.00 WHERE product_id = 1;
UPDATE products SET units_in_stock = 20 WHERE product_id = 2; -- Change stock too

-- Commit Transaction 2
COMMIT;
```

**Expected Output/Explanation:**
*   **Session 1 - First `SELECT`:** Shows original prices for Chai (18.00) and Chang (19.00).
*   **Session 2 - `UPDATE` & `COMMIT`:** Changes prices.
*   **Session 1 - Second `SELECT` (AVG):** This will still show the **original average price** based on the *snapshot taken at the beginning of Transaction 1*. The `unit_price` for Product ID 1 will still be seen as 18.00 by Session 1, and Product ID 2 as 19.00, even though Session 2 committed new values. The second explicit `SELECT` will also show the original values.

**Why it would be wrong (or impossible in lower isolation) for this scenario:**
For a report like "Potential Revenue Report" or any calculation that spans multiple queries within a single logical unit, a `Non-Repeatable Read` (possible in `READ COMMITTED`) is problematic. The data changes between reads, making the final result unreliable or inconsistent with the initial state. For instance, if the manager sums stock and prices to estimate potential revenue, and prices change mid-way, their final revenue calculation will be based on a mix of old and new data, which is logically flawed. `REPEATABLE READ` enforces a consistent snapshot, preventing this exact anomaly.

**Learning Point:**
This task clearly demonstrates how `REPEATABLE READ` prevents **Non-Repeatable Reads**. All reads within a `REPEATABLE READ` transaction see a consistent snapshot of the database as it existed at the moment the transaction began. This ensures data stability for complex calculations or reports that require multiple reads of the same data, making the operation logically sound.

---

#### **Task 3: Demonstrate Phantom Read in `REPEATABLE READ` (PostgreSQL Nuance)**

**Description:**
As discussed, PostgreSQL's `REPEATABLE READ` still allows "phantom reads" caused by newly inserted rows. Let's test this: Session 1 counts all products in 'Beverages', Session 2 inserts a *new* product into 'Beverages', and then Session 1 re-counts.

**Application Context:** Auditing or summary reports where the count of items in a category must remain stable throughout a complex process. If the count changes unexpectedly, downstream calculations or validations might be erroneous.

**Instructions:**
1.  **Session 1:** Set isolation level, then run initial `BEGIN` and `SELECT count(*)`.
2.  **Session 2:** Run `INSERT` and `COMMIT`.
3.  **Session 1:** Run second `SELECT count(*)` and `COMMIT`.

**SQL Query:**

**Session 1: Manager Report (REPEATABLE READ)**

```sql
-- Ensure new product ID for test is free and product is deleted if left from previous runs
DELETE FROM products WHERE product_id = 9999;

-- Set isolation level to REPEATABLE READ
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Start Transaction 1
BEGIN;

-- Initial read: Count products in 'Beverages' (category_id = 1)
SELECT
    COUNT(*) AS initial_beverage_count
FROM
    products p
WHERE
    p.category_id = 1;

-- *** PAUSE HERE ***
-- Now switch to Session 2 and run its commands.
-- After Session 2 commits, switch back to Session 1.

-- Second read: After a concurrent INSERT, re-count the products.
SELECT
    COUNT(*) AS final_beverage_count
FROM
    products p
WHERE
    p.category_id = 1;

-- End Transaction 1
COMMIT;

-- Reset isolation level to default for future labs
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Clean up (delete the inserted product)
DELETE FROM products WHERE product_id = 9999;
```

**Session 2: Inventory Update**

```sql
-- Start Transaction 2
BEGIN;

-- Insert a NEW product into 'Beverages' category (category_id = 1)
INSERT INTO products (product_id, product_name, supplier_id, category_id, discontinued, unit_price, units_in_stock)
VALUES (9999, 'New Test Beverage', 1, 1, 0, 10.00, 10);

-- Commit Transaction 2
COMMIT;
```

**Expected Output/Explanation:**
*   **Session 1 - First `SELECT`:** Shows the initial count of 'Beverages' products (e.g., 12).
*   **Session 2 - `INSERT` & `COMMIT`:** Adds a new product.
*   **Session 1 - Second `SELECT`:** This will still show the **initial count** (e.g., 12), *not* the increased count (e.g., 13).

**Why this is a Phantom Read (and how PG handles it):**
The definition of a Phantom Read is when the *set of rows* matching a `WHERE` clause changes. Even though Session 1 did not see the *new* row in its count at `REPEATABLE READ`, conceptually, if it had tried to *process* these rows (e.g., `SELECT * FOR UPDATE`), it *could* have encountered blocking or errors because the new row exists. PostgreSQL prevents serialization anomalies by guaranteeing that if your transaction modifies data, and another transaction modifies data *that would conflict with your modifications (e.g., by inserting a row that you would have locked)*, one of the transactions will fail. This is why it works differently from other databases.

**Learning Point:**
This task demonstrates that while PostgreSQL's `REPEATABLE READ` prevents non-repeatable reads (changes to existing rows), it *also* prevents *phantom reads from new inserts* from affecting the logical consistency of your transaction. The *count* doesn't change, meaning your transaction maintains a stable view of data existence. If you tried to *modify* rows that include the new phantom row at this level, PostgreSQL might detect a serialization anomaly and raise an error on `COMMIT`.

---

#### **Task 4: Guaranteeing Full Isolation with `SERIALIZABLE`**

**Description:**
To truly guarantee that no concurrency anomalies (including all types of phantom reads, even from new inserts, and more complex issues like write skew) can occur, `SERIALIZABLE` isolation is required. Let's rerun Task 3 (Phantom Read scenario) under `SERIALIZABLE` isolation. In PostgreSQL, `SERIALIZABLE` achieves this by using a pessimistic concurrency control scheme (snapshot isolation augmented with checks for serialization anomalies).

**Application Context:** Highly sensitive operations (e.g., financial ledger consistency, complex multi-user reservation systems, scientific simulations) where even the slightest inconsistency from concurrent operations is unacceptable.

**Instructions:**
1.  **Session 1:** Set isolation level, then run initial `BEGIN` and `SELECT count(*)`.
2.  **Session 2:** Run `INSERT` and `COMMIT`.
3.  **Session 1:** Run second `SELECT count(*)`, then `COMMIT`. Observe the potential `serialization_failure` error.

**SQL Query:**

**Session 1: Manager Report (SERIALIZABLE)**

```sql
-- Ensure new product ID for test is free and product is deleted if left from previous runs
DELETE FROM products WHERE product_id = 9999;

-- Set isolation level to SERIALIZABLE
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Start Transaction 1
BEGIN;

-- Initial read: Count products in 'Beverages' (category_id = 1)
SELECT
    COUNT(*) AS initial_beverage_count
FROM
    products p
WHERE
    p.category_id = 1;

-- *** PAUSE HERE ***
-- Now switch to Session 2 and run its commands.
-- After Session 2 commits, switch back to Session 1.

-- Second read: After a concurrent INSERT, re-count the products.
-- This SELECT alone won't cause the error. The conflict is detected on COMMIT.
SELECT
    COUNT(*) AS final_beverage_count
FROM
    products p
WHERE
    p.category_id = 1;

-- End Transaction 1
COMMIT; -- This is where a serialization_failure might occur if a conflict is detected.

-- Reset isolation level to default for future labs
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Clean up (delete the inserted product)
DELETE FROM products WHERE product_id = 9999;
```

**Session 2: Inventory Update**

```sql
-- Start Transaction 2
BEGIN;

-- Insert a NEW product into 'Beverages' category (category_id = 1)
INSERT INTO products (product_id, product_name, supplier_id, category_id, discontinued, unit_price, units_in_stock)
VALUES (9999, 'New Test Beverage', 1, 1, 0, 10.00, 10);

-- Commit Transaction 2
COMMIT;
```

**Expected Output/Explanation:**
*   **Session 1 - First `SELECT`:** Shows the initial count of 'Beverages' products.
*   **Session 2 - `INSERT` & `COMMIT`:** Adds a new product.
*   **Session 1 - Second `SELECT`:** Will show the *original count* (like `REPEATABLE READ`).
*   **Session 1 - `COMMIT`:** This is the critical step. PostgreSQL will detect that Session 1's view of the data (the count of products in beverages) became "stale" relative to a committed concurrent transaction (Session 2 inserting a new beverage product). PostgreSQL will abort Session 1's transaction with an `ERROR: could not serialize access due to concurrent update` (SQLSTATE `40001`). This is PostgreSQL's way of preventing the serialization anomaly. The expectation is that the client application retries the failed transaction.

**Learning Point:**
This task demonstrates `SERIALIZABLE` isolation. PostgreSQL achieves full serializability by detecting potential conflicts and aborting one of the conflicting transactions. In this phantom read scenario, the `COMMIT` of the `SERIALIZABLE` transaction detects that the snapshot it was operating on is no longer truly serializable with respect to the `INSERT` from Session 2. This guarantees that operations *always* yield the same result as if executed serially, but requires client-side retry logic for transactions that encounter serialization failures.

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **Write Skew (Advanced `SERIALIZABLE` Test):** Design a scenario involving two transactions, each reading and then updating *different* data, but where the combination of their updates violates a business rule if run concurrently. Show `READ COMMITTED` allows it, but `SERIALIZABLE` causes a serialization failure. Example: two products sharing a total stock limit.
2.  **Explicit Locking:** Explore `SELECT FOR SHARE` and `SELECT FOR UPDATE` clauses. How do they interact with isolation levels? Can they reduce the likelihood of serialization failures (by acquiring locks early) or non-repeatable reads in `READ COMMITTED` (by blocking concurrent updates)?
3.  **Isolation Level Trade-offs:** Discuss the implications of choosing a higher isolation level (e.g., increased resource usage, higher contention, more transaction retries for `SERIALIZABLE`). When is each level appropriate for different parts of a Northwind application (e.g., `READ COMMITTED` for browsing, `REPEATABLE READ` for batch reports, `SERIALIZABLE` for critical financial transfers)?
4.  **Clean up:** Remember to `DELETE FROM products WHERE product_id = 9999;` and `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` in both sessions after completing the lab.

---

### **Conclusion:**

In this lab, you gained a comprehensive understanding of PostgreSQL's **Transaction Isolation Levels**. You clearly observed the **Non-Repeatable Read** anomaly at `READ COMMITTED` and how `REPEATABLE READ` prevents it by providing a consistent data snapshot. Furthermore, you delved into PostgreSQL's specific implementation of `REPEATABLE READ` and how it handles Phantom Reads. Finally, you experienced the strong consistency guarantees of `SERIALIZABLE` isolation, which detects and prevents all concurrency anomalies, requiring client-side retry mechanisms. This knowledge is fundamental for designing robust and correct concurrent applications in a multi-user database environment like Northwind.