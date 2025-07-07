# **Lab Title:** Advanced PL/pgSQL Functions for Business Logic

**Objective:**
This lab will provide hands-on experience with PostgreSQL's procedural language, PL/pgSQL. You will learn to create functions that incorporate variables, conditional logic, loops, and DML operations, as well as handle exceptions. These functions will solve complex business problems relevant to the Northwind database, such as calculating order totals, classifying customers, managing inventory, and processing order fulfillments.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
    *   *(If not installed, please refer to the installation instructions from the previous interaction or a reliable source for Northwind PostgreSQL setup.)*
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line, DBeaver, VS Code with PostgreSQL extension).
4.  **Basic SQL Knowledge:** Familiarity with fundamental SQL syntax (`SELECT`, `FROM`, `WHERE`, `JOIN`).

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run a simple query to confirm connectivity and data presence (e.g., `SELECT count(*) FROM customers;`). You should see a count reflecting the number of customers.
3.  **Enable PL/pgSQL Language:** Although usually enabled by default, ensure PL/pgSQL is available. Run `CREATE EXTENSION IF NOT EXISTS plpgsql;` (This command generally isn't strictly necessary as PL/pgSQL is usually installed by default in PostgreSQL, but it's good practice to ensure).

---

### **Lab Exercises:**

**Introduction to Tables Used:**
*   `orders`: Contains general order information like `freight`.
*   `order_details`: Stores individual product details including `unit_price`, `quantity`, and `discount`.
*   `products`: Stores product information like `units_in_stock`, `reorder_level`, and `product_name`.
*   `suppliers`: Stores supplier information like `company_name`.
*   `customers`: Contains customer details and `customer_id`.

We will also create a temporary helper table named `InventoryTransactionLog` to track inventory changes.

---

#### **Task 1: Scalar PL/pgSQL Function - Calculate Total Order Value (Including Freight)**

**Description:**
Create a PL/pgSQL function named `get_full_order_value` that calculates the *total value* of a specific order. This total value should include the sum of all discounted line items from `order_details` *plus* the `freight` cost from the `orders` table.

The function should:
1.  Take `p_order_id` (smallint) as input.
2.  Declare a local variable for `line_item_total` (real) and `total_order_value` (real).
3.  Calculate the sum of `(unit_price * quantity * (1 - discount))` for all items in the given order.
4.  Add the `freight` cost for that order to the calculated sum.
5.  Return the `total_order_value` (real).

**SQL Query:**

```sql
CREATE OR REPLACE FUNCTION get_full_order_value(p_order_id smallint)
RETURNS real AS $$
DECLARE
    line_item_total real;
    freight_cost    real;
    total_order_value real;
BEGIN
    -- Calculate sum of discounted line items
    SELECT SUM(od.unit_price * od.quantity * (1 - od.discount))
    INTO line_item_total
    FROM order_details od
    WHERE od.order_id = p_order_id;

    -- Get freight cost
    SELECT o.freight
    INTO freight_cost
    FROM orders o
    WHERE o.order_id = p_order_id;

    -- Handle case where order_id might not exist (both totals would be NULL)
    IF line_item_total IS NULL THEN
        RETURN NULL;
    END IF;

    -- Calculate final total
    total_order_value := line_item_total + COALESCE(freight_cost, 0.0);

    RETURN total_order_value;
END;
$$ LANGUAGE plpgsql;

-- Test the function for order ID 10248
SELECT get_full_order_value(10248) AS order_total_10248;

-- Test for order ID 10250 (which had a discount)
SELECT get_full_order_value(10250) AS order_total_10250;

-- Test for a non-existent order ID
SELECT get_full_order_value(99999) AS order_total_non_existent;
```

**Expected Output/Explanation:**
The `CREATE FUNCTION` statement should execute successfully.
*   For `order_id` 10248, the result should be approximately `517.58` (after calculating `(14*12) + (9.8*10) + (34.8*5) + 32.38`).
*   For `order_id` 10250, the result should be approximately `532.92`.
*   For `order_id` 99999, the result should be `NULL` (due to the `IF` condition).

**Learning Point:**
This task demonstrates the basic structure of a PL/pgSQL function: `DECLARE` block for local variables, `BEGIN...END` block for procedural logic, `SELECT INTO` for assigning query results to variables, conditional `IF` statements, and returning a scalar value.

---

#### **Task 2: PL/pgSQL Function with Conditional Logic - Classify Customer Value**

**Description:**
Create a PL/pgSQL function named `classify_customer_value` that categorizes a customer based on their total spending.

The function should:
1.  Take `p_customer_id` (character varying) as input.
2.  Calculate the total value of all orders placed by that customer (using the `get_full_order_value` function from Task 1, or by summing order details directly if you prefer, but using the function is good practice for reusability).
3.  Classify the customer:
    *   If total spending is `NULL` (no orders) or `0`: 'No Orders'
    *   If total spending is greater than `5000`: 'High-Value'
    *   If total spending is between `1000` and `5000` (inclusive): 'Medium-Value'
    *   Otherwise: 'Low-Value'
4.  Return the classification `TEXT`.

**SQL Query:**

```sql
CREATE OR REPLACE FUNCTION classify_customer_value(p_customer_id character varying)
RETURNS text AS $$
DECLARE
    customer_total_spending real;
    order_cursor CURSOR FOR
        SELECT order_id FROM orders WHERE customer_id = p_customer_id;
    current_order_id smallint;
BEGIN
    customer_total_spending := 0.0; -- Initialize to 0.0

    -- Loop through each order for the customer and sum its total value
    OPEN order_cursor;
    LOOP
        FETCH order_cursor INTO current_order_id;
        EXIT WHEN NOT FOUND;
        customer_total_spending := customer_total_spending + COALESCE(get_full_order_value(current_order_id), 0.0);
    END LOOP;
    CLOSE order_cursor;

    -- Classify based on total spending
    IF customer_total_spending IS NULL OR customer_total_spending = 0.0 THEN
        RETURN 'No Orders';
    ELSIF customer_total_spending > 5000.0 THEN
        RETURN 'High-Value';
    ELSIF customer_total_spending >= 1000.0 THEN
        RETURN 'Medium-Value';
    ELSE
        RETURN 'Low-Value';
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Test with a known high-value customer (e.g., ERNSH)
SELECT c.company_name, classify_customer_value(c.customer_id) AS customer_classification
FROM customers c WHERE c.customer_id = 'ERNSH';

-- Test with a known low-value customer (e.g., ANATR)
SELECT c.company_name, classify_customer_value(c.customer_id) AS customer_classification
FROM customers c WHERE c.customer_id = 'ANATR';

-- Test with a customer who might have no orders (if one exists in your data, or create one)
-- SELECT classify_customer_value('ZZZZA') AS customer_classification;
```

**Expected Output/Explanation:**
The function should be created successfully.
*   'ERNSH' should return 'High-Value' (or 'Medium-Value' if their total orders are less than 5000 in your dataset).
*   'ANATR' should return 'Low-Value'.
*   A non-existent customer ID should return 'No Orders'.

**Learning Point:**
This task demonstrates conditional logic (`IF...ELSIF...ELSE...END IF`) in PL/pgSQL, which is essential for implementing branching business rules. It also shows how to use a `CURSOR` and `LOOP` to iterate over a result set from a query, summing up values by calling another custom function.

---

#### **Task 3: PL/pgSQL Function Returning a Set of Records - Get Low Stock Products with Supplier Info**

**Description:**
Create a PL/pgSQL function named `get_low_stock_products_with_supplier` that identifies products with dangerously low stock levels.

The function should:
1.  Take no parameters.
2.  Return a `TABLE` (or `SETOF RECORD`) with columns: `product_name` (character varying), `units_in_stock` (smallint), `reorder_level` (smallint), and `supplier_company_name` (character varying).
3.  Iterate through all products that have `units_in_stock` less than or equal to their `reorder_level` AND `reorder_level` is greater than 0.
4.  For each such product, retrieve its name, current stock, reorder level, and the `company_name` of its supplier.
5.  Use `RETURN NEXT` to add each matching product's details to the result set.

**SQL Query:**

```sql
CREATE OR REPLACE FUNCTION get_low_stock_products_with_supplier()
RETURNS TABLE (
    product_name character varying,
    units_in_stock smallint,
    reorder_level smallint,
    supplier_company_name character varying
) AS $$
DECLARE
    product_record RECORD;
BEGIN
    FOR product_record IN
        SELECT
            p.product_name,
            p.units_in_stock,
            p.reorder_level,
            s.company_name AS supplier_company_name
        FROM
            products p
        JOIN
            suppliers s ON p.supplier_id = s.supplier_id
        WHERE
            p.units_in_stock <= p.reorder_level
            AND p.reorder_level > 0
            AND p.discontinued = 0 -- Exclude discontinued products from reorder
    LOOP
        product_name := product_record.product_name;
        units_in_stock := product_record.units_in_stock;
        reorder_level := product_record.reorder_level;
        supplier_company_name := product_record.supplier_company_name;
        RETURN NEXT; -- Return the current row and continue the loop
    END LOOP;
    RETURN; -- Indicate end of set
END;
$$ LANGUAGE plpgsql;

-- Call the function
SELECT * FROM get_low_stock_products_with_supplier();
```

**Expected Output/Explanation:**
The function should be created successfully. The query will return a table showing products like 'Chai', 'Konbu', 'Genen Shouyu', 'Teatime Chocolate Biscuits', 'Sir Rodney''s Scones', 'Mascarpone Fabioli', 'Geitost', 'Scottish Longbreads', 'Outback Lager', 'Longlife Tofu', and 'Original Frankfurter grüne Soße' (or similar, depending on current stock levels and your data). Each row will include the product name, stock, reorder level, and supplier company name.

**Learning Point:**
This task demonstrates a `FOR...LOOP` iterating over the results of a `SELECT` query, which is a common pattern for processing multiple rows in PL/pgSQL. It also shows how to use `RETURN NEXT` to build a result set when the function returns `TABLE` (or `SETOF RECORD`).

---

#### **Task 4: PL/pgSQL Function with DML and Exception Handling - Fulfill Order Line Item & Update Inventory**

**Description:**
Create a robust PL/pgSQL function named `fulfill_order_line_item` that simulates fulfilling a single line item of an order. This function will involve DML (Data Manipulation Language) and must handle various error conditions gracefully.

First, create a temporary table `InventoryTransaction` to log all inventory changes.

The function `fulfill_order_line_item` should:
1.  Take `p_order_id` (smallint), `p_product_id` (smallint), and `p_quantity_to_fulfill` (smallint) as input.
2.  **Declare:** Variables for current stock, product details (name), and the quantity ordered from `order_details`.
3.  **Validate Input:**
    *   Check if the `order_details` entry exists for the given `p_order_id` and `p_product_id`. If not, `RAISE EXCEPTION`.
    *   Check if `p_quantity_to_fulfill` is positive and not greater than the `quantity` ordered in `order_details`. If invalid, `RAISE EXCEPTION`.
4.  **Check Stock:** Retrieve the current `units_in_stock` for `p_product_id` from the `products` table.
5.  **Perform Fulfillment (Transactional):**
    *   If `units_in_stock` is less than `p_quantity_to_fulfill`, `RAISE EXCEPTION` for 'Insufficient stock'.
    *   **UPDATE:** Decrease `units_in_stock` in the `products` table by `p_quantity_to_fulfill`.
    *   **LOG:** Insert a record into the `InventoryTransaction` temporary table, logging the `product_id`, `transaction_date`, `quantity_change` (negative for fulfillment), and `description`.
6.  **Error Handling:** Use `EXCEPTION WHEN ... THEN` blocks to catch specific exceptions (e.g., `NO_DATA_FOUND` for invalid `order_details` entry, custom exceptions for insufficient stock/invalid quantity).
7.  **Return:** Return a `TEXT` status message (e.g., 'Success', 'Error: Insufficient stock', 'Error: Invalid order line item').

**SQL Query:**

```sql
-- 1. Create the temporary inventory transaction log table
DROP TABLE IF EXISTS InventoryTransaction CASCADE;
CREATE TEMPORARY TABLE InventoryTransaction (
    transaction_id SERIAL PRIMARY KEY,
    product_id SMALLINT,
    transaction_date TIMESTAMP DEFAULT NOW(),
    quantity_change SMALLINT,
    description TEXT
);

-- 2. Create the function
CREATE OR REPLACE FUNCTION fulfill_order_line_item(
    p_order_id smallint,
    p_product_id smallint,
    p_quantity_to_fulfill smallint
)
RETURNS text AS $$
DECLARE
    current_stock smallint;
    ordered_quantity smallint;
    product_name_val character varying(40);
BEGIN
    -- Start a transaction block (implicitly handled by PL/pgSQL if not explicitly opened)
    -- Check if order_details entry exists and get ordered quantity
    SELECT od.quantity
    INTO ordered_quantity
    FROM order_details od
    WHERE od.order_id = p_order_id AND od.product_id = p_product_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Order line item (Order ID: %, Product ID: %) not found.', p_order_id, p_product_id;
    END IF;

    -- Validate fulfillment quantity
    IF p_quantity_to_fulfill <= 0 OR p_quantity_to_fulfill > ordered_quantity THEN
        RAISE EXCEPTION 'Invalid quantity to fulfill (%). Must be positive and not exceed ordered quantity (%).', p_quantity_to_fulfill, ordered_quantity;
    END IF;

    -- Get current stock and product name
    SELECT p.units_in_stock, p.product_name
    INTO current_stock, product_name_val
    FROM products p
    WHERE p.product_id = p_product_id;

    IF current_stock IS NULL THEN -- Should not happen if product_id is valid, but good to check
        RAISE EXCEPTION 'Product ID % not found in products table.', p_product_id;
    END IF;

    -- Check for insufficient stock
    IF current_stock < p_quantity_to_fulfill THEN
        RAISE EXCEPTION 'Insufficient stock for product "%" (ID: %). Available: %, Requested: %.',
                        product_name_val, p_product_id, current_stock, p_quantity_to_fulfill;
    END IF;

    -- Perform inventory update
    UPDATE products
    SET units_in_stock = units_in_stock - p_quantity_to_fulfill
    WHERE product_id = p_product_id;

    -- Log the transaction
    INSERT INTO InventoryTransaction (product_id, quantity_change, description)
    VALUES (p_product_id, -p_quantity_to_fulfill, 'Fulfillment for Order ' || p_order_id);

    RETURN 'Success';

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        -- This shouldn't be hit due to the explicit NOT FOUND check, but as an example
        RETURN 'Error: Data not found for specified order/product.';
    WHEN SQLSTATE 'P0001' THEN -- This SQLSTATE is for RAISE EXCEPTION
        RETURN 'Error: ' || SQLERRM; -- SQLERRM contains the message from RAISE EXCEPTION
    WHEN OTHERS THEN
        RETURN 'An unexpected error occurred: ' || SQLSTATE || ' - ' || SQLERRM;
END;
$$ LANGUAGE plpgsql;

-- Before test: Get initial stock for product ID 11 (Queso Cabrales)
SELECT product_name, units_in_stock FROM products WHERE product_id = 11;

-- Test 1: Successful fulfillment for order 10248, product 11 (quantity 12)
SELECT fulfill_order_line_item(10248, 11, 12);
-- Verify updated stock and log
SELECT product_name, units_in_stock FROM products WHERE product_id = 11;
SELECT * FROM InventoryTransaction WHERE product_id = 11;

-- Test 2: Insufficient stock (try to fulfill more than available or just enough to make it insufficient after previous fulfillments)
-- Find the current stock of product 11 (Queso Cabrales)
-- SELECT units_in_stock FROM products WHERE product_id = 11;
-- Try to fulfill more than is now in stock
SELECT fulfill_order_line_item(10248, 11, 20); -- This should likely fail with insufficient stock
SELECT product_name, units_in_stock FROM products WHERE product_id = 11; -- Stock should NOT change if failed

-- Test 3: Invalid order line item (order_id 10248, product_id 999 - non-existent)
SELECT fulfill_order_line_item(10248, 999, 1);

-- Test 4: Invalid fulfillment quantity (p_quantity_to_fulfill = 0)
SELECT fulfill_order_line_item(10248, 11, 0);

-- Test 5: Invalid fulfillment quantity (p_quantity_to_fulfill > ordered_quantity)
-- The order_details for 10248, 42 has quantity 10
SELECT fulfill_order_line_item(10248, 42, 15);
```

**Expected Output/Explanation:**
The function should be created successfully.
*   **Test 1:** Returns 'Success'. `units_in_stock` for product 11 should decrease by 12. A log entry will appear in `InventoryTransaction`.
*   **Test 2:** Returns 'Error: Insufficient stock...'. `units_in_stock` for product 11 should *not* change further. No new log entry.
*   **Test 3:** Returns 'Error: Order line item (Order ID: 10248, Product ID: 999) not found.'.
*   **Test 4:** Returns 'Error: Invalid quantity to fulfill (0). Must be positive...'.
*   **Test 5:** Returns 'Error: Invalid quantity to fulfill (15). Must be positive and not exceed ordered quantity (10).'.

**Learning Point:**
This task demonstrates advanced PL/pgSQL features: performing DML (`UPDATE`, `INSERT`) within a function, handling explicit `RAISE EXCEPTION` for business rule violations, and using `EXCEPTION WHEN ... THEN` blocks to gracefully catch and respond to different error types (`SQLSTATE 'P0001'` for custom exceptions, `NO_DATA_FOUND`, `OTHERS` for general errors) by returning informative messages. It simulates a crucial transactional business process.

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **Refine Stock Check:** Modify `get_low_stock_products_with_supplier` to also show products that have `units_on_order` in addition to `units_in_stock` for a more complete reorder picture.
2.  **Order Summary Function:** Create a PL/pgSQL function `get_customer_order_summary(p_customer_id)` that returns a `TABLE` with `order_id`, `order_date`, and `order_total_value` (calling `get_full_order_value` for each order).
3.  **Bulk Fulfillment:** Create a new PL/pgSQL function `fulfill_order(p_order_id)` that attempts to fulfill *all* line items for a given order by iterating through `order_details` and calling `fulfill_order_line_item` for each product. The function should return 'Order Fully Fulfilled' or 'Partial Fulfillment: [reason]' (if any line item failed). This will require using `CONTINUE` in loop for partial fulfillment or `RAISE EXCEPTION` for full rollback.
4.  **Clean up Temporary Tables:** Remember to `DROP TABLE InventoryTransaction;` when you are done with the lab.

**Further Exploration:**
*   **`FOR` loops with `EXIT WHEN` and `CONTINUE WHEN`:** Explore more complex loop control structures.
*   **`GET DIAGNOSTICS`:** Research how to use `GET DIAGNOSTICS` to retrieve more information about the last executed SQL command (e.g., number of rows affected).
*   **Transactions in PL/pgSQL:** While PL/pgSQL functions run within an implicit transaction, understand how `BEGIN; ... EXCEPTION; ... COMMIT; / ROLLBACK;` blocks work if you explicitly define them within a function.
*   **Trigger Functions:** Learn how PL/pgSQL functions can be used as `TRIGGER` functions to automatically execute logic before or after DML operations on tables.

---

### **Conclusion:**

In this lab, you significantly expanded your PostgreSQL skills by delving into PL/pgSQL functions. You learned to implement procedural logic, manage variables, apply conditional statements, iterate over data, perform DML operations, and critically, handle errors and exceptions. By applying these concepts to Northwind-specific scenarios like calculating comprehensive order totals, classifying customers, finding low-stock items, and managing inventory fulfillment, you've gained practical experience in building robust and intelligent server-side business logic.