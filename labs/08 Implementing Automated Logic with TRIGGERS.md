### Implementing Automated Logic with `TRIGGERS`

**Objective:**
This lab provides a detailed, hands-on exploration of PostgreSQL `TRIGGERS`. You will learn to create trigger functions using PL/pgSQL, define triggers for various DML events (`INSERT`, `UPDATE`, `DELETE`), understand the nuances of `BEFORE` vs. `AFTER`, `FOR EACH ROW` vs. `FOR EACH STATEMENT`, utilize `OLD` and `NEW` pseudo-records, implement conditional trigger firing with `WHEN`, and handle exceptions. These concepts will be applied to enhance data integrity, implement business rules, and maintain audit trails within the Northwind database.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
    *   *(If not installed, please refer to the installation instructions from the previous interaction or a reliable source for Northwind PostgreSQL setup.)*
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line, DBeaver, VS Code with PostgreSQL extension).
4.  **Basic SQL Knowledge:** Familiarity with fundamental SQL syntax (`SELECT`, `FROM`, `WHERE`, `JOIN`, `INSERT`, `UPDATE`, `DELETE`).
5.  **Basic PL/pgSQL Knowledge:** Understanding of `DECLARE`, `BEGIN...END`, `SELECT INTO`, `IF` statements. (Covered in previous PL/pgSQL lab).

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run a simple query to confirm connectivity and data presence (e.g., `SELECT count(*) FROM customers;`). You should see a count reflecting the number of customers.
3.  **Enable PL/pgSQL:** While generally enabled by default, ensure PL/pgSQL is active. (Run `CREATE EXTENSION IF NOT EXISTS plpgsql;` if you encounter language-related errors).

---

### **Lab Exercises:**

**Introduction to Tables Used:**
*   `products`: Will be used for validating new products and inventory management.
*   `employees`: Will be used to enforce reporting structure rules and log title changes.
*   `order_details`: Central to inventory adjustments during order processing.
*   `customers`: Will be used for archiving deleted records.
*   `ProductPriceAudit` (new temporary table): To log price changes.
*   `EmployeeChangesLog` (new temporary table): To audit employee data modifications.
*   `InventoryChangeLog` (new temporary table): To track all inventory adjustments by triggers.
*   `CustomerArchive` (new temporary table): To store deleted customer records.

---

#### **Trigger Key Concepts: `OLD` and `NEW` Pseudo-Records**
Within a trigger function, PostgreSQL provides special **pseudo-records** that represent the row data during the DML operation:
*   **`NEW`**: Refers to the new row data **after** an `INSERT` or **after** an `UPDATE` (the new values).
*   **`OLD`**: Refers to the old row data **before** an `UPDATE` or **before** a `DELETE` (the original values).
*   `NEW` is `NULL` for `DELETE` triggers.
*   `OLD` is `NULL` for `INSERT` triggers.
*   For `BEFORE` triggers, you can modify `NEW` to change the data that is actually inserted/updated.

---

#### **Task 1: `BEFORE INSERT OR UPDATE` Trigger (Validation & Defaulting)**

**Description:**
Northwind needs to ensure data integrity for its products:
1.  Any new or updated product `unit_price` must be strictly positive (`> 0`). If not, the operation should be prevented.
2.  If a new product is inserted with `reorder_level` as `NULL` or `0`, it should automatically default to `10`.

**Steps:**
1.  Create a PL/pgSQL trigger function to implement the validation and defaulting logic.
2.  Create a `BEFORE INSERT OR UPDATE FOR EACH ROW` trigger on the `products` table, linking it to the function.

**SQL Query:**

```sql
-- 1. Create the trigger function
CREATE OR REPLACE FUNCTION check_product_price_and_reorder_level()
RETURNS TRIGGER AS $$
BEGIN
    -- Validation: Check unit_price
    IF NEW.unit_price <= 0 THEN
        RAISE EXCEPTION 'Product unit price must be a positive value.';
    END IF;

    -- Defaulting: Set reorder_level if NULL or 0
    IF NEW.reorder_level IS NULL OR NEW.reorder_level = 0 THEN
        NEW.reorder_level := 10;
    END IF;

    RETURN NEW; -- For BEFORE triggers, always return NEW (possibly modified)
END;
$$ LANGUAGE plpgsql;

-- 2. Create the trigger
DROP TRIGGER IF EXISTS trg_products_validation ON products;
CREATE TRIGGER trg_products_validation
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION check_product_price_and_reorder_level();

-- --- Test Cases ---
-- Test 1: Valid INSERT
INSERT INTO products (product_id, product_name, supplier_id, category_id, discontinued, unit_price, units_in_stock, reorder_level)
VALUES (999, 'Test Product A', 1, 1, 0, 15.00, 100, 5);
SELECT product_id, product_name, unit_price, reorder_level FROM products WHERE product_id = 999;
DELETE FROM products WHERE product_id = 999;

-- Test 2: INSERT with unit_price = 0 (should FAIL)
INSERT INTO products (product_id, product_name, supplier_id, category_id, discontinued, unit_price, units_in_stock)
VALUES (1000, 'Test Product B', 1, 1, 0, 0.00, 50);
-- This INSERT should throw an exception. The product should NOT be inserted.
SELECT product_id, product_name FROM products WHERE product_id = 1000;

-- Test 3: INSERT with reorder_level = NULL (should default to 10)
INSERT INTO products (product_id, product_name, supplier_id, category_id, discontinued, unit_price, units_in_stock, reorder_level)
VALUES (1001, 'Test Product C', 1, 1, 0, 25.00, 75, NULL);
SELECT product_id, product_name, unit_price, reorder_level FROM products WHERE product_id = 1001;
DELETE FROM products WHERE product_id = 1001;

-- Test 4: UPDATE an existing product's price to <= 0 (should FAIL)
SELECT product_id, product_name, unit_price FROM products WHERE product_id = 1;
UPDATE products SET unit_price = -5.00 WHERE product_id = 1;
-- This UPDATE should throw an exception. The unit_price should NOT change.
SELECT product_id, product_name, unit_price FROM products WHERE product_id = 1;

-- Test 5: UPDATE an existing product's reorder_level to 0 (should default to 10)
UPDATE products SET reorder_level = 0 WHERE product_id = 1;
SELECT product_id, product_name, reorder_level FROM products WHERE product_id = 1;
-- Reset unit_price and reorder_level for product 1 for subsequent lab tasks if needed
UPDATE products SET unit_price = 18.00, reorder_level = 10 WHERE product_id = 1;
```

**Expected Output/Explanation:**
*   Test 1 & 3: Successful inserts, with product 1001's `reorder_level` defaulting to 10.
*   Test 2 & 4: Will result in `ERROR: Product unit price must be a positive value.`, preventing the operation.
*   Test 5: Product 1's `reorder_level` will become 10, despite setting it to 0.

**Learning Point:**
This task demonstrates a `BEFORE` trigger operating on both `INSERT` and `UPDATE` events. Key takeaways include:
*   Using `RAISE EXCEPTION` within a `BEFORE` trigger function to halt the operation and signal an error.
*   Modifying `NEW` pseudo-record properties (e.g., `NEW.reorder_level := 10`) to alter the data *before* it is written to the table.
*   The `RETURN NEW;` statement is crucial for `BEFORE` row-level triggers to pass the (potentially modified) row to the next stage of processing.

---

#### **Task 2: `BEFORE INSERT OR UPDATE` Trigger (Complex Validation & Auditing)**

**Description:**
Northwind enforces a strict hierarchy:
1.  An employee cannot report to themselves (their `reports_to` cannot be their own `employee_id`).
2.  Any change to an employee's `title` should be logged in an `EmployeeChangesLog` table.

**Steps:**
1.  Create the temporary `EmployeeChangesLog` table.
2.  Create a PL/pgSQL trigger function to implement both validation and logging.
3.  Create a `BEFORE INSERT OR UPDATE FOR EACH ROW` trigger on `employees`.

**SQL Query:**

```sql
-- 1. Create the temporary audit table
DROP TABLE IF EXISTS EmployeeChangesLog CASCADE;
CREATE TEMPORARY TABLE EmployeeChangesLog (
    log_id SERIAL PRIMARY KEY,
    employee_id SMALLINT,
    old_title CHARACTER VARYING(30),
    new_title CHARACTER VARYING(30),
    change_timestamp TIMESTAMP DEFAULT NOW(),
    operation_type TEXT
);

-- 2. Create the trigger function
CREATE OR REPLACE FUNCTION enforce_employee_rules_and_log()
RETURNS TRIGGER AS $$
BEGIN
    -- Validation: Employee cannot report to themselves
    IF NEW.reports_to = NEW.employee_id THEN
        RAISE EXCEPTION 'An employee cannot report to themselves (Employee ID: %).', NEW.employee_id;
    END IF;

    -- Auditing: Log title changes (only for UPDATE operations, and only if title actually changed)
    IF TG_OP = 'UPDATE' AND OLD.title IS DISTINCT FROM NEW.title THEN
        INSERT INTO EmployeeChangesLog (employee_id, old_title, new_title, operation_type)
        VALUES (OLD.employee_id, OLD.title, NEW.title, 'TITLE_CHANGE');
    ELSIF TG_OP = 'INSERT' THEN
        -- Log initial title on insert, or skip as per business rule.
        -- For this lab, let's log title on any insert as well.
        INSERT INTO EmployeeChangesLog (employee_id, old_title, new_title, operation_type)
        VALUES (NEW.employee_id, NULL, NEW.title, 'NEW_EMPLOYEE_TITLE');
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 3. Create the trigger
DROP TRIGGER IF EXISTS trg_employee_rules_and_log ON employees;
CREATE TRIGGER trg_employee_rules_and_log
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION enforce_employee_rules_and_log();

-- --- Test Cases ---
-- Test 1: Valid UPDATE - Change title for employee 1
SELECT employee_id, first_name, last_name, title FROM employees WHERE employee_id = 1;
UPDATE employees SET title = 'Senior Sales Representative' WHERE employee_id = 1;
-- Verify log
SELECT * FROM EmployeeChangesLog WHERE employee_id = 1;
-- Reset title
UPDATE employees SET title = 'Sales Representative' WHERE employee_id = 1;

-- Test 2: Valid INSERT - New employee with a title
INSERT INTO employees (employee_id, last_name, first_name, title, birth_date, hire_date)
VALUES (10, 'Smith', 'John', 'Junior Sales Rep', '1990-05-10', '2023-01-01');
SELECT * FROM EmployeeChangesLog WHERE employee_id = 10;
DELETE FROM employees WHERE employee_id = 10;

-- Test 3: Invalid UPDATE - Employee reports to themselves (should FAIL)
-- Andrew Fuller (employee_id 2) reports to NULL initially
SELECT employee_id, first_name, last_name, reports_to FROM employees WHERE employee_id = 2;
UPDATE employees SET reports_to = 2 WHERE employee_id = 2;
-- This UPDATE should throw an exception. reports_to should NOT change.
SELECT employee_id, first_name, last_name, reports_to FROM employees WHERE employee_id = 2;

-- Test 4: Invalid INSERT - New employee attempts to report to themselves (should FAIL)
INSERT INTO employees (employee_id, last_name, first_name, title, birth_date, hire_date, reports_to)
VALUES (11, 'Doe', 'Jane', 'New Role', '1985-01-01', '2023-02-01', 11);
-- This INSERT should throw an exception. The employee should NOT be inserted.
SELECT employee_id FROM employees WHERE employee_id = 11;
```

**Expected Output/Explanation:**
*   Test 1 & 2: Successful operations, with corresponding entries in `EmployeeChangesLog`. `TG_OP` determines the `operation_type` in the log.
*   Test 3 & 4: Will result in `ERROR: An employee cannot report to themselves...`, preventing the operation.

**Learning Point:**
This task builds on `BEFORE` triggers and introduces:
*   More complex `IF` conditions for validation.
*   Accessing both `OLD` (original row state) and `NEW` (proposed new row state) pseudo-records within `UPDATE` operations for comparison and auditing.
*   Using `TG_OP` (a special variable provided in trigger functions) to determine the type of DML operation (`'INSERT'`, `'UPDATE'`, `'DELETE'`) that fired the trigger, allowing different logic paths.

---

#### **Task 3: `AFTER INSERT, UPDATE, DELETE` Trigger (Inventory Management)**

**Description:**
Northwind needs real-time inventory adjustments based on `order_details` changes.
1.  When new items are added to an order (`INSERT` on `order_details`), decrease `units_in_stock` of the respective `product`.
2.  When the `quantity` of an `order_detail` changes (`UPDATE`), adjust `units_in_stock` accordingly (subtract new quantity, add old quantity).
3.  When an `order_detail` is removed (`DELETE`), increase `units_in_stock` of the respective `product`.

**Steps:**
1.  Create a temporary `InventoryChangeLog` table to audit trigger actions.
2.  Create a PL/pgSQL trigger function to handle all three DML events for inventory.
3.  Create an `AFTER INSERT OR UPDATE OR DELETE FOR EACH ROW` trigger on `order_details`.

**SQL Query:**

```sql
-- 1. Create a temporary log table for inventory changes made by trigger
DROP TABLE IF EXISTS InventoryChangeLog CASCADE;
CREATE TEMPORARY TABLE InventoryChangeLog (
    log_id SERIAL PRIMARY KEY,
    product_id SMALLINT,
    quantity_change SMALLINT,
    operation_type TEXT,
    order_id SMALLINT,
    log_timestamp TIMESTAMP DEFAULT NOW()
);

-- 2. Create the trigger function
CREATE OR REPLACE FUNCTION adjust_product_inventory()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        -- Decrease stock for new order_detail entry
        UPDATE products
        SET units_in_stock = units_in_stock - NEW.quantity
        WHERE product_id = NEW.product_id;
        INSERT INTO InventoryChangeLog (product_id, quantity_change, operation_type, order_id)
        VALUES (NEW.product_id, -NEW.quantity, 'ORDER_ITEM_ADD', NEW.order_id);

    ELSIF TG_OP = 'UPDATE' THEN
        -- Adjust stock based on quantity change
        -- Stock changes by (OLD.quantity - NEW.quantity)
        UPDATE products
        SET units_in_stock = units_in_stock + OLD.quantity - NEW.quantity
        WHERE product_id = NEW.product_id;
        INSERT INTO InventoryChangeLog (product_id, quantity_change, operation_type, order_id)
        VALUES (NEW.product_id, OLD.quantity - NEW.quantity, 'ORDER_ITEM_QTY_UPDATE', NEW.order_id);

    ELSIF TG_OP = 'DELETE' THEN
        -- Increase stock for deleted order_detail entry
        UPDATE products
        SET units_in_stock = units_in_stock + OLD.quantity
        WHERE product_id = OLD.product_id;
        INSERT INTO InventoryChangeLog (product_id, quantity_change, operation_type, order_id)
        VALUES (OLD.product_id, OLD.quantity, 'ORDER_ITEM_REMOVE', OLD.order_id);
    END IF;

    RETURN NULL; -- AFTER triggers return NULL
END;
$$ LANGUAGE plpgsql;

-- 3. Create the trigger
DROP TRIGGER IF EXISTS trg_adjust_inventory ON order_details;
CREATE TRIGGER trg_adjust_inventory
AFTER INSERT OR UPDATE OR DELETE ON order_details
FOR EACH ROW
EXECUTE FUNCTION adjust_product_inventory();

-- --- Test Cases ---
-- Get initial stock for product ID 11 (Queso Cabrales)
SELECT product_name, units_in_stock FROM products WHERE product_id = 11;
-- Clean up log for product 11
DELETE FROM InventoryChangeLog WHERE product_id = 11;

-- Test 1: INSERT a new order_detail (order 10248, product 11, quantity 5)
-- Note: You might need to temporarily restore product_id 11 in order 10248 first if deleted by previous labs
-- If it already exists, use a different product_id that's not in 10248, or update an existing one.
-- For this test, let's insert a NEW order_detail entry for product 11.
-- Need to ensure this combination is not already in order_details for order 10248.
-- Let's use product 74 (Longlife Tofu) for order 10248, if it's not present (initial quantity in db for order 10248, product 74 is 21).
-- First, verify product 74 initial stock
SELECT product_name, units_in_stock FROM products WHERE product_id = 74; -- Should be 4
-- Make sure it's not in order 10248, or delete it if it is
-- DELETE FROM order_details WHERE order_id = 10248 AND product_id = 74;
-- INSERT new line item
INSERT INTO order_details (order_id, product_id, unit_price, quantity, discount)
VALUES (10248, 74, 10.00, 5, 0); -- 5 quantity

-- Verify product stock and log
SELECT product_name, units_in_stock FROM products WHERE product_id = 74; -- Should decrease by 5
SELECT * FROM InventoryChangeLog WHERE product_id = 74 AND operation_type = 'ORDER_ITEM_ADD';

-- Test 2: UPDATE quantity of an existing order_detail (order 10248, product 11)
-- Initial quantity for product 11 in order 10248 is 12 (from script)
SELECT quantity FROM order_details WHERE order_id = 10248 AND product_id = 11;
SELECT units_in_stock FROM products WHERE product_id = 11; -- Initial stock 22
UPDATE order_details SET quantity = 8 WHERE order_id = 10248 AND product_id = 11; -- Quantity changed from 12 to 8
-- Stock should increase by 4 (12 - 8)
SELECT product_name, units_in_stock FROM products WHERE product_id = 11; -- Should be 26
SELECT * FROM InventoryChangeLog WHERE product_id = 11 AND operation_type = 'ORDER_ITEM_QTY_UPDATE';

-- Test 3: DELETE an order_detail (order 10248, product 42)
-- Initial quantity for product 42 in order 10248 is 10
SELECT quantity FROM order_details WHERE order_id = 10248 AND product_id = 42;
SELECT units_in_stock FROM products WHERE product_id = 42; -- Initial stock 26
DELETE FROM order_details WHERE order_id = 10248 AND product_id = 42;
-- Stock should increase by 10
SELECT product_name, units_in_stock FROM products WHERE product_id = 42; -- Should be 36
SELECT * FROM InventoryChangeLog WHERE product_id = 42 AND operation_type = 'ORDER_ITEM_REMOVE';

-- Clean up and reset for further lab runs (re-insert deleted rows)
-- Reinsert product 74 for order 10248
DELETE FROM order_details WHERE order_id = 10248 AND product_id = 74; -- This delete will trigger adjust_product_inventory to adjust stock.
INSERT INTO order_details (order_id, product_id, unit_price, quantity, discount)
VALUES (10248, 74, 10.00, 5, 0); -- This insert will trigger adjust_product_inventory to adjust stock.
-- Reinsert product 42 for order 10248
DELETE FROM order_details WHERE order_id = 10248 AND product_id = 42; -- This delete will trigger adjust_product_inventory to adjust stock.
INSERT INTO order_details VALUES (10248, 42, 9.80000019, 10, 0); -- This insert will trigger adjust_product_inventory to adjust stock.
-- Reset product 11 quantity
UPDATE order_details SET quantity = 12 WHERE order_id = 10248 AND product_id = 11;
```

**Expected Output/Explanation:**
*   Each `INSERT`, `UPDATE`, `DELETE` operation on `order_details` will show `INSERT 0 1`, `UPDATE 1`, or `DELETE 1` respectively.
*   The `units_in_stock` for the affected products will change correctly (e.g., for `INSERT`, stock decreases by `NEW.quantity`; for `UPDATE`, stock adjusts by `OLD.quantity - NEW.quantity`; for `DELETE`, stock increases by `OLD.quantity`).
*   The `InventoryChangeLog` will accurately record each adjustment made by the trigger.

**Learning Point:**
This task demonstrates `AFTER` triggers, which execute *after* the DML operation on the table.
*   They are ideal for auditing, logging, or performing cascading actions that don't need to prevent the original operation.
*   `AFTER` triggers always `RETURN NULL;`.
*   This trigger combines logic for `INSERT`, `UPDATE`, and `DELETE` events within a single function using `TG_OP`.
*   It highlights the use of `OLD` and `NEW` pseudo-records in different contexts (`INSERT` uses `NEW`, `DELETE` uses `OLD`, `UPDATE` uses both).

---

#### **Task 4: `AFTER DELETE` Trigger (Archiving Data)**

**Description:**
Northwind wants to archive customer records instead of permanently deleting them. When a customer is deleted from the `customers` table, their details should be moved to a `CustomerArchive` table.

**Steps:**
1.  Create the `CustomerArchive` table with a similar structure to `customers`.
2.  Create a PL/pgSQL trigger function to insert the deleted row into the archive.
3.  Create an `AFTER DELETE FOR EACH ROW` trigger on `customers`.

**SQL Query:**

```sql
-- 1. Create the temporary archive table (matching customers table structure)
DROP TABLE IF EXISTS CustomerArchive CASCADE;
CREATE TEMPORARY TABLE CustomerArchive (LIKE customers INCLUDING ALL);
ALTER TABLE CustomerArchive ADD COLUMN archived_at TIMESTAMP DEFAULT NOW();

-- 2. Create the trigger function
CREATE OR REPLACE FUNCTION archive_deleted_customer()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO CustomerArchive
    SELECT OLD.*, NOW(); -- Insert the old row data into the archive table, add timestamp
    RETURN NULL; -- AFTER triggers return NULL
END;
$$ LANGUAGE plpgsql;

-- 3. Create the trigger
DROP TRIGGER IF EXISTS trg_archive_customer ON customers;
CREATE TRIGGER trg_archive_customer
AFTER DELETE ON customers
FOR EACH ROW
EXECUTE FUNCTION archive_deleted_customer();

-- --- Test Cases ---
-- Test 1: Delete a customer (e.g., 'ALFKI')
-- Check original customer existence
SELECT * FROM customers WHERE customer_id = 'ALFKI';
SELECT * FROM CustomerArchive WHERE customer_id = 'ALFKI'; -- Should be empty

-- Important: To delete 'ALFKI', you must first delete dependent records (orders, order_details)
-- This is a real-world constraint, not a trigger issue.
-- For a lab, we'll try to delete a customer with few/no orders or temporarily disable FKs (not recommended in prod).
-- Let's try customer 'WOLZA' (Wolski Zajazd), who often has fewer direct dependencies in small datasets.
-- If FK error, you might need to manually delete associated order_details and orders first.
-- Example for WOLZA:
-- DELETE FROM order_details WHERE order_id IN (SELECT order_id FROM orders WHERE customer_id = 'WOLZA');
-- DELETE FROM orders WHERE customer_id = 'WOLZA';
SELECT * FROM customers WHERE customer_id = 'WOLZA';
SELECT * FROM CustomerArchive WHERE customer_id = 'WOLZA';

DELETE FROM customers WHERE customer_id = 'WOLZA';

-- Verify: Customer 'WOLZA' should be gone from 'customers' but present in 'CustomerArchive'
SELECT * FROM customers WHERE customer_id = 'WOLZA'; -- Should return 0 rows
SELECT * FROM CustomerArchive WHERE customer_id = 'WOLZA'; -- Should return 1 row with archived data

-- Clean up: Re-insert WOLZA if you want it back for other labs
-- INSERT INTO customers VALUES ('WOLZA', 'Wolski  Zajazd', 'Zbyszek Piestrzeniewicz', 'Owner', 'ul. Filtrowa 68', 'Warszawa', NULL, '01-012', 'Poland', '(26) 642-7012', '(26) 642-7012');
```

**Expected Output/Explanation:**
*   The `DELETE` operation will succeed (assuming no FK constraints block it).
*   The `customers` table will no longer contain the deleted customer.
*   The `CustomerArchive` table will contain a full copy of the deleted customer's record, along with the `archived_at` timestamp.

**Learning Point:**
This task demonstrates an `AFTER DELETE` trigger for archiving data. The `OLD` pseudo-record is essential here, as it provides access to the data of the row that was just deleted. This pattern is commonly used for auditing, disaster recovery, or soft-deleting where the original table is cleared.

---

#### **Task 5: `WHEN` Clause (Conditional Trigger Firing)**

**Description:**
Northwind wants to implement a special logging rule: every time a product's name is updated, log it into `ProductNameChangeLog`, but *only if* the new product name contains the word "Coffee" (case-insensitive) or "Tea" (case-insensitive).

**Steps:**
1.  Create a temporary `ProductNameChangeLog` table.
2.  Create a PL/pgSQL trigger function to perform the logging.
3.  Create an `AFTER UPDATE FOR EACH ROW` trigger on `products` with a `WHEN` clause.

**SQL Query:**

```sql
-- 1. Create temporary log table
DROP TABLE IF EXISTS ProductNameChangeLog CASCADE;
CREATE TEMPORARY TABLE ProductNameChangeLog (
    log_id SERIAL PRIMARY KEY,
    product_id SMALLINT,
    old_product_name VARCHAR(40),
    new_product_name VARCHAR(40),
    change_timestamp TIMESTAMP DEFAULT NOW()
);

-- 2. Create the trigger function
CREATE OR REPLACE FUNCTION log_specific_product_name_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO ProductNameChangeLog (product_id, old_product_name, new_product_name)
    VALUES (OLD.product_id, OLD.product_name, NEW.product_name);
    RETURN NULL; -- AFTER triggers return NULL
END;
$$ LANGUAGE plpgsql;

-- 3. Create the trigger with a WHEN clause
DROP TRIGGER IF EXISTS trg_log_name_changes_conditional ON products;
CREATE TRIGGER trg_log_name_changes_conditional
AFTER UPDATE OF product_name ON products -- Only fire if product_name column is updated
FOR EACH ROW
WHEN (NEW.product_name ILIKE '%Coffee%' OR NEW.product_name ILIKE '%Tea%')
EXECUTE FUNCTION log_specific_product_name_changes();

-- --- Test Cases ---
-- Test 1: Update 'Chai' (product_id 1) to 'Chai Tea' (should log)
SELECT product_name FROM products WHERE product_id = 1;
UPDATE products SET product_name = 'Chai Tea' WHERE product_id = 1;
SELECT * FROM ProductNameChangeLog WHERE product_id = 1;
-- Reset name
UPDATE products SET product_name = 'Chai' WHERE product_id = 1; -- This update should NOT log (new name 'Chai' doesn't match WHEN)

-- Test 2: Update 'Ipoh Coffee' (product_id 43) to 'Premium Ipoh Coffee' (should log)
SELECT product_name FROM products WHERE product_id = 43;
UPDATE products SET product_name = 'Premium Ipoh Coffee' WHERE product_id = 43;
SELECT * FROM ProductNameChangeLog WHERE product_id = 43;
-- Reset name
UPDATE products SET product_name = 'Ipoh Coffee' WHERE product_id = 43;

-- Test 3: Update 'Chang' (product_id 2) to 'Chang Beer' (should NOT log)
SELECT product_name FROM products WHERE product_id = 2;
UPDATE products SET product_name = 'Chang Beer' WHERE product_id = 2;
SELECT * FROM ProductNameChangeLog WHERE product_id = 2; -- Should be empty
-- Reset name
UPDATE products SET product_name = 'Chang' WHERE product_id = 2;
```

**Expected Output/Explanation:**
*   Test 1 & 2: Successful updates, with entries in `ProductNameChangeLog` only when the `WHEN` condition is met.
*   Test 3: Successful update, but *no* entry in `ProductNameChangeLog` because "Beer" does not contain "Coffee" or "Tea".

**Learning Point:**
This task introduces the `WHEN` clause in `CREATE TRIGGER`.
*   The `WHEN` clause allows a trigger to fire conditionally, based on the data in the `OLD` and/or `NEW` pseudo-records.
*   The trigger function is only executed *if* the `WHEN` condition evaluates to `TRUE`. This saves execution time for the function when the condition is not met.
*   Using `AFTER UPDATE OF column_name` specifies that the trigger only fires if that specific column is part of the `UPDATE` statement.

---

#### **Task 6: Disabling and Enabling Triggers**

**Description:**
Sometimes, for bulk data loading or maintenance, you might need to temporarily disable triggers on a table. Practice disabling and enabling one of your created triggers.

**Steps:**
1.  Disable the `trg_products_validation` trigger (from Task 1).
2.  Perform an `INSERT` that *would normally fail* due to validation rules. Observe it succeeding.
3.  Re-enable the trigger.
4.  Perform the same `INSERT` again. Observe it failing this time.

**SQL Query:**

```sql
-- Test 1: Try to insert an invalid product (unit_price = 0), should FAIL
INSERT INTO products (product_id, product_name, supplier_id, category_id, discontinued, unit_price, units_in_stock)
VALUES (1002, 'Temp Product Invalid', 1, 1, 0, 0.00, 10);
-- Should give an error: "Product unit price must be a positive value."

-- Disable the trigger
ALTER TABLE products DISABLE TRIGGER trg_products_validation;
SELECT tgname, tgenabled FROM pg_trigger WHERE tgrelid = 'products'::regclass AND tgname = 'trg_products_validation';

-- Test 2: Try to insert the same invalid product again (should SUCCEED now)
INSERT INTO products (product_id, product_name, supplier_id, category_id, discontinued, unit_price, units_in_stock)
VALUES (1002, 'Temp Product Invalid', 1, 1, 0, 0.00, 10);
SELECT product_id, product_name, unit_price FROM products WHERE product_id = 1002;
DELETE FROM products WHERE product_id = 1002; -- Clean up

-- Re-enable the trigger
ALTER TABLE products ENABLE TRIGGER trg_products_validation;
SELECT tgname, tgenabled FROM pg_trigger WHERE tgrelid = 'products'::regclass AND tgname = 'trg_products_validation';

-- Test 3: Try to insert the invalid product again (should FAIL again)
INSERT INTO products (product_id, product_name, supplier_id, category_id, discontinued, unit_price, units_in_stock)
VALUES (1002, 'Temp Product Invalid', 1, 1, 0, 0.00, 10);
-- Should give an error again.
```

**Expected Output/Explanation:**
*   Test 1 & 3: Will result in an error, preventing insertion.
*   Test 2: Will successfully insert the product with `unit_price = 0`, demonstrating that the trigger was disabled. The `SELECT` query will show the new product.

**Learning Point:**
This task demonstrates `ALTER TABLE ... DISABLE TRIGGER` and `ENABLE TRIGGER`.
*   These commands are useful for temporarily bypassing trigger logic, often for bulk data loads or maintenance tasks where immediate validation or side effects are not desired.
*   You can also disable/enable `ALL TRIGGERS` on a table.
*   `pg_trigger` system catalog can be queried to check the status of triggers.

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **Statement-Level Trigger:** Create a simple `AFTER INSERT ON order_details FOR EACH STATEMENT` trigger that logs a single message "New order details added for order [order_id]" when *any* new order details are inserted, rather than per row.
    *   *Hint: Statement-level triggers do not have `OLD` or `NEW` pseudo-records.* You would typically iterate using a query or rely on `TG_TABLE_NAME` and `TG_OP`.
2.  **Combined Validation (Cross-Table):** Enhance `adjust_product_inventory` to prevent an `order_details` insert if `NEW.quantity` for `NEW.product_id` would result in `units_in_stock` going below zero. This requires changing the trigger to `BEFORE INSERT OR UPDATE` and doing the stock check and `RAISE EXCEPTION` there.
3.  **Complex Archiving:** Modify the `archive_deleted_customer` trigger to *also* move all associated `orders` and `order_details` records to separate archive tables. This would require an `AFTER DELETE` trigger on `customers` and a more complex PL/pgSQL function.
4.  **Clean up:** Remember to `DROP TRIGGER` for all triggers you created (e.g., `DROP TRIGGER trg_products_validation ON products;`), `DROP FUNCTION` for all trigger functions, and `DROP TABLE` for all temporary tables (e.g., `DROP TABLE EmployeeChangesLog;`) to clean up your database.

**Further Exploration:**
*   **Triggers vs. Rules:** Revisit the comparison. When would you absolutely use a trigger over a rule (e.g., for complex procedural logic, auditing, transactions)?
*   **`FOR EACH STATEMENT` Triggers:** Understand their primary use cases (e.g., security logging, aggregate checks, permission logging).
*   **Trigger Firing Order:** If multiple `BEFORE` triggers are on the same table/event, or `BEFORE` and `AFTER` triggers, how do they interact?
*   **`CONSTRAINT TRIGGER`:** Learn about a special type of `AFTER` trigger that fires at the end of the transaction, allowing for more complex data integrity checks.
*   **Event Triggers:** Explore triggers that fire on DDL events (e.g., `CREATE TABLE`, `DROP FUNCTION`) at the database level.

---

### **Conclusion:**

In this comprehensive lab, you gained deep practical experience with PostgreSQL `TRIGGERS`. You mastered defining trigger functions in PL/pgSQL, implementing `BEFORE` and `AFTER` triggers for `INSERT`, `UPDATE`, and `DELETE` operations, and utilizing the `OLD` and `NEW` pseudo-records. You applied these skills to enforce data validation, manage inventory, audit changes, and archive data within the Northwind context, solidifying your ability to build robust and automated database solutions.