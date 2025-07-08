# Implementing Stored Procedures in PostgreSQL

## Introduction to Stored Procedures

Stored procedures are powerful constructs that allow you to encapsulate complex business logic directly within the database server. They are pre-compiled collections of SQL and procedural statements (written in languages like PL/pgSQL) that can be executed on demand. Stored procedures act as an essential layer for creating robust, secure, and efficient database applications.

Historically, PostgreSQL primarily used **functions** for server-side procedural logic. While functions are versatile, they operate under certain transactional constraints. With the introduction of the `PROCEDURE` command in PostgreSQL 11, a new type of routine became available, offering more explicit control over transactions within their body, making them akin to what other relational database management systems (RDBMS) refer to as "stored procedures."

### Why Use Stored Procedures?

1.  **Encapsulation of Business Logic:** They centralize complex, multi-step operations (e.g., processing an entire order, updating multiple related tables) directly in the database.
2.  **Reusability:** Once defined, a procedure can be called multiple times by different applications or users, avoiding redundant code.
3.  **Security:** You can grant users permission to execute a procedure without granting them direct permissions on the underlying tables, enhancing data security.
4.  **Performance:**
    *   **Reduced Network Traffic:** Instead of sending multiple SQL statements over the network, only a single `CALL` command is sent.
    *   **Optimized Execution:** Procedures are parsed and compiled once, and their execution plans can be cached, leading to faster subsequent executions.
    *   **Atomic Operations:** Procedures facilitate explicit transaction management, allowing a series of operations to be treated as a single, atomic unit.

### Stored Procedures vs. Functions (A Key Distinction)

While both PostgreSQL functions and procedures (as introduced in PG 11+) allow you to write procedural code, their primary distinguishing factor lies in **transaction management**:

*   **Functions (using `CREATE FUNCTION`):**
    *   Cannot execute transaction control commands (`COMMIT`, `ROLLBACK`, `SAVEPOINT`) within their body.
    *   Always execute within an existing transaction context. If called directly, they run in an implicit transaction. If called within a larger transaction, they are part of that transaction and cannot commit or roll back parts of it.
    *   Are designed to return a value (scalar or a set/table).
    *   Are called using `SELECT function_name(...)`.

*   **Procedures (using `CREATE PROCEDURE`):**
    *   **Can execute transaction control commands (`COMMIT`, `ROLLBACK`, `SAVEPOINT`) within their body.** This is their most significant feature.
    *   Can perform operations across multiple transactions.
    *   Do not directly return a value in the same way functions do. Any output is typically via `OUT` parameters or `RAISE NOTICE`.
    *   Are called using `CALL procedure_name(...)`.

This chapter will focus on these unique capabilities of PostgreSQL stored procedures, particularly their transaction management features, through practical Northwind database examples.

## Enabling PL/pgSQL

PostgreSQL procedures are primarily written in PL/pgSQL, PostgreSQL's native procedural language. While PL/pgSQL is usually enabled by default in a standard PostgreSQL installation, it's good practice to ensure it's available.

```sql
-- Ensure PL/pgSQL language extension is available
-- This command is typically not strictly necessary as PL/pgSQL is usually installed by default.
CREATE EXTENSION IF NOT EXISTS plpgsql;
```

If you ever encounter errors related to `language "plpgsql" does not exist`, running the above command (as a superuser) will fix it.

## Basic Procedure Syntax

The general structure for defining a stored procedure in PostgreSQL is as follows:

```sql
CREATE [ OR REPLACE ] PROCEDURE procedure_name (
    [ parameter_name_1 ] [ IN | OUT | INOUT ] parameter_type_1,
    [ parameter_name_2 ] [ IN | OUT | INOUT ] parameter_type_2,
    -- ... more parameters ...
)
LANGUAGE plpgsql
AS $$
DECLARE
    -- Optional: Variable declarations
BEGIN
    -- Procedure logic goes here
    -- This can include SQL statements (SELECT, INSERT, UPDATE, DELETE)
    -- PL/pgSQL control structures (IF, LOOP, FOR, WHILE)
    -- Transaction control commands (COMMIT, ROLLBACK, SAVEPOINT)

    -- Optional: Exception handling
    -- EXCEPTION
    --     WHEN some_exception THEN
    --         -- Handle error
    --         RAISE EXCEPTION 'An error occurred: %', SQLERRM;
END;
$$;
```

**Key Components:**

*   **`CREATE [OR REPLACE] PROCEDURE procedure_name(...)`**: Defines the procedure. `OR REPLACE` allows you to redefine an existing procedure without dropping it first.
*   **Parameters**:
    *   Each parameter has a `name` (optional, but good practice for readability) and a `type`.
    *   **`IN` (default)**: The parameter is passed into the procedure. The procedure can read its value but cannot change it.
    *   **`OUT`**: The parameter is used to return a value from the procedure to the caller. The procedure can set its value.
    *   **`INOUT`**: The parameter is passed into the procedure, and its value can be modified by the procedure and returned to the caller.
*   **`LANGUAGE plpgsql`**: Specifies that the procedure's body is written in PL/pgSQL.
*   **`AS $$ ... $$`**: Delimits the procedure's body. `$$` are dollar-quoted string literals, allowing you to avoid complex escaping of single quotes within your code.
*   **`DECLARE`**: (Optional) Block for declaring local variables, cursors, and other procedural elements.
*   **`BEGIN ... END`**: Contains the core logic of the procedure.

### Simple "Hello World" Procedure Example

Let's start with a basic procedure that simply logs a message.

```sql
CREATE OR REPLACE PROCEDURE greet_northwind_user(IN username TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Hello, %! Welcome to the Northwind database.', username;
END;
$$;
```

## Calling Procedures

Unlike functions, which are called using `SELECT`, stored procedures are executed using the `CALL` statement.

### Calling the "Hello World" Procedure

```sql
CALL greet_northwind_user('Maria Anders');
-- Expected Output (as a NOTICE message in your client):
-- NOTICE:  Hello, Maria Anders! Welcome to the Northwind database.
-- CALL
```

When calling a procedure with `OUT` or `INOUT` parameters, you typically pass variables from your client environment (if supported) or `NULL` placeholders, and then retrieve the output. In psql, you can declare local variables for this.

```sql
-- Procedure with an OUT parameter
CREATE OR REPLACE PROCEDURE get_customer_contact(
    IN customer_id_param character varying,
    OUT contact_name_out character varying,
    OUT phone_out character varying
)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT contact_name, phone
    INTO contact_name_out, phone_out
    FROM customers
    WHERE customer_id = customer_id_param;

    IF NOT FOUND THEN
        contact_name_out := 'Customer Not Found';
        phone_out := 'N/A';
    END IF;
END;
$$;

-- Calling the procedure with OUT parameters in psql
DO $$
DECLARE
    v_contact_name character varying;
    v_phone        character varying;
BEGIN
    CALL get_customer_contact('ALFKI', v_contact_name, v_phone);
    RAISE NOTICE 'Customer ALFKI Contact: % Phone: %', v_contact_name, v_phone;

    CALL get_customer_contact('NONEXIST', v_contact_name, v_phone);
    RAISE NOTICE 'Customer NONEXIST Contact: % Phone: %', v_contact_name, v_phone;
END;
$$;
-- Expected Output (as NOTICE messages):
-- NOTICE:  Customer ALFKI Contact: Maria Anders Phone: 030-0074321
-- NOTICE:  Customer NONEXIST Contact: Customer Not Found Phone: N/A
```

## Core Concepts in PL/pgSQL Procedures

Procedures leverage the full power of PL/pgSQL, allowing for sophisticated logic.

### Variable Declarations

Variables are defined in the `DECLARE` block.

```sql
DECLARE
    total_amount NUMERIC(10, 2) DEFAULT 0.0; -- Numeric variable with a default value
    customer_name VARCHAR(40);             -- String variable
    is_premium BOOLEAN := FALSE;           -- Boolean variable with assignment operator
    order_cursor CURSOR FOR SELECT order_id, order_date FROM orders WHERE customer_id = 'VINET'; -- Cursor declaration
```

### Control Structures

PL/pgSQL provides standard programming control structures.

*   **Conditional Statements (`IF...ELSIF...ELSE`)**:
    ```sql
    IF quantity_in_stock < reorder_level THEN
        RAISE NOTICE 'Product % needs reordering.', product_name;
    ELSIF quantity_in_stock < 50 THEN
        RAISE NOTICE 'Product % is getting low.', product_name;
    ELSE
        RAISE NOTICE 'Product % stock is healthy.', product_name;
    END IF;
    ```
*   **Loops (`LOOP`, `WHILE`, `FOR`)**:
    ```sql
    -- Basic loop with EXIT
    LOOP
        -- do something
        EXIT WHEN some_condition;
    END LOOP;

    -- WHILE loop
    WHILE counter < 10 LOOP
        -- increment counter
        counter := counter + 1;
    END LOOP;

    -- FOR loop (iterating over a range)
    FOR i IN 1..10 LOOP
        RAISE NOTICE 'Loop iteration: %', i;
    END LOOP;

    -- FOR loop (iterating over query results) - very common!
    FOR customer_record IN SELECT customer_id, company_name FROM customers WHERE country = 'USA' LOOP
        RAISE NOTICE 'Processing customer: % (%)', customer_record.company_name, customer_record.customer_id;
    END LOOP;
    ```

### Error Handling (`EXCEPTION` Block)

Procedures can catch and handle errors that occur during their execution.

```sql
BEGIN
    -- Code that might raise an exception
    INSERT INTO products (product_id, product_name, ...) VALUES (1, 'Duplicate ID', ...); -- This would cause a PK violation

EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Caught unique violation: Product ID already exists.';
        -- You could log this, try an update instead, or return a specific error code
        -- ROLLBACK; -- You can rollback partial work here
    WHEN division_by_zero THEN
        RAISE NOTICE 'Caught division by zero error.';
    WHEN OTHERS THEN -- Catches any other unhandled exception
        RAISE NOTICE 'An unexpected error occurred: SQLSTATE % - %', SQLSTATE, SQLERRM;
        -- ROLLBACK; -- Or rollback all changes
END;
```

*   `SQLSTATE`: A 5-character code indicating the specific error type (e.g., `23505` for `unique_violation`).
*   `SQLERRM`: The human-readable error message.

## Explicit Transaction Management in Procedures

This is the most crucial aspect that distinguishes PostgreSQL procedures from functions. A procedure can explicitly use `COMMIT`, `ROLLBACK`, and `SAVEPOINT` statements within its body.

When a procedure is called:
*   If it is called **outside** an existing transaction block, the `CALL` statement itself will initiate an implicit transaction. Any `COMMIT` or `ROLLBACK` within the procedure will then operate on this implicit transaction.
*   If it is called **inside** an explicit transaction block (e.g., after `BEGIN;` or `START TRANSACTION;`), then any `COMMIT` or `ROLLBACK` within the procedure will operate on the *caller's transaction*, potentially committing or rolling back the work done *before* the procedure was called, as well as the work within the procedure. This is powerful but requires extreme caution.

### `BEGIN;`

Starts a new transaction block. This is often used at the beginning of a procedure to ensure that all operations within the procedure are treated as a single atomic unit.


### `COMMIT;`

Ends the current transaction, making all changes permanent. If the procedure was called within a larger transaction, `COMMIT;` will commit that entire parent transaction.

### `ROLLBACK;`

Aborts the current transaction, undoing all changes since the transaction started (or since the last `COMMIT`). Similar to `COMMIT;`, `ROLLBACK;` will roll back the entire parent transaction if the procedure was called within one.

### `SAVEPOINT savepoint_name;`

Sets a savepoint within the current transaction. This allows you to roll back to a specific point within the transaction without abandoning the entire transaction.

### `ROLLBACK TO SAVEPOINT savepoint_name;`

Undoes all changes made since the specified savepoint was set, but the transaction remains active.

### `RELEASE SAVEPOINT savepoint_name;`

Removes a savepoint, keeping all changes made since that savepoint (unless a later `ROLLBACK` or `ROLLBACK TO SAVEPOINT` affects them).

**Crucial Warning:** Using `COMMIT` or `ROLLBACK` inside a procedure when it's part of a larger, explicit transaction can lead to unexpected behavior for the caller, as it will commit/rollback the caller's transaction. It's generally recommended to only use `COMMIT`/`ROLLBACK` inside a procedure if the procedure is explicitly designed to be the sole transactional unit, or if it's called as a top-level command. For most nested scenarios, using `SAVEPOINT`s might be safer if partial rollbacks are needed.

## Real-World Northwind Examples

Let's apply these concepts to typical Northwind business scenarios.

### Example 1: `UPDATE` and Audit Customer Contact

**Scenario:** A customer's contact information (phone and fax) is updated. This operation should be audited by logging the old and new values.

**Application Context:** Data governance, compliance, and historical tracking of master data changes.

**Steps:**
1.  Create an `CustomerContactAudit` table.
2.  Create a procedure `update_customer_contact` that updates `customers` and logs to `CustomerContactAudit`.

```sql
-- 1. Create the audit table
DROP TABLE IF EXISTS CustomerContactAudit CASCADE;
CREATE TEMPORARY TABLE CustomerContactAudit (
    audit_id SERIAL PRIMARY KEY,
    customer_id CHARACTER VARYING(5) NOT NULL,
    old_phone CHARACTER VARYING(24),
    new_phone CHARACTER VARYING(24),
    old_fax CHARACTER VARYING(24),
    new_fax CHARACTER VARYING(24),
    change_timestamp TIMESTAMP DEFAULT NOW(),
    changed_by TEXT DEFAULT CURRENT_USER
);

-- 2. Create the procedure
CREATE OR REPLACE PROCEDURE update_customer_contact(
    IN p_customer_id CHARACTER VARYING,
    IN p_new_phone CHARACTER VARYING,
    IN p_new_fax CHARACTER VARYING
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_old_phone CHARACTER VARYING;
    v_old_fax CHARACTER VARYING;
BEGIN
    -- Get old values for auditing
    SELECT phone, fax
    INTO v_old_phone, v_old_fax
    FROM customers
    WHERE customer_id = p_customer_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Customer ID % not found.', p_customer_id;
    END IF;

    -- Update the customer's contact info
    UPDATE customers
    SET
        phone = p_new_phone,
        fax = p_new_fax
    WHERE
        customer_id = p_customer_id;

    -- Log the change only if values are different
    IF v_old_phone IS DISTINCT FROM p_new_phone OR v_old_fax IS DISTINCT FROM p_new_fax THEN
        INSERT INTO CustomerContactAudit (customer_id, old_phone, new_phone, old_fax, new_fax)
        VALUES (p_customer_id, v_old_phone, p_new_phone, v_old_fax, p_new_fax);
    END IF;

    RAISE NOTICE 'Contact information updated and audited for customer %', p_customer_id;

END;
$$;

-- --- Test Cases ---

-- Before: Check customer ALFKI's contact info and audit log
SELECT customer_id, company_name, phone, fax FROM customers WHERE customer_id = 'ALFKI';
SELECT * FROM CustomerContactAudit WHERE customer_id = 'ALFKI';

-- Test 1: Update ALFKI's phone and fax
CALL update_customer_contact('ALFKI', '(030) 555-1234', '(030) 555-5678');

-- After: Verify update and audit log
SELECT customer_id, company_name, phone, fax FROM customers WHERE customer_id = 'ALFKI';
SELECT * FROM CustomerContactAudit WHERE customer_id = 'ALFKI';

-- Test 2: Update an non-existent customer (should raise exception)
CALL update_customer_contact('NONEX', '(111) 222-3333', NULL);

-- Test 3: Update a customer with same values (should update customer, but NOT log)
CALL update_customer_contact('ALFKI', '(030) 555-1234', '(030) 555-5678');
SELECT * FROM CustomerContactAudit WHERE customer_id = 'ALFKI'; -- Should only have the first audit entry

-- Clean up: Reset customer ALFKI's phone/fax and clear audit log
UPDATE customers SET phone = '030-0074321', fax = '030-0076545' WHERE customer_id = 'ALFKI';
DELETE FROM CustomerContactAudit WHERE customer_id = 'ALFKI';
```

### Example 2: Complex Order Fulfillment with Explicit Transaction Control

**Scenario:** Process a new order received from a customer. This involves several steps:
1.  Validate that the product exists and has sufficient stock.
2.  Decrease the `units_in_stock` for each ordered product.
3.  Insert the main order record into the `orders` table.
4.  Insert each line item into the `order_details` table.
5.  If *any* step fails (e.g., insufficient stock, invalid product, database error), the entire operation (stock reduction, order insert, order details insert) must be rolled back to ensure data consistency.

**Application Context:** Mission-critical business processes where atomicity is paramount (e.g., e-commerce transactions, financial transfers).

**Steps:**
1.  Create a temporary `InventoryLog` table (if not already created from previous labs) to track stock changes.
2.  Create a procedure `process_new_order` that handles all steps transactionally.
3.  Simulate success and failure cases.

```sql
-- 1. Create a temporary Inventory Log table
DROP TABLE IF EXISTS InventoryLog CASCADE;
CREATE TEMPORARY TABLE InventoryLog (
    log_id SERIAL PRIMARY KEY,
    product_id SMALLINT,
    quantity_change SMALLINT,
    log_type TEXT, -- e.g., 'ORDER_FULFILLMENT', 'STOCK_ADJUSTMENT'
    order_id SMALLINT,
    log_timestamp TIMESTAMP DEFAULT NOW()
);

-- 2. Create the procedure
CREATE OR REPLACE PROCEDURE process_new_order(
    IN p_customer_id character varying,
    IN p_employee_id smallint,
    IN p_required_date date,
    IN p_ship_via smallint,
    IN p_freight real,
    IN p_ship_name character varying,
    IN p_ship_address character varying,
    IN p_ship_city character varying,
    IN p_ship_country character varying,
    IN p_items JSONB -- JSONB array of objects: [{"product_id": 1, "quantity": 10, "unit_price": 18.0, "discount": 0.0}]
)
LANGUAGE plpgsql
AS $$
DECLARE
    new_order_id smallint;
    item_record JSONB;
    v_product_id smallint;
    v_quantity smallint;
    v_unit_price real;
    v_discount real;
    current_stock smallint;
    product_name_val character varying(40);
BEGIN
    -- Start a new transaction (if not already in one, or if outer transaction is to be controlled by procedure)
    -- If called inside an explicit transaction, this COMMIT/ROLLBACK affects the parent.
    -- For this scenario, we assume this procedure is the primary transactional unit.
    -- If you want this procedure to be able to rollback *itself* but not the *caller*,
    -- you'd typically use SAVEPOINTS or separate transaction management in the caller.
    -- For demonstration, this procedure assumes it controls the transaction from here.
    -- This means if an error occurs and is caught, we can ROLLBACK.
    -- If it's a critical application, you might use BEGIN/COMMIT/ROLLBACK.
    -- Here, let's allow it to COMMIT/ROLLBACK an implicit transaction if called standalone.

    -- Get a new order_id before inserting (assuming order_id is SERIAL and auto-assigned, not chosen manually)
    -- If order_id is SERIAL, it will be generated on INSERT, so we need to retrieve it.

    -- Check if customer and employee exist (basic validation)
    IF NOT EXISTS (SELECT 1 FROM customers WHERE customer_id = p_customer_id) THEN
        RAISE EXCEPTION 'Invalid Customer ID: %', p_customer_id;
    END IF;
    IF NOT EXISTS (SELECT 1 FROM employees WHERE employee_id = p_employee_id) THEN
        RAISE EXCEPTION 'Invalid Employee ID: %', p_employee_id;
    END IF;

    -- Step 1: Validate stock for all items BEFORE inserting anything
    FOR item_record IN SELECT * FROM jsonb_array_elements(p_items)
    LOOP
        v_product_id := (item_record ->> 'product_id')::smallint;
        v_quantity   := (item_record ->> 'quantity')::smallint;

        SELECT units_in_stock, product_name
        INTO current_stock, product_name_val
        FROM products
        WHERE product_id = v_product_id;

        IF NOT FOUND THEN
            RAISE EXCEPTION 'Product ID % not found.', v_product_id;
        END IF;

        IF current_stock < v_quantity THEN
            RAISE EXCEPTION 'Insufficient stock for product "%" (ID: %). Available: %, Requested: %.',
                            product_name_val, v_product_id, current_stock, v_quantity;
        END IF;
    END LOOP;

    -- All validations passed, proceed with DML
    -- Step 2: Insert the main order record
    INSERT INTO orders (
        customer_id, employee_id, order_date, required_date, shipped_date,
        ship_via, freight, ship_name, ship_address, ship_city, ship_region,
        ship_postal_code, ship_country
    ) VALUES (
        p_customer_id, p_employee_id, NOW()::DATE, p_required_date, NULL,
        p_ship_via, p_freight, p_ship_name, p_ship_address, p_ship_city, p_ship_country
    ) RETURNING order_id INTO new_order_id; -- Get the newly generated order_id

    -- Step 3: Insert each line item and decrease stock
    FOR item_record IN SELECT * FROM jsonb_array_elements(p_items)
    LOOP
        v_product_id := (item_record ->> 'product_id')::smallint;
        v_quantity   := (item_record ->> 'quantity')::smallint;
        v_unit_price := (item_record ->> 'unit_price')::real;
        v_discount   := (item_record ->> 'discount')::real;

        INSERT INTO order_details (order_id, product_id, unit_price, quantity, discount)
        VALUES (new_order_id, v_product_id, v_unit_price, v_quantity, v_discount);

        -- Decrease stock
        UPDATE products
        SET units_in_stock = units_in_stock - v_quantity
        WHERE product_id = v_product_id;

        -- Log stock change
        INSERT INTO InventoryLog (product_id, quantity_change, log_type, order_id)
        VALUES (v_product_id, -v_quantity, 'ORDER_FULFILLMENT', new_order_id);
    END LOOP;

    RAISE NOTICE 'Order % processed successfully for customer %', new_order_id, p_customer_id;
    COMMIT; -- Explicitly commit the transaction
    
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Order processing failed: % - %', SQLSTATE, SQLERRM;
        ROLLBACK; -- Explicitly roll back on any error
        RAISE; -- Re-raise the exception after rollback to notify the caller
END;
$$;

-- --- Test Cases ---

-- Before: Check customer, product stock, and order counts
SELECT company_name FROM customers WHERE customer_id = 'ALFKI';
SELECT product_name, units_in_stock FROM products WHERE product_id IN (1, 2, 3);
SELECT count(*) FROM orders WHERE customer_id = 'ALFKI';
SELECT count(*) FROM order_details WHERE order_id IN (SELECT order_id FROM orders WHERE customer_id = 'ALFKI');
SELECT * FROM InventoryLog;

-- Test 1: Successful Order Processing
CALL process_new_order(
    'ALFKI', 1, '1997-08-01'::DATE, 1, 5.00, 'Alfreds Futterkiste', 'Obere Str. 57', 'Berlin', 'Germany',
    '[
        {"product_id": 1, "quantity": 5, "unit_price": 18.0, "discount": 0.0},
        {"product_id": 2, "quantity": 10, "unit_price": 19.0, "discount": 0.1}
    ]'::JSONB
);

-- After: Verify order, stock, and log
SELECT order_id, customer_id, order_date, freight FROM orders WHERE customer_id = 'ALFKI' ORDER BY order_id DESC LIMIT 1;
SELECT product_name, units_in_stock FROM products WHERE product_id IN (1, 2);
SELECT * FROM InventoryLog WHERE product_id IN (1, 2) ORDER BY log_id DESC LIMIT 2;

-- Test 2: Order Processing Failure (Insufficient Stock for Product 3)
-- Find current stock of Aniseed Syrup (product_id 3)
SELECT product_name, units_in_stock FROM products WHERE product_id = 3;
-- Try to order more than is in stock (e.g., 200, if stock is 13)
CALL process_new_order(
    'VINET', 2, '1997-08-05'::DATE, 2, 10.00, 'Vins et alcools Chevalier', '59 rue de l''Abbaye', 'Reims', 'France',
    '[
        {"product_id": 1, "quantity": 1, "unit_price": 18.0, "discount": 0.0},
        {"product_id": 3, "quantity": 200, "unit_price": 10.0, "discount": 0.0} -- Insufficient stock
    ]'::JSONB
);

-- After Failure: Verify NO order was created and stock did NOT change
SELECT count(*) FROM orders WHERE customer_id = 'VINET' AND order_date = '1997-08-05'::DATE; -- Should be 0
SELECT product_name, units_in_stock FROM products WHERE product_id IN (1, 3); -- Stock should be unchanged
SELECT * FROM InventoryLog WHERE product_id IN (1, 3) AND order_id IS NULL; -- No log entries from this failed attempt

-- Test 3: Order Processing Failure (Invalid Product ID)
CALL process_new_order(
    'ALFKI', 1, '1997-08-02'::DATE, 1, 5.00, 'Alfreds Futterkiste', 'Obere Str. 57', 'Berlin', 'Germany',
    '[
        {"product_id": 9999, "quantity": 5, "unit_price": 1.0, "discount": 0.0} -- Invalid product ID
    ]'::JSONB
);
-- After Failure: Verify NO order was created and stock did NOT change
SELECT count(*) FROM orders WHERE customer_id = 'ALFKI' AND order_date = '1997-08-02'::DATE; -- Should be 0
```

## Stored Procedures vs. Functions Revisited (Detailed Comparison)

| Feature                   | PostgreSQL **Function** (`CREATE FUNCTION`)                                                                              | PostgreSQL **Procedure** (`CREATE PROCEDURE`)                                                                                |
| :------------------------ | :----------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------- |
| **Purpose**               | Designed to return a value (scalar, set, table). Primarily used in `SELECT` statements, `WHERE` clauses, or expressions. | Designed to execute a sequence of operations, typically with side effects (DML on tables). Does not directly return a value. |
| **Calling Method**        | `SELECT function_name(args)` or used within expressions.                                                                 | `CALL procedure_name(args)` as a standalone statement.                                                                       |
| **Transaction Control**   | **Cannot** contain `COMMIT`, `ROLLBACK`, `SAVEPOINT`. Executes within the caller's transaction context.                  | **Can** contain `COMMIT`, `ROLLBACK`, `SAVEPOINT`. Has explicit control over the transaction.                                |
| **Transaction Scope**     | Always part of the encompassing transaction. If an error occurs, the entire transaction typically rolls back.            | Can begin, commit, or roll back its own transaction, or control the parent transaction it was called within.                 |
| **Return Value**          | Mandatory (scalar, `SETOF type`, `TABLE(...)`).                                                                          | No direct return value. Output can be via `OUT` parameters or `RAISE NOTICE`.                                                |
| **`INOUT` Parameters**    | Supported.                                                                                                               | Supported.                                                                                                                   |
| **`TG_OP`, `OLD`, `NEW`** | Available in trigger functions.                                                                                          | Not directly available outside of trigger functions.                                                                         |
| **Side Effects**          | Can have side effects (DML), but this often implies running within the caller's transaction.                             | Primarily designed for side effects (DML).                                                                                   |
| **Use Cases**             | Calculations, data transformations, generating sets of rows, complex lookups.                                            | Multi-step business processes, transactional workflows, batch processing, complex data migrations.                           |

The choice between a function and a procedure boils down to whether you need to manage transactions explicitly within the routine and whether the routine's primary purpose is to return data or to perform an action. For true transactional control and complex DML workflows, procedures are the appropriate choice.

## Conclusion

PostgreSQL's stored procedures, introduced with the `CREATE PROCEDURE` command, are a significant addition to its server-side programming capabilities. They empower developers to encapsulate intricate business logic, manage data consistency through explicit transaction control, and enhance the performance and security of database operations.

By understanding the unique ability of procedures to issue `COMMIT`, `ROLLBACK`, and `SAVEPOINT` statements, you can design robust transactional workflows that ensure atomicity across multiple DML operations. While this power comes with the responsibility of careful transaction management (especially when nested), procedures are an invaluable tool for building sophisticated and reliable applications on PostgreSQL.