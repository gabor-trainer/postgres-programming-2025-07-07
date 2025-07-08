### Mastering Stored Procedures and Transaction Management

**Objective:**
This comprehensive lab will provide in-depth hands-on experience with PostgreSQL `STORED PROCEDURES`. You will learn how to define procedures, handle `IN`/`OUT` parameters, implement complex multi-step business logic, and, most importantly, manage transactions explicitly using `COMMIT`, `ROLLBACK`, and `SAVEPOINT`. The lab will cover real-life Northwind scenarios, including customer payment processing, atomic order fulfillment, monthly sales reporting, and conditional product reclassification with partial rollbacks.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended) that supports `CREATE PROCEDURE`.
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
    *   *(If not installed, please refer to the installation instructions from the previous interaction or a reliable source for Northwind PostgreSQL setup.)*
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line, DBeaver, VS Code with PostgreSQL extension).
4.  **Basic SQL Knowledge:** Familiarity with fundamental SQL syntax (`SELECT`, `FROM`, `WHERE`, `JOIN`, `INSERT`, `UPDATE`, `DELETE`).
5.  **PL/pgSQL Knowledge:** Understanding of `DECLARE`, `BEGIN...END`, `SELECT INTO`, `IF` statements, `FOR` loops, `RAISE EXCEPTION`, and basic `EXCEPTION WHEN OTHERS` handling. (Refer to the previous PL/pgSQL functions lab).

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run a simple query to confirm connectivity and data presence (e.g., `SELECT count(*) FROM customers;`). You should see a count reflecting the number of customers.
3.  **Enable PL/pgSQL:** Ensure PL/pgSQL is active. (Run `CREATE EXTENSION IF NOT EXISTS plpgsql;` if you encounter language-related errors).
4.  **Prepare Customers Table (Add `credit_limit`):** For one of the tasks, we need a `credit_limit` column on the `customers` table.

```sql
ALTER TABLE customers ADD COLUMN IF NOT EXISTS credit_limit NUMERIC(10, 2) DEFAULT 10000.00;
-- Set some example credit limits for a few customers
UPDATE customers SET credit_limit = 15000.00 WHERE customer_id = 'ALFKI';
UPDATE customers SET credit_limit = 5000.00 WHERE customer_id = 'ANATR';
UPDATE customers SET credit_limit = 20000.00 WHERE customer_id = 'ERNSH';
```

---

### **Lab Exercises:**

**Introduction to Tables Used:**
*   `customers`: Will be updated for credit limits and customer payments.
*   `employees`: Will be used for employee reporting status.
*   `products`: Inventory adjustments and reclassification.
*   `orders`, `order_details`: Core of order processing and sales reporting.
*   `CustomerPayments` (new table): To log customer payments.
*   `InventoryLog` (new table): To track inventory changes.
*   `MonthlySalesSummary` (new table): To store aggregated monthly sales data.

---

#### **Task 1: Basic Procedure with `OUT` Parameters - Employee Reporting Status**

**Description:**
Northwind's HR department wants a quick way to check an employee's reporting line. Create a stored procedure named `get_employee_reporting_status` that, given an `employee_id`, returns their `first_name`, `last_name`, and a `reporting_status` indicating whether they 'Report to CEO', 'Report to Manager', or have 'No Reports' (for those who are managers or don't report to anyone).

**Application Context:** Quick HR inquiries, organizational hierarchy checks.

**SQL Query:**

```sql
CREATE OR REPLACE PROCEDURE get_employee_reporting_status(
    IN p_employee_id SMALLINT,
    OUT p_first_name VARCHAR(10),
    OUT p_last_name VARCHAR(20),
    OUT p_reporting_status TEXT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_reports_to SMALLINT;
BEGIN
    SELECT first_name, last_name, reports_to
    INTO p_first_name, p_last_name, v_reports_to
    FROM employees
    WHERE employee_id = p_employee_id;

    IF NOT FOUND THEN
        p_first_name := NULL;
        p_last_name := NULL;
        p_reporting_status := 'Employee Not Found';
        RETURN;
    END IF;

    IF v_reports_to IS NULL THEN
        p_reporting_status := 'No Reports (Top Level)';
    ELSE
        -- Check if reports_to is the CEO (employee_id 2 in Northwind, Andrew Fuller)
        IF v_reports_to = 2 THEN
            p_reporting_status := 'Reports to CEO';
        ELSE
            p_reporting_status := 'Reports to Manager';
        END IF;
    END IF;

END;
$$;

-- --- Test Cases ---
DO $$
DECLARE
    v_fname VARCHAR(10);
    v_lname VARCHAR(20);
    v_status TEXT;
BEGIN
    -- Test 1: Employee reports to CEO (e.g., Nancy Davolio, employee_id 1 reports to Andrew Fuller, employee_id 2)
    CALL get_employee_reporting_status(1, v_fname, v_lname, v_status);
    RAISE NOTICE 'Employee: % % | Status: %', v_fname, v_lname, v_status;

    -- Test 2: Employee is CEO (Andrew Fuller, employee_id 2, reports_to NULL)
    CALL get_employee_reporting_status(2, v_fname, v_lname, v_status);
    RAISE NOTICE 'Employee: % % | Status: %', v_fname, v_lname, v_status;

    -- Test 3: Employee reports to a manager (e.g., Michael Suyama, employee_id 6 reports to Steven Buchanan, employee_id 5)
    CALL get_employee_reporting_status(6, v_fname, v_lname, v_status);
    RAISE NOTICE 'Employee: % % | Status: %', v_fname, v_lname, v_status;

    -- Test 4: Non-existent employee
    CALL get_employee_reporting_status(99, v_fname, v_lname, v_status);
    RAISE NOTICE 'Employee: % % | Status: %', v_fname, v_lname, v_status;
END;
$$;
```

**Expected Output/Explanation:**
*   Test 1: `Nancy Davolio | Reports to CEO`
*   Test 2: `Andrew Fuller | No Reports (Top Level)`
*   Test 3: `Michael Suyama | Reports to Manager`
*   Test 4: `(null) (null) | Employee Not Found`

**Learning Point:**
This task introduces the basic `CREATE PROCEDURE` syntax with `IN` and `OUT` parameters. It demonstrates how to assign values to `OUT` parameters within the PL/pgSQL block and use `IF/ELSIF/ELSE` for conditional logic. Procedures do not `RETURN` a value in the function sense, but rather signal completion and provide `OUT` parameter values.

---

#### **Task 2: Transactional Procedure - Process Customer Payment**

**Description:**
Northwind needs a robust system to process customer payments. Create a stored procedure `process_customer_payment` that handles a payment transaction.

The procedure should:
1.  Take `p_customer_id` (character varying) and `p_payment_amount` (numeric) as input.
2.  Validate: `p_payment_amount` must be positive.
3.  **Update:** Increase the `credit_limit` of the specified customer by `p_payment_amount` (simulating receiving payment which increases available credit/reduces outstanding balance).
4.  **Log:** Insert a record into a `CustomerPayments` table, recording the `customer_id`, `payment_amount`, and `payment_date`.
5.  **Transaction Control:**
    *   If the customer ID is not found, `RAISE EXCEPTION` and `ROLLBACK` the entire operation.
    *   If `p_payment_amount` is not positive, `RAISE EXCEPTION` and `ROLLBACK`.
    *   If all steps are successful, `COMMIT` the transaction.

**Application Context:** Financial transaction processing, ensuring atomicity of payment updates and logging.

**SQL Query:**

```sql
-- 1. Create the CustomerPayments table
DROP TABLE IF EXISTS CustomerPayments CASCADE;
CREATE TABLE CustomerPayments (
    payment_id SERIAL PRIMARY KEY,
    customer_id CHARACTER VARYING(5) NOT NULL,
    payment_amount NUMERIC(10, 2) NOT NULL,
    payment_date TIMESTAMP DEFAULT NOW()
);

-- 2. Create the procedure
CREATE OR REPLACE PROCEDURE process_customer_payment(
    IN p_customer_id CHARACTER VARYING,
    IN p_payment_amount NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Explicitly start a transaction if not already in one, or continue the caller's transaction.
    -- Procedures give us the power to commit/rollback.

    -- Validation 1: Payment amount must be positive
    IF p_payment_amount <= 0 THEN
        RAISE EXCEPTION 'Payment amount must be positive.';
    END IF;

    -- Update customer's credit limit
    UPDATE customers
    SET credit_limit = credit_limit + p_payment_amount
    WHERE customer_id = p_customer_id;

    -- Validation 2: Check if customer was found
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Customer ID % not found.', p_customer_id;
    END IF;

    -- Log the payment
    INSERT INTO CustomerPayments (customer_id, payment_amount)
    VALUES (p_customer_id, p_payment_amount);

    RAISE NOTICE 'Payment of % received and processed for customer %.', p_payment_amount, p_customer_id;
    COMMIT; -- Explicitly commit the transaction on success

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Payment processing failed for customer %: % - %', p_customer_id, SQLSTATE, SQLERRM;
        ROLLBACK; -- Explicitly roll back on any error
        RAISE; -- Re-raise the original exception after rollback
END;
$$;

-- --- Test Cases ---

-- Before: Check initial credit_limit and payment log for ALFKI
SELECT customer_id, company_name, credit_limit FROM customers WHERE customer_id = 'ALFKI';
SELECT * FROM CustomerPayments WHERE customer_id = 'ALFKI';

-- Test 1: Successful payment for ALFKI
CALL process_customer_payment('ALFKI', 1000.00);

-- After: Verify updated credit_limit and payment log
SELECT customer_id, company_name, credit_limit FROM customers WHERE customer_id = 'ALFKI';
SELECT * FROM CustomerPayments WHERE customer_id = 'ALFKI';

-- Test 2: Payment for non-existent customer (should FAIL and ROLLBACK)
CALL process_customer_payment('NONEX', 500.00);

-- After: Verify that NO changes were made (check credit_limit for existing customers, and CustomerPayments table)
SELECT customer_id, company_name, credit_limit FROM customers WHERE customer_id = 'ALFKI';
SELECT * FROM CustomerPayments WHERE customer_id = 'NONEX'; -- Should be empty

-- Test 3: Payment with zero amount (should FAIL and ROLLBACK)
CALL process_customer_payment('ANATR', 0.00);

-- After: Verify that NO changes were made
SELECT customer_id, company_name, credit_limit FROM customers WHERE customer_id = 'ANATR';
SELECT * FROM CustomerPayments WHERE customer_id = 'ANATR'; -- Should be empty

-- Clean up: Reset ALFKI's credit_limit and clear payments
UPDATE customers SET credit_limit = 15000.00 WHERE customer_id = 'ALFKI';
TRUNCATE TABLE CustomerPayments;
```

**Expected Output/Explanation:**
*   Test 1: `NOTICE: Payment of 1000.00 received and processed for customer ALFKI.`, `COMMIT` will be implicitly called. `ALFKI`'s `credit_limit` will increase, and a new payment record will appear.
*   Test 2: `NOTICE: Payment processing failed for customer NONEX: ...`, followed by `ERROR` about customer not found. `ROLLBACK` will be called. No changes to any table.
*   Test 3: `NOTICE: Payment processing failed for customer ANATR: ...`, followed by `ERROR` about payment amount. `ROLLBACK` will be called. No changes to any table.

**Learning Point:**
This task demonstrates the core power of procedures: explicit `COMMIT` and `ROLLBACK` within the procedural block. The `EXCEPTION` handler ensures that any error (including those raised by `RAISE EXCEPTION` or SQL errors) leads to a full rollback, preserving data consistency.

---

#### **Task 3: Transactional Procedure - Atomic Order Dispatch with Inventory Adjustment**

**Description:**
When an order is dispatched, Northwind needs to atomically adjust inventory and mark the order as shipped. Create a stored procedure `dispatch_order`.

The procedure should:
1.  Take `p_order_id` (smallint) as input.
2.  Validate: The order must exist and not already be shipped (no `shipped_date`).
3.  **Iterate:** Loop through each product in `order_details` for the given order.
4.  **For each item:**
    *   Validate: Product exists and has sufficient `units_in_stock`.
    *   **Update:** Decrease `units_in_stock` in the `products` table.
    *   **Log:** Insert a record into a `InventoryLog` table.
5.  **Final Update:** Set `shipped_date` for the order in the `orders` table to `NOW()::DATE`.
6.  **Transaction Control:** If at *any point* an item has insufficient stock, or any other error occurs, the *entire dispatch operation* (all stock reductions, order update, inventory logs) must `ROLLBACK`. Otherwise, `COMMIT`.

**Application Context:** Core business operation, ensuring inventory accuracy and order fulfillment atomicity.

**SQL Query:**

```sql
-- 1. Create InventoryLog table (if not exists from previous labs)
DROP TABLE IF EXISTS InventoryLog CASCADE;
CREATE TABLE InventoryLog (
    log_id SERIAL PRIMARY KEY,
    product_id SMALLINT,
    quantity_change SMALLINT,
    log_type TEXT, -- e.g., 'ORDER_FULFILLMENT', 'STOCK_ADJUSTMENT'
    order_id SMALLINT,
    log_timestamp TIMESTAMP DEFAULT NOW()
);

-- 2. Create the procedure
CREATE OR REPLACE PROCEDURE dispatch_order(IN p_order_id SMALLINT)
LANGUAGE plpgsql
AS $$
DECLARE
    order_details_rec RECORD;
    v_customer_id CHARACTER VARYING;
    v_order_shipped_date DATE;
    v_product_name VARCHAR(40);
    v_current_stock SMALLINT;
BEGIN
    -- Check if order exists and is not already shipped
    SELECT customer_id, shipped_date
    INTO v_customer_id, v_order_shipped_date
    FROM orders
    WHERE order_id = p_order_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Order ID % not found.', p_order_id;
    END IF;

    IF v_order_shipped_date IS NOT NULL THEN
        RAISE EXCEPTION 'Order ID % has already been shipped on %.', p_order_id, v_order_shipped_date;
    END IF;

    -- Loop through each item in the order for validation and update
    FOR order_details_rec IN
        SELECT od.product_id, od.quantity, p.product_name, p.units_in_stock
        FROM order_details od
        JOIN products p ON od.product_id = p.product_id
        WHERE od.order_id = p_order_id
    LOOP
        -- Check for sufficient stock before decreasing
        v_product_name := order_details_rec.product_name;
        v_current_stock := order_details_rec.units_in_stock;

        IF v_current_stock < order_details_rec.quantity THEN
            RAISE EXCEPTION 'Insufficient stock for product "%" (ID: %). Available: %, Requested: % in order %.',
                            v_product_name, order_details_rec.product_id, v_current_stock, order_details_rec.quantity, p_order_id;
        END IF;

        -- Decrease stock
        UPDATE products
        SET units_in_stock = units_in_stock - order_details_rec.quantity
        WHERE product_id = order_details_rec.product_id;

        -- Log the inventory change
        INSERT INTO InventoryLog (product_id, quantity_change, log_type, order_id)
        VALUES (order_details_rec.product_id, -order_details_rec.quantity, 'ORDER_DISPATCH', p_order_id);
    END LOOP;

    -- Update order shipped_date
    UPDATE orders
    SET shipped_date = NOW()::DATE
    WHERE order_id = p_order_id;

    RAISE NOTICE 'Order % successfully dispatched and inventory updated.', p_order_id;
    COMMIT; -- Commit the entire transaction

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Order dispatch failed for order %: % - %', p_order_id, SQLSTATE, SQLERRM;
        ROLLBACK; -- Rollback all changes if any error occurs
        RAISE; -- Re-raise the exception
END;
$$;

-- --- Test Cases ---

-- Find an unshipped order, e.g., 10248 (shipped_date is '1996-07-16')
-- Find one that is NOT shipped, e.g. 11008 (shipped_date is NULL)
SELECT order_id, customer_id, shipped_date FROM orders WHERE order_id = 11008;
-- Check stock for products in order 11008: product 28 (stock 26), 34 (stock 111), 71 (stock 26)
SELECT product_id, product_name, units_in_stock FROM products WHERE product_id IN (28, 34, 71);
SELECT * FROM order_details WHERE order_id = 11008; -- Quantities: 28:70, 34:90, 71:21
-- Note: product 28:70 and 34:90 are much higher than current stock. This will cause failure.
-- Adjust product 28 and 34 stock to allow success for the lab.
UPDATE products SET units_in_stock = 100 WHERE product_id IN (28, 34);
SELECT product_id, product_name, units_in_stock FROM products WHERE product_id IN (28, 34, 71);

-- Before: Verify initial state for order 11008
SELECT order_id, shipped_date FROM orders WHERE order_id = 11008;
SELECT p.product_id, p.product_name, p.units_in_stock, od.quantity AS ordered_qty
FROM products p JOIN order_details od ON p.product_id = od.product_id
WHERE od.order_id = 11008;
SELECT * FROM InventoryLog WHERE order_id = 11008;

-- Test 1: Successful Dispatch (order 11008)
CALL dispatch_order(11008);

-- After: Verify order shipped, stock reduced, logs created
SELECT order_id, shipped_date FROM orders WHERE order_id = 11008; -- Shipped date should be set
SELECT p.product_id, p.product_name, p.units_in_stock
FROM products p JOIN order_details od ON p.product_id = od.product_id
WHERE od.order_id = 11008; -- Stock should be reduced by ordered quantities
SELECT * FROM InventoryLog WHERE order_id = 11008; -- Should have 3 log entries

-- Test 2: Dispatch of already shipped order (e.g., 10248)
CALL dispatch_order(10248); -- Should fail

-- Test 3: Dispatch with insufficient stock (re-create situation)
-- Reset order 11009 for re-test, it's customer_id = 'GODOS', employee_id = 2, shipped_date=NULL
-- Restore stock levels of product 11 (Queso Cabrales) and product 69 (Gudbrandsdalsost) for order 11009.
UPDATE products SET units_in_stock = 1 WHERE product_id = 11; -- Stock 1 (Ordered 15)
UPDATE products SET units_in_stock = 1 WHERE product_id = 69; -- Stock 1 (Ordered 10)
UPDATE products SET units_in_stock = 100 WHERE product_id = 16; -- Stock 100 (Ordered 18)

CALL dispatch_order(11009); -- Should fail due to product 11 insufficient stock

-- After failure: Verify NO order was shipped, stock did NOT change
SELECT order_id, shipped_date FROM orders WHERE order_id = 11009; -- Shipped date should still be NULL
SELECT product_id, units_in_stock FROM products WHERE product_id IN (11, 16, 69); -- Stock should be original levels for all three.
SELECT * FROM InventoryLog WHERE order_id = 11009; -- Should be empty

-- Clean up: Restore data for subsequent runs/other labs
-- Reset order 11008 shipped_date and stock (requires reversing changes)
UPDATE orders SET shipped_date = NULL WHERE order_id = 11008;
UPDATE products SET units_in_stock = units_in_stock + 70 WHERE product_id = 28;
UPDATE products SET units_in_stock = units_in_stock + 90 WHERE product_id = 34;
UPDATE products SET units_in_stock = units_in_stock + 21 WHERE product_id = 71;
-- Restore stock of 11, 16, 69 to original values
UPDATE products SET units_in_stock = 22 WHERE product_id = 11;
UPDATE products SET units_in_stock = 29 WHERE product_id = 16;
UPDATE products SET units_in_stock = 26 WHERE product_id = 69;
TRUNCATE TABLE InventoryLog;
```

**Expected Output/Explanation:**
*   Test 1: `NOTICE: Order 11008 successfully dispatched...`, `COMMIT` will be called. `shipped_date` will be set, `units_in_stock` will decrease, `InventoryLog` will have entries.
*   Test 2: `ERROR: Order ID 10248 has already been shipped...`, `ROLLBACK` will be called. No changes.
*   Test 3: `NOTICE: Order dispatch failed for order 11009: ...`, `ERROR: Insufficient stock...`, `ROLLBACK` will be called. All changes (even for product 11 if its stock was reduced before the `RAISE`) will be undone.

**Learning Point:**
This task reinforces explicit `COMMIT`/`ROLLBACK` for multi-step transactional integrity. It showcases how a `FOR LOOP` iterating over a query can be used to process child records, and how `RAISE EXCEPTION` in such a loop automatically triggers the `ROLLBACK` in the `EXCEPTION` block, guaranteeing atomicity for the entire order dispatch.

---

#### **Task 4: Accounting/Reporting Procedure - Generate Monthly Sales Summary**

**Description:**
Northwind's accounting department needs a monthly sales summary. Create a stored procedure `generate_monthly_sales_summary` that calculates total sales and total freight for a given month and year, and inserts/updates this data into a `MonthlySalesSummary` table.

**Application Context:** Financial reporting, business intelligence, data warehousing (populating summary tables).

**SQL Query:**

```sql
-- 1. Create the MonthlySalesSummary table
DROP TABLE IF EXISTS MonthlySalesSummary CASCADE;
CREATE TABLE MonthlySalesSummary (
    summary_month SMALLINT NOT NULL,
    summary_year SMALLINT NOT NULL,
    total_sales NUMERIC(15, 2) NOT NULL,
    total_freight NUMERIC(15, 2) NOT NULL,
    generated_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (summary_month, summary_year)
);

-- 2. Create the procedure
CREATE OR REPLACE PROCEDURE generate_monthly_sales_summary(
    IN p_month SMALLINT,
    IN p_year SMALLINT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_total_sales NUMERIC(15, 2);
    v_total_freight NUMERIC(15, 2);
BEGIN
    -- Calculate total sales and freight for the specified month/year
    SELECT
        COALESCE(SUM(od.unit_price * od.quantity * (1 - od.discount)), 0),
        COALESCE(SUM(o.freight), 0)
    INTO
        v_total_sales,
        v_total_freight
    FROM
        orders o
    JOIN
        order_details od ON o.order_id = od.order_id
    WHERE
        EXTRACT(MONTH FROM o.order_date) = p_month
        AND EXTRACT(YEAR FROM o.order_date) = p_year;

    -- Insert or update the summary table
    INSERT INTO MonthlySalesSummary (summary_month, summary_year, total_sales, total_freight)
    VALUES (p_month, p_year, v_total_sales, v_total_freight)
    ON CONFLICT (summary_month, summary_year) DO UPDATE SET
        total_sales = EXCLUDED.total_sales,
        total_freight = EXCLUDED.total_freight,
        generated_at = NOW(); -- Update timestamp on refresh

    RAISE NOTICE 'Monthly sales summary generated/updated for %/%', p_month, p_year;
    COMMIT; -- Commit the operation
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Error generating monthly sales summary for %/%: % - %', p_month, p_year, SQLSTATE, SQLERRM;
        ROLLBACK;
        RAISE;
END;
$$;

-- --- Test Cases ---

-- Before: Check summary table (should be empty initially)
SELECT * FROM MonthlySalesSummary;

-- Test 1: Generate summary for a specific month/year (e.g., July 1996)
CALL generate_monthly_sales_summary(7, 1996);

-- After: Verify summary table
SELECT * FROM MonthlySalesSummary;

-- Test 2: Generate summary for another month/year (e.g., April 1998)
CALL generate_monthly_sales_summary(4, 1998);

-- After: Verify both entries
SELECT * FROM MonthlySalesSummary;

-- Test 3: Re-generate summary for July 1996 (should update existing entry)
-- This assumes original data is static for testing.
CALL generate_monthly_sales_summary(7, 1996);
SELECT * FROM MonthlySalesSummary; -- Check generated_at timestamp

-- Test 4: Generate summary for a month/year with no data (e.g., Jan 2000)
CALL generate_monthly_sales_summary(1, 2000);
SELECT * FROM MonthlySalesSummary; -- Should show 0 for totals

-- Clean up
TRUNCATE TABLE MonthlySalesSummary;
```

**Expected Output/Explanation:**
*   Test 1 & 2: `NOTICE: Monthly sales summary generated/updated...`, `COMMIT` called. New entries in `MonthlySalesSummary`.
*   Test 3: `NOTICE: Monthly sales summary generated/updated...`, `COMMIT` called. The existing entry for 7/1996 will be updated (check `generated_at`).
*   Test 4: `NOTICE: Monthly sales summary generated/updated...`, `COMMIT` called. An entry for 1/2000 will appear with 0 for totals.

**Learning Point:**
This task demonstrates using procedures for aggregation and reporting. It shows:
*   Using `EXTRACT` to get month/year from dates.
*   `COALESCE` to handle `SUM` returning `NULL` for no matching rows.
*   `INSERT ... ON CONFLICT DO UPDATE` (UPSERT) for efficiently handling new or existing summary records.
*   Explicit `COMMIT` for a self-contained reporting operation.

---

#### **Task 5: Advanced Transaction Control - Product Reclassification with `SAVEPOINT`**

**Description:**
Northwind sometimes decides to re-enable products that were previously discontinued, but only if they meet certain criteria (e.g., they can be priced competitively). Create a procedure `reclassify_discontinued_products_in_category` that allows for a partial rollback.

The procedure should:
1.  Take `p_category_id` (smallint) and `p_price_increase_percentage` (numeric) as input.
2.  **Start Transaction:** Begin an explicit transaction block.
3.  **Step 1: Re-enable Products:** Update all products in `p_category_id` that are `discontinued = 1` to `discontinued = 0`.
4.  **Set `SAVEPOINT`:** Create a `SAVEPOINT` named `price_update_attempt`.
5.  **Step 2: Attempt Price Update (Transactional within Savepoint):**
    *   Update the `unit_price` of these re-enabled products by `p_price_increase_percentage`.
    *   **Crucial:** Wrap this price update logic in its own `BEGIN...EXCEPTION...END` block.
    *   If the price update fails (e.g., due to an underlying trigger that prevents non-positive prices, or a calculated price becomes too low/high based on some complex validation): `ROLLBACK TO SAVEPOINT price_update_attempt`.
    *   If successful: `RELEASE SAVEPOINT price_update_attempt`.
6.  **Final `COMMIT`:** `COMMIT` the entire transaction. The `discontinued` status change should always persist, even if the price update rolls back.

**Application Context:** Complex multi-step operations where part of the operation might fail, but other parts should still persist.

**SQL Query:**

```sql
-- Before: Verify initial state of product 17 (Alice Mutton, discontinued, category 6, price 39)
-- And product 9 (Mishi Kobe Niku, discontinued, category 6, price 97)
SELECT product_id, product_name, category_id, unit_price, discontinued FROM products WHERE product_id IN (9, 17);

-- Create a temporary product price audit log for this task to see more clearly.
-- DROP TABLE IF EXISTS TempPriceAuditLog CASCADE;
-- CREATE TEMPORARY TABLE TempPriceAuditLog (
--    log_id SERIAL PRIMARY KEY,
--    product_id SMALLINT,
--    old_price REAL,
--    new_price REAL,
--    change_type TEXT,
--    change_timestamp TIMESTAMP DEFAULT NOW()
-- );

-- Optionally, to simulate a price validation error:
-- Let's define a trigger function from previous lab (Task 1) that errors on unit_price <= 0
-- If you already have trg_products_validation from Task 1 running, this is covered.
-- If not, define this simple trigger to simulate failure:
-- CREATE OR REPLACE FUNCTION enforce_positive_price() RETURNS TRIGGER AS $$
-- BEGIN IF NEW.unit_price <= 0 THEN RAISE EXCEPTION 'Price must be positive.'; END IF; RETURN NEW; END;
-- $$ LANGUAGE plpgsql;
-- CREATE TRIGGER trg_enforce_positive_price BEFORE UPDATE ON products FOR EACH ROW EXECUTE FUNCTION enforce_positive_price();


CREATE OR REPLACE PROCEDURE reclassify_discontinued_products_in_category(
    IN p_category_id SMALLINT,
    IN p_price_increase_percentage NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_products_re_enabled_count INT;
    v_new_price REAL;
BEGIN
    -- This procedure starts its own transaction if not called within one.
    -- If called within one, COMMIT/ROLLBACK will affect the parent transaction.

    -- Step 1: Re-enable Discontinued Products in the specified category
    UPDATE products
    SET discontinued = 0 -- Set to active
    WHERE category_id = p_category_id AND discontinued = 1;

    GET DIAGNOSTICS v_products_re_enabled_count = ROW_COUNT;
    RAISE NOTICE 'Re-enabled % products in category %.', v_products_re_enabled_count, p_category_id;

    -- Set SAVEPOINT for the price update attempt
    SAVEPOINT price_update_attempt;
    RAISE NOTICE 'Savepoint "price_update_attempt" set.';

    BEGIN
        -- Step 2: Attempt Price Update
        -- Only update products that were just re-enabled (discontinued = 0 now)
        UPDATE products
        SET unit_price = unit_price * (1 + p_price_increase_percentage / 100.0)
        WHERE category_id = p_category_id AND discontinued = 0; -- Target re-enabled products

        RAISE NOTICE 'Price update attempt successful for re-enabled products in category %. Changes committed within savepoint.', p_category_id;
        RELEASE SAVEPOINT price_update_attempt; -- Commit changes up to this savepoint
        
    EXCEPTION
        WHEN OTHERS THEN
            RAISE NOTICE 'Price update failed for re-enabled products in category %: % - %', p_category_id, SQLSTATE, SQLERRM;
            ROLLBACK TO SAVEPOINT price_update_attempt; -- Rollback only the price changes
            RAISE NOTICE 'Rolled back to savepoint "price_update_attempt". Price changes undone, but discontinued status change persists.';
    END;

    COMMIT; -- Commit the entire transaction. Discontinued status change persists.
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Reclassification procedure failed for category %: % - %', p_category_id, SQLSTATE, SQLERRM;
        ROLLBACK; -- Rollback everything if an error happened outside the SAVEPOINT block
        RAISE;
END;
$$;

-- --- Test Cases ---

-- Before: Products 9, 17 are discontinued in category 6 (Meat/Poultry)
SELECT product_id, product_name, category_id, unit_price, discontinued FROM products WHERE product_id IN (9, 17);

-- Test 1: Successful Reclassification and Price Update (Category 6, 10% increase)
-- Product 17: Alice Mutton (price 39). Product 9: Mishi Kobe Niku (price 97)
CALL reclassify_discontinued_products_in_category(6, 10);
SELECT product_id, product_name, category_id, unit_price, discontinued FROM products WHERE product_id IN (9, 17);

-- Restore original state (discontinued=1 for 9, 17; original prices)
UPDATE products SET discontinued = 1 WHERE product_id IN (9, 17);
UPDATE products SET unit_price = 97 WHERE product_id = 9;
UPDATE products SET unit_price = 39 WHERE product_id = 17;


-- Test 2: Reclassification with Price Update Failure (Category 6, -100% increase to force <= 0 price)
-- This relies on the BEFORE UPDATE trigger on products that validates unit_price > 0.
-- Product 17 (price 39), Product 9 (price 97)
CALL reclassify_discontinued_products_in_category(6, -100); -- Will cause calculated price <= 0

-- After: Verify discontinued status changed, but prices are original
SELECT product_id, product_name, category_id, unit_price, discontinued FROM products WHERE product_id IN (9, 17);

-- Clean up: Restore original state (discontinued=1 for 9, 17; original prices)
UPDATE products SET discontinued = 1 WHERE product_id IN (9, 17);
UPDATE products SET unit_price = 97 WHERE product_id = 9;
UPDATE products SET unit_price = 39 WHERE product_id = 17;
```

**Expected Output/Explanation:**
*   **Test 1 (Success):** `NOTICE: Re-enabled 2 products in category 6.`, `NOTICE: Savepoint "price_update_attempt" set.`, `NOTICE: Price update attempt successful...`, `COMMIT` will be called. Products 9 and 17 will have `discontinued = 0` and their `unit_price` increased by 10%.
*   **Test 2 (Price Update Failure):** `NOTICE: Re-enabled 2 products in category 6.`, `NOTICE: Savepoint "price_update_attempt" set.`, then `NOTICE: Price update failed...`, then `NOTICE: Rolled back to savepoint "price_update_attempt"...`, `COMMIT` will be called. Products 9 and 17 will have `discontinued = 0`, but their `unit_price` will remain unchanged. This demonstrates the partial rollback.

**Learning Point:**
This task demonstrates advanced transaction control with `SAVEPOINT`, `ROLLBACK TO SAVEPOINT`, and `RELEASE SAVEPOINT`.
*   It shows how to perform a series of operations where some parts can be individually rolled back while the main transaction (and earlier changes within it) persists.
*   This pattern is crucial for complex business processes where partial failures are acceptable or need specific handling without losing all prior work.
*   The nested `BEGIN...EXCEPTION...END` block around the problematic part is key for capturing the error and performing the `ROLLBACK TO SAVEPOINT`.

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **Refine `process_new_order`:** Add `OUT` parameters to `process_new_order` to return the `new_order_id` and a `status_message` (e.g., 'SUCCESS', 'FAILURE: [Reason]').
2.  **Credit Check in Order Processing:** Integrate `process_new_order` with a customer credit check. Before allowing the order, calculate the order's total value (use logic from the `get_full_order_value` function you might have from previous labs or calculate it here) and ensure `customer.credit_limit` is sufficient. If not, `RAISE EXCEPTION` and `ROLLBACK`.
3.  **Customer Deactivation Procedure:** Create a procedure `deactivate_customer(p_customer_id)` that sets `customer.active = FALSE` (you'd need to add this column), `ROLLBACK`s any outstanding `orders` for that customer (by setting `orders.status = 'CANCELLED'`), and logs the deactivation. This should be a single transactional unit.
4.  **Batch Processing Procedure:** Create a procedure `process_daily_inventory_adjustments()` that reads from a `StagingInventoryAdjustments` table (create it). For each row, it updates `products.units_in_stock`. If an update fails for any reason, it logs the error but *continues* processing other adjustments, and `COMMIT`s the successful ones at the end. (This implies `COMMIT`ing *inside* the loop for each successful item, which is a specific pattern for batch processing, or using `SAVEPOINT`s for each item.)

**Further Exploration:**
*   **Performance of Procedures:** Discuss how procedures reduce network round-trips and can benefit from cached execution plans.
*   **Permissions on Procedures:** Learn how to `GRANT EXECUTE ON PROCEDURE` to specific roles/users without granting direct table access, enhancing security.
*   **Nesting Procedures:** Explore calling one procedure from within another.
*   **Trigger vs. Procedure:** Revisit the scenarios where a trigger is appropriate (automatic response to DML) versus a procedure (explicitly invoked, controls transactions).
*   **DRY Run for Transactions:** Always explain to new users that for procedures with `COMMIT`/`ROLLBACK`, it's vital to test them in a development environment before deploying to production, as accidental commits/rollbacks can have serious data integrity consequences.

---

### **Conclusion:**

In this extensive lab, you have gained mastery over PostgreSQL `STORED PROCEDURES` and their critical role in transaction management. You successfully implemented procedures to perform multi-step business operations, utilize `IN`/`OUT` parameters, and execute robust error handling. Most importantly, you gained hands-on experience with explicit `COMMIT`, `ROLLBACK`, and `SAVEPOINT` commands, enabling you to build atomic and resilient transactional workflows. These skills are fundamental for developing enterprise-grade database applications that demand data consistency and reliability, making you well-equipped to tackle complex data challenges in a real-world environment like Northwind.