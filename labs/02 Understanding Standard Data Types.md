
### Understanding Standard Data Types (`BOOLEAN`, `NUMERIC`, `CHAR`, `DATETIME`)

**Objective:**
This lab will provide practical experience with fundamental PostgreSQL data types including `BOOLEAN`, `NUMERIC`, `CHAR`, and `DATETIME`. You will learn how to define these types, insert data into them, and query data using them within the context of a new order tracking table, simulating a real-life Northwind scenario.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
    *   *(If not installed, please refer to the installation instructions from the previous interaction or a reliable source for Northwind PostgreSQL setup.)*
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line, DBeaver, VS Code with PostgreSQL extension).
4.  **Basic SQL Knowledge:** Familiarity with fundamental SQL syntax (`SELECT`, `FROM`, `WHERE`).

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run a simple query to confirm connectivity and data presence (e.g., `SELECT count(*) FROM customers;`). You should see a count reflecting the number of customers.

---

### **Lab Exercises:**

**Introduction to Tables Used:**
*   `orders`: We will reference existing orders to link our new tracking information.
*   `OrderTracking` (new table): This temporary table will be created within this lab to demonstrate the specified data types. This table simulates tracking additional details about Northwind orders.

---

#### **Task 1: Creating a Custom Order Tracking Table**

**Description:**
You need to create a new temporary table named `OrderTracking` to store additional fulfillment details for Northwind orders. This table will use various standard data types.

The `OrderTracking` table should have the following columns:
*   `tracking_id`: A unique identifier for each tracking entry (auto-incrementing integer).
*   `order_id`: The ID of the order being tracked (links to the `orders` table).
*   `is_priority`: A flag indicating if the order is high priority (true/false).
*   `delivery_status_code`: A single character code representing the delivery status (e.g., 'P' for Pending, 'S' for Shipped, 'D' for Delivered).
*   `estimated_delivery_date`: The anticipated date of delivery.
*   `actual_delivery_date`: The actual date the order was delivered (can be NULL initially).
*   `freight_cost_usd`: The cost of shipping in USD, with up to two decimal places.

**SQL Query:**

```sql
DROP TABLE IF EXISTS OrderTracking; -- Drop if exists for clean slate

CREATE TABLE OrderTracking (
    tracking_id SERIAL PRIMARY KEY,
    order_id SMALLINT REFERENCES orders(order_id),
    is_priority BOOLEAN,
    delivery_status_code CHAR(1),
    estimated_delivery_date DATE,
    actual_delivery_date DATE,
    freight_cost_usd NUMERIC(10, 2)
);
```

**Expected Output/Explanation:**
The query should execute successfully, indicated by a "CREATE TABLE" message. No rows will be returned as this is a DDL (Data Definition Language) statement. You can verify the table creation by checking your SQL client's schema browser or by running `\dt OrderTracking` in `psql`.

**Learning Point:**
This task demonstrates the `CREATE TABLE` statement and the declaration of fundamental PostgreSQL data types: `SERIAL` (for auto-incrementing integer IDs), `SMALLINT` (for small integer numbers), `BOOLEAN` (for true/false values), `CHAR(1)` (for fixed-length single characters), `DATE` (for date values without time), and `NUMERIC(precision, scale)` (for precise decimal numbers like currency). The `REFERENCES` clause also shows basic foreign key constraint definition.

---

#### **Task 2: Inserting Sample Tracking Data**

**Description:**
Now, populate your `OrderTracking` table with some sample data. Insert at least three rows that showcase different values for the new data types. Try to pick existing `order_id` values from the `orders` table.

*   Insert one pending priority order.
*   Insert one shipped non-priority order.
*   Insert one delivered priority order.

**SQL Query:**

```sql
INSERT INTO OrderTracking (order_id, is_priority, delivery_status_code, estimated_delivery_date, actual_delivery_date, freight_cost_usd)
VALUES
    (10248, TRUE, 'P', '1996-07-20', NULL, 32.38),
    (10249, FALSE, 'S', '1996-07-15', NULL, 11.61),
    (10250, TRUE, 'D', '1996-07-12', '1996-07-12', 65.83),
    (10251, FALSE, 'P', '1996-08-10', NULL, 41.34);
```

**Expected Output/Explanation:**
You should see a message indicating "INSERT 0 4" (or similar, depending on how many rows you inserted), meaning 4 rows were successfully added.

**Learning Point:**
This task demonstrates the `INSERT INTO` statement, focusing on how to provide values for `BOOLEAN` (using `TRUE`/`FALSE` or `'t'`/`'f'`), `CHAR` (single characters in quotes), `DATE` (using standard `YYYY-MM-DD` string format), and `NUMERIC` values. It also shows `NULL` insertion for columns where data isn't yet available.

---

#### **Task 3: Querying Boolean and Numeric Data**

**Description:**
Retrieve all tracking entries for high-priority orders (`is_priority` is TRUE) with a shipping cost greater than 50. Display the `order_id`, `is_priority`, and `freight_cost_usd`.

**SQL Query:**

```sql
SELECT
    order_id,
    is_priority,
    freight_cost_usd
FROM
    OrderTracking
WHERE
    is_priority = TRUE AND freight_cost_usd > 50;
```

**Expected Output/Explanation:**
You should see at least one row for `order_id` 10250, with `is_priority` as 't' (or true) and `freight_cost_usd` as 65.83 (depending on your inserted data, others might appear if their freight cost meets the criteria).

**Learning Point:**
This task highlights how to filter data using `BOOLEAN` columns directly (`is_priority = TRUE`) and `NUMERIC` columns using comparison operators (`>`). It reinforces combining conditions with `AND`.

---

#### **Task 4: Querying Character and Datetime Data**

**Description:**
Find all orders that are currently 'Pending' (`delivery_status_code` = 'P') and have an `estimated_delivery_date` in July 1996. Display all columns from the `OrderTracking` table.

**SQL Query:**

```sql
SELECT
    *
FROM
    OrderTracking
WHERE
    delivery_status_code = 'P' AND estimated_delivery_date BETWEEN '1996-07-01' AND '1996-07-31';
```

**Expected Output/Explanation:**
You should see the row for `order_id` 10248, showing its 'P' status and estimated delivery date in July 1996.

**Learning Point:**
This task demonstrates filtering using `CHAR` columns with exact matches and `DATE` columns using the `BETWEEN` operator for a date range. It also shows `SELECT *` to retrieve all columns.

---

#### **Task 5: Updating Tracking Information with Datetime and Boolean**

**Description:**
Simulate an update: Mark the order with `order_id` 10249 as delivered and set its `actual_delivery_date` to '1996-07-10'. Also, change its `delivery_status_code` to 'D'.

**SQL Query:**

```sql
UPDATE
    OrderTracking
SET
    actual_delivery_date = '1996-07-10',
    delivery_status_code = 'D'
WHERE
    order_id = 10249;

-- Verify the update
SELECT * FROM OrderTracking WHERE order_id = 10249;
```

**Expected Output/Explanation:**
The `UPDATE` statement should return "UPDATE 1" (or similar), indicating one row was affected. The subsequent `SELECT` query should show the updated `actual_delivery_date` and `delivery_status_code` for order 10249.

**Learning Point:**
This task demonstrates the `UPDATE` statement, focusing on how to modify `DATE` and `CHAR` column values for existing records. It shows the practical use of updating status and recording actual timestamps.

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **Calculate Average Freight Cost:** Calculate the average `freight_cost_usd` for all priority orders (`is_priority = TRUE`).
2.  **Count by Status:** Count how many orders are in each `delivery_status_code`.
3.  **Shipped but Not Delivered:** Find orders that have `delivery_status_code` as 'S' but still have a `NULL` for `actual_delivery_date`.
4.  **Join with Original Orders:** Join your `OrderTracking` table with the `orders` table to display the customer name for each tracked order.

**Further Exploration:**
*   Research other PostgreSQL `NUMERIC` types (`INTEGER`, `DECIMAL`, `FLOAT`).
*   Explore other PostgreSQL `DATETIME` types (`TIMESTAMP`, `TIME`) and their differences.
*   Look into `VARCHAR` vs `CHAR` data types for strings. Why might `CHAR(1)` be preferred over `VARCHAR(1)` here?
*   Practice deleting records from `OrderTracking` using `DELETE` and filtering.

---

In this lab, you gained practical experience defining, inserting, and querying data using essential PostgreSQL data types: `BOOLEAN`, `NUMERIC`, `CHAR`, and `DATETIME`. By creating and manipulating a custom `OrderTracking` table related to the Northwind database, you reinforced your understanding of how these data types are used to represent different kinds of real-world information and how to interact with them effectively using SQL.