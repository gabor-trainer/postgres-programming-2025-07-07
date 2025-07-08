
### Mastering Query Rewriting with `RULES`

**Objective:**
This lab will provide an in-depth, hands-on understanding of PostgreSQL `RULES`. You will learn how to define rules for `INSERT`, `UPDATE`, and `DELETE` operations, using different actions (`DO NOTHING`, `DO ALSO`, `DO INSTEAD`), and apply them to various Northwind scenarios including preventing actions, auditing, data redirection, and enabling DML on views.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
    *   *(If not installed, please refer to the installation instructions from the previous interaction or a reliable source for Northwind PostgreSQL setup.)*
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line, DBeaver, VS Code with PostgreSQL extension).
4.  **Basic SQL Knowledge:** Familiarity with fundamental SQL syntax (`SELECT`, `FROM`, `WHERE`, `JOIN`, `INSERT`, `UPDATE`, `DELETE`).

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run a simple query to confirm connectivity and data presence (e.g., `SELECT count(*) FROM customers;`). You should see a count reflecting the number of customers.

---

### **Lab Exercises:**

**Introduction to Tables Used:**
*   `categories`: Will be used to prevent deletion of critical categories.
*   `products`: Will be used for auditing price changes and soft-deletes.
*   `orders`: Will be used for standardizing data on insert.
*   `customers`: Will be used for view DML exercises.
*   `CustomerProductStats` (new temporary view): To demonstrate DML on views.
*   `ProductPriceAudit` (new temporary table): To log price changes.
*   `SoftDeletedProducts` (new temporary table): To store "soft-deleted" products.

---

#### **Task 1: `ON DELETE` Rule - Preventing Deletion (`DO INSTEAD NOTHING`)**

**Description:**
Northwind wants to prevent accidental deletion of their most important product categories. Create a rule that prevents any attempt to delete the 'Beverages' category (Category ID 1) from the `categories` table. When a `DELETE` command targets this category, the rule should simply do nothing instead of executing the delete.

**SQL Query:**

```sql
-- Ensure the rule doesn't exist from previous runs
DROP RULE IF EXISTS no_delete_beverages ON categories;

CREATE RULE no_delete_beverages AS
ON DELETE TO categories
WHERE OLD.category_id = 1 -- OLD refers to the row being deleted
DO INSTEAD NOTHING;

-- Test Case 1: Try to delete 'Beverages' (should fail silently, pay attenstion to message 'DELETE 0')
DELETE FROM categories WHERE category_id = 1;

-- Test Case 2: Try to delete 'Condiments' (Category ID 2) (should succeed, pay attenstion to message 'DELETE 1')
INSERT INTO categories (category_id, category_name, description, picture)
VALUES (100, 'Condiments2', 'Sweet and savory sauces, relishes, spreads, and seasonings2', '\x')
ON CONFLICT (category_id) DO NOTHING;

DELETE FROM categories WHERE category_id = 100; -- This will remove Condiments temporarily. Remember to re-insert if you need it later.

-- Verify categories table content
SELECT * FROM categories ORDER BY category_id;

-- Clean up (optional, but good for idempotent testing): Re-insert Condiments if it was deleted

```

**Expected Output/Explanation:**
*   `DELETE FROM categories WHERE category_id = 1;` should show `DELETE 0`. The row for 'Beverages' should still exist in the `categories` table.
*   `DELETE FROM categories WHERE category_id = 2;` should show `DELETE 1`. The row for 'Condiments' should be removed.
*   The `SELECT * FROM categories` will confirm the presence of 'Beverages' and the absence (then re-presence after re-insert) of 'Condiments'.

**Learning Point:**
This task demonstrates the `CREATE RULE` syntax, including `ON DELETE TO table`, the `WHERE OLD.column` clause to refer to the old row's data, and the `DO INSTEAD NOTHING` action. `INSTEAD NOTHING` completely overrides the original command, preventing its execution.

---

#### **Task 2: `ON INSERT` Rule - Data Standardization/Modification (`DO INSTEAD INSERT`)**

**Description:**
Northwind often receives orders where the `freight` cost is left blank or set to zero. Create a rule that, whenever a new order is inserted, if its `freight` value is `NULL` or `0`, it is automatically set to a default of `10.00`. The rule should `INSTEAD` insert the modified data.

**SQL Query:**

```sql
-- Ensure the rule doesn't exist from previous runs
DROP RULE IF EXISTS standardize_order_freight ON orders;

CREATE RULE standardize_order_freight AS
ON INSERT TO orders
WHERE NEW.freight IS NULL OR NEW.freight = 0.0
DO INSTEAD
    INSERT INTO orders (
        order_id, customer_id, employee_id, order_date, required_date, shipped_date,
        ship_via, freight, ship_name, ship_address, ship_city, ship_region,
        ship_postal_code, ship_country
    ) VALUES (
        NEW.order_id, NEW.customer_id, NEW.employee_id, NEW.order_date, NEW.required_date, NEW.shipped_date,
        NEW.ship_via, 10.00, NEW.ship_name, NEW.ship_address, NEW.ship_city, NEW.ship_region,
        NEW.ship_postal_code, NEW.ship_country
    );

-- Test Case 1: Insert a new order with NULL freight
INSERT INTO orders (order_id, customer_id, employee_id, order_date, required_date, ship_via)
VALUES (90001, 'ALFKI', 1, '2023-01-01', '2023-01-15', 1);

-- Test Case 2: Insert a new order with freight = 0
INSERT INTO orders (order_id, customer_id, employee_id, order_date, required_date, ship_via, freight)
VALUES (90002, 'ANATR', 2, '2023-01-02', '2023-01-16', 2, 0.0);

-- Test Case 3: Insert a new order with a specified freight (should not be affected by the rule)
INSERT INTO orders (order_id, customer_id, employee_id, order_date, required_date, ship_via, freight)
VALUES (90003, 'ANTON', 3, '2023-01-03', '2023-01-17', 3, 25.50);

-- Verify the inserted orders
SELECT order_id, customer_id, freight FROM orders WHERE order_id >= 90001 AND order_id <= 90003;

-- Clean up
DELETE FROM orders WHERE order_id >= 90001 AND order_id <= 90003;
```

**Expected Output/Explanation:**
*   The `INSERT` statements for 90001 and 90002 will appear to insert with their specified freight (NULL or 0), but the `SELECT` verification will show `10.00` for their `freight` value.
*   The `INSERT` for 90003 will retain its `25.50` freight.

**Learning Point:**
This task demonstrates `DO INSTEAD` with an `INSERT` command. The `NEW` pseudo-record is crucial here, allowing the rule to access the data provided in the original `INSERT` statement and modify specific fields (`NEW.freight` in this case) before the `INSTEAD` action executes.

---

#### **Task 3: `ON UPDATE` Rule - Soft Deletion (`DO INSTEAD UPDATE`)**

**Description:**
Instead of hard-deleting products, Northwind wants to implement a soft-delete mechanism for products. When a `DELETE` command is issued for a product, a rule should intercept it and instead update a new `is_deleted` flag for that product to `TRUE`.

**Steps:**
1.  Add an `is_deleted` column to the `products` table (or create a temporary helper table `SoftDeletedProducts` to store "deleted" product IDs and their state). Let's use a helper table to avoid schema modification on the main `products` table for the lab.
2.  Create a rule on `products` that intercepts `DELETE` operations.

**SQL Query:**

```sql
-- 1. Create a temporary helper table for soft-deleted products
DROP TABLE IF EXISTS SoftDeletedProducts CASCADE;
CREATE TEMPORARY TABLE SoftDeletedProducts (
    product_id SMALLINT PRIMARY KEY,
    deleted_at TIMESTAMP DEFAULT NOW()
);

-- 2. Ensure the rule doesn't exist from previous runs
DROP RULE IF EXISTS soft_delete_product ON products;

CREATE RULE soft_delete_product AS
ON DELETE TO products
WHERE OLD.discontinued = 0 -- Only soft-delete non-discontinued products for realism
DO INSTEAD (
    -- Insert a record into our soft-delete log
    INSERT INTO SoftDeletedProducts (product_id) VALUES (OLD.product_id);

    -- Also, we might want to update the original product's 'discontinued' status,
    -- but that would typically be handled by a separate UPDATE rule or business logic.
    -- For simplicity, this rule just logs the soft-delete.
);

-- Test Case 1: Attempt to delete a non-discontinued product (e.g., 'Chai', product_id 1)
-- Check its current state first
SELECT product_id, product_name, discontinued FROM products WHERE product_id = 1;
SELECT * FROM SoftDeletedProducts WHERE product_id = 1; -- Should be empty

DELETE FROM products WHERE product_id = 1;

-- Verify: Product should still exist in 'products', but a record should be in 'SoftDeletedProducts'
SELECT product_id, product_name, discontinued FROM products WHERE product_id = 1;
SELECT * FROM SoftDeletedProducts WHERE product_id = 1;

-- Test Case 2: Attempt to delete a discontinued product (e.g., 'Chef Anton''s Gumbo Mix', product_id 5)
-- This DELETE would normally succeed unless an FK constraint exists.
-- Our rule explicitly targets discontinued = 0. So this delete should proceed (or fail on FK).
-- We expect our rule to *not* fire for this case if it was already discontinued.
SELECT product_id, product_name, discontinued FROM products WHERE product_id = 5;
DELETE FROM products WHERE product_id = 5; -- This might fail due to FKs from order_details.
-- If it fails due to FK, it's normal. The point is the rule didn't intercept.

-- Clean up SoftDeletedProducts log (optional)
DELETE FROM SoftDeletedProducts WHERE product_id = 1;
```

**Expected Output/Explanation:**
*   `DELETE FROM products WHERE product_id = 1;` should show `DELETE 0`. The product 'Chai' will still be in the `products` table.
*   The `SELECT * FROM SoftDeletedProducts WHERE product_id = 1;` should now show a record for `product_id = 1`, indicating it was soft-deleted.
*   For `product_id = 5`, if it's already discontinued, the `DELETE` command might still fail due to foreign key constraints from `order_details`. The key learning is that the `soft_delete_product` rule, due to its `WHERE OLD.discontinued = 0` condition, *would not* intercept this deletion attempt.

**Learning Point:**
This task demonstrates `DO INSTEAD` with a `DELETE` operation, where the rule effectively changes a `DELETE` into an `INSERT` into a log. It highlights the use of the `OLD` pseudo-record to access details of the row that *would have been* deleted. It also shows how a rule's `WHERE` clause can prevent it from firing.
*Note: Directly replacing a `DELETE` with an `UPDATE` on the same table (e.g., `UPDATE products SET is_deleted = TRUE WHERE product_id = OLD.product_id;`) is also a common soft-delete pattern for rules, though triggers are generally preferred.*

---

#### **Task 4: `ON UPDATE` Rule - Auditing/Logging (`DO ALSO INSERT`)**

**Description:**
Northwind wants to keep a log of all price changes for products. Create a rule that, whenever the `unit_price` of a product in the `products` table is updated, it also inserts a record into a new temporary audit table, `ProductPriceAudit`, detailing the old and new prices.

**Steps:**
1.  Create the temporary `ProductPriceAudit` table.
2.  Create a rule on `products` for `ON UPDATE`.

**SQL Query:**

```sql
-- 1. Create a temporary audit table
DROP TABLE IF EXISTS ProductPriceAudit CASCADE;
CREATE TEMPORARY TABLE ProductPriceAudit (
    audit_id SERIAL PRIMARY KEY,
    product_id SMALLINT,
    old_unit_price REAL,
    new_unit_price REAL,
    change_timestamp TIMESTAMP DEFAULT NOW()
);

-- 2. Ensure the rule doesn't exist from previous runs
DROP RULE IF EXISTS log_price_changes ON products;

CREATE RULE log_price_changes AS
ON UPDATE TO products
WHERE OLD.unit_price IS DISTINCT FROM NEW.unit_price -- Only log if price actually changed
DO ALSO
    INSERT INTO ProductPriceAudit (product_id, old_unit_price, new_unit_price)
    VALUES (NEW.product_id, OLD.unit_price, NEW.unit_price);

-- Test Case 1: Update product 'Chang' (product_id 2) price
-- Check current price
SELECT product_id, product_name, unit_price FROM products WHERE product_id = 2;
SELECT * FROM ProductPriceAudit WHERE product_id = 2; -- Should be empty

UPDATE products SET unit_price = 20.00 WHERE product_id = 2;

-- Verify: Check updated price in products and log in audit table
SELECT product_id, product_name, unit_price FROM products WHERE product_id = 2;
SELECT * FROM ProductPriceAudit WHERE product_id = 2;

-- Test Case 2: Update product 'Aniseed Syrup' (product_id 3) price (should also log)
UPDATE products SET unit_price = 11.50 WHERE product_id = 3;
SELECT * FROM ProductPriceAudit WHERE product_id = 3;

-- Test Case 3: Update product 'Chai' (product_id 1) price to the same value (should NOT log)
UPDATE products SET unit_price = 18.00 WHERE product_id = 1;
SELECT * FROM ProductPriceAudit WHERE product_id = 1; -- Should be empty if 18.00 was original price

-- Clean up
DELETE FROM ProductPriceAudit WHERE product_id IN (1, 2, 3);
```

**Expected Output/Explanation:**
*   `UPDATE products SET unit_price = 20.00 WHERE product_id = 2;` should show `UPDATE 1`.
*   `SELECT * FROM ProductPriceAudit WHERE product_id = 2;` will show a new row detailing the price change for product 2.
*   Similarly for product 3.
*   For product 1, if the unit price was already 18.00, the `UPDATE` will still show `UPDATE 1` (as PostgreSQL still processes the update even if no value changes), but the audit table will *not* have a new entry for product 1 because of the `WHERE OLD.unit_price IS DISTINCT FROM NEW.unit_price` condition in the rule.

**Learning Point:**
This task demonstrates the `DO ALSO` action. This means the original `UPDATE` command executes as normal, *and* the rule's command (an `INSERT` into the audit table) also executes. The `OLD` and `NEW` pseudo-records are essential here to capture both the state before and after the update. `IS DISTINCT FROM` is a NULL-safe comparison operator often useful in rules/triggers.

---

#### **Task 5: DML on Views (`INSTEAD OF INSERT`, `INSTEAD OF UPDATE`, `INSTEAD OF DELETE`)**

**Description:**
Rules are particularly powerful for enabling DML operations on non-updatable views. Create a view that combines customer and product information. Then, create rules to allow `INSERT`, `UPDATE`, and `DELETE` operations on this view, mapping them to DML on the underlying base tables.

**Steps:**
1.  Create a view named `CustomerProductStats` which displays a customer's `company_name`, a `product_name` they ordered, and the `quantity` of that product.
2.  Create an `ON INSERT` rule for `CustomerProductStats` to insert into `orders` and `order_details`.
3.  Create an `ON UPDATE` rule for `CustomerProductStats` to update `order_details` (e.g., `quantity`).
4.  Create an `ON DELETE` rule for `CustomerProductStats` to delete from `order_details`.

**SQL Query:**

```sql
-- 1. Create the non-updatable view
DROP VIEW IF EXISTS CustomerProductStats;
CREATE VIEW CustomerProductStats AS
SELECT
    c.customer_id,
    c.company_name,
    p.product_id,
    p.product_name,
    od.order_id,
    od.quantity,
    od.unit_price,
    od.discount
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
JOIN
    order_details od ON o.order_id = od.order_id
JOIN
    products p ON od.product_id = p.product_id;

-- Verify view (initial data)
SELECT * FROM CustomerProductStats WHERE customer_id = 'ALFKI' LIMIT 5;

-- 2. Create ON INSERT rule for the view
-- This is complex, as it requires inserting into two tables and linking them.
-- For simplicity, let's assume we are inserting a new order_detail for an existing order.
-- The NEW pseudo-record will contain the values from the attempted INSERT on the view.

DROP RULE IF EXISTS insert_on_customer_product_stats ON CustomerProductStats;
CREATE RULE insert_on_customer_product_stats AS
ON INSERT TO CustomerProductStats
DO INSTEAD (
    -- Attempt to find an existing order for the customer, or insert a minimal new one.
    -- This part is illustrative; a real scenario would have more robust order creation logic.
    -- For this lab, let's simplify: try to insert into order_details for an existing order_id.
    -- If NEW.order_id is provided, use it. Otherwise, this would need complex PL/pgSQL function.

    -- This rule demonstrates inserting a *new* order_detail for an *existing* order for a customer.
    -- We'll assume the INSERT on the view provides customer_id, product_id, order_id, quantity, unit_price, discount
    INSERT INTO order_details (order_id, product_id, unit_price, quantity, discount)
    SELECT
        NEW.order_id,
        NEW.product_id,
        NEW.unit_price,
        NEW.quantity,
        NEW.discount
    WHERE EXISTS (SELECT 1 FROM orders WHERE order_id = NEW.order_id AND customer_id = NEW.customer_id)
);


-- 3. Create ON UPDATE rule for the view (e.g., updating quantity)
DROP RULE IF EXISTS update_on_customer_product_stats ON CustomerProductStats;
CREATE RULE update_on_customer_product_stats AS
ON UPDATE TO CustomerProductStats
WHERE OLD.customer_id = NEW.customer_id AND OLD.product_id = NEW.product_id AND OLD.order_id = NEW.order_id -- Identifiers
DO INSTEAD
    UPDATE order_details
    SET
        quantity = NEW.quantity,
        unit_price = NEW.unit_price,
        discount = NEW.discount
    WHERE
        order_id = OLD.order_id AND product_id = OLD.product_id;

-- 4. Create ON DELETE rule for the view
DROP RULE IF EXISTS delete_on_customer_product_stats ON CustomerProductStats;
CREATE RULE delete_on_customer_product_stats AS
ON DELETE TO CustomerProductStats
DO INSTEAD
    DELETE FROM order_details
    WHERE
        order_id = OLD.order_id AND product_id = OLD.product_id;

-- --- Test Cases for View DML ---
-- Find an existing order detail for testing updates/deletes
SELECT * FROM CustomerProductStats WHERE customer_id = 'ALFKI' AND product_id = 11 AND order_id = 10248;

-- Test Update: Change quantity of product 11 in order 10248 for ALFKI from 12 to 15
UPDATE CustomerProductStats SET quantity = 15 WHERE customer_id = 'ALFKI' AND product_id = 11 AND order_id = 10248;
-- Verify in base table
SELECT quantity FROM order_details WHERE order_id = 10248 AND product_id = 11;
-- Reset quantity for next test if needed
UPDATE CustomerProductStats SET quantity = 12 WHERE customer_id = 'ALFKI' AND product_id = 11 AND order_id = 10248;

-- Test Delete: Delete product 42 from order 10248 for ALFKI
-- Check existence first
SELECT * FROM CustomerProductStats WHERE customer_id = 'ALFKI' AND product_id = 42 AND order_id = 10248;
DELETE FROM CustomerProductStats WHERE customer_id = 'ALFKI' AND product_id = 42 AND order_id = 10248;
-- Verify in base table and view
SELECT * FROM order_details WHERE order_id = 10248 AND product_id = 42; -- Should be empty
SELECT * FROM CustomerProductStats WHERE customer_id = 'ALFKI' AND product_id = 42 AND order_id = 10248; -- Should be empty
-- Restore the deleted record for future lab runs (if rule is dropped)
INSERT INTO order_details (order_id, product_id, unit_price, quantity, discount)
VALUES (10248, 42, 9.80000019, 10, 0);

-- Test Insert: Insert a new line item for existing order 10248 (customer ALFKI)
-- Add product 72 (quantity 5) again, pretending it's a new unique line item for simplicity.
-- This requires careful selection of product_id to avoid PK conflict in order_details.
-- Let's use a non-existent product ID for order 10248 to demonstrate.
-- First, find a product_id NOT in order 10248: e.g., product_id 1 (Chai)
SELECT * FROM order_details WHERE order_id = 10248 AND product_id = 1; -- Should be empty
INSERT INTO CustomerProductStats (customer_id, product_id, order_id, quantity, unit_price, discount, company_name, product_name)
VALUES ('ALFKI', 1, 10248, 3, 18.00, 0.0, 'Alfreds Futterkiste', 'Chai');
-- Verify in base table
SELECT * FROM order_details WHERE order_id = 10248 AND product_id = 1;
-- Clean up inserted record
DELETE FROM order_details WHERE order_id = 10248 AND product_id = 1;
```

**Expected Output/Explanation:**
*   The `UPDATE` statement on the view will appear to update 1 row. Verification will show the `quantity` change in the base `order_details` table.
*   The `DELETE` statement on the view will appear to delete 1 row. Verification will show the record removed from the base `order_details` table and thus from the view.
*   The `INSERT` statement on the view will appear to insert 1 row. Verification will show a new record in the `order_details` table.

**Learning Point:**
This task demonstrates the most common and powerful use case for `RULES`: enabling `INSTEAD OF` DML on views. It shows how the `OLD` and `NEW` pseudo-records are used to map view operations to corresponding DML operations on the underlying base tables, effectively making a complex view "updatable." It also demonstrates how a single `DO INSTEAD` can contain a single command. (Note: For very complex scenarios involving multiple linked tables, using a PL/pgSQL `INSTEAD OF` trigger on the view is often more practical than multiple rules or complex rule syntax).

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **Chained `DO ALSO`:** Modify Task 4's `log_price_changes` rule so that `DO ALSO` performs *two* separate actions:
    *   Inserts into `ProductPriceAudit` (as it currently does).
    *   *And also* inserts a generic log message into a `SystemActivityLog` temporary table (which you'd need to create) every time a product price is changed.
    *   *Careful: rules can lead to infinite loops if not designed well. Ensure actions don't inadvertently trigger the same rule again. This is typically why triggers are preferred.*
2.  **Conditional `INSTEAD` and `ALSO`:** For a new temporary `CriticalOrders` table (which has `order_id`), create a rule on `orders` `ON DELETE` where:
    *   If the deleted `order_id` is in `CriticalOrders`, `DO INSTEAD NOTHING`.
    *   Else (for non-critical orders), `DO ALSO` insert the `order_id` into a `DeletedOrdersLog` temporary table.
    *   *(Hint: You might need two separate rules for this conditional `INSTEAD` vs `ALSO` logic, or a very careful `WHERE` clause on the `INSTEAD` rule.)*
3.  **Complex View Insert Rule:** Enhance the `insert_on_customer_product_stats` rule from Task 5 to:
    *   If `NEW.order_id` is not provided, create a *new* minimal order for the `NEW.customer_id` first, and then use its generated `order_id` for inserting into `order_details`. (This will likely push you towards a PL/pgSQL `INSTEAD OF` trigger, highlighting the limitations of pure SQL rules for complex procedural logic).

**Further Exploration:**
*   **Rules vs. Triggers:** **Crucially, research and understand why database professionals generally recommend using `TRIGGERS` over `RULES` for DML operations on base tables.** Focus on atomicity, error handling, debugging, and firing order. Rules are primarily designed for view rewriting, while triggers are for procedural logic.
*   **Rule Firing Order:** If multiple rules exist for the same event (`ON INSERT`, `ON UPDATE`, `ON DELETE`) on the same table, understand how PostgreSQL determines their execution order.
*   **Performance Considerations:** Discuss when rules might impose performance overhead.
*   **`NOTIFY` with Rules:** Explore how rules can be used with `NOTIFY` to send asynchronous notifications.
*   **Clean up:** Remember to `DROP RULE` for all rules you created, and `DROP VIEW` for `CustomerProductStats`, and `DROP TABLE` for `ProductPriceAudit` and `SoftDeletedProducts` (if not temporary) to clean up your database.

---

### **Conclusion:**

In this extensive lab, you gained hands-on experience with PostgreSQL's powerful `RULES` feature. You learned to implement query rewriting logic for `INSERT`, `UPDATE`, and `DELETE` operations, utilizing `DO NOTHING` to prevent actions, `DO ALSO` for parallel logging/auditing, and `DO INSTEAD` for data modification, redirection, and enabling DML on complex views. This lab highlighted the unique capabilities of rules, particularly in view updatability, while implicitly setting the stage for understanding their distinctions from and typical preferences for `TRIGGERS` in DML scenarios.