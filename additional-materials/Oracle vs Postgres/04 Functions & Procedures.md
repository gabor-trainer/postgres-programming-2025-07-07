## Functions & Procedures

Stored program units form the backbone of business logic within the database. While both Oracle and PostgreSQL offer functions and procedures, their design philosophies, especially concerning transactional behavior and result set handling, diverge significantly. Understanding these distinctions is crucial for successful migration and development.

### Defining Functions (`CREATE FUNCTION...RETURN <type>`)

In PostgreSQL, the `CREATE FUNCTION` statement is used to define a stored program unit that must always return a value. Traditionally, it has been the primary vehicle for all procedural code.

| Oracle (PL/SQL)                                                                                          | Postgres (PL/pgSQL)                                                                                         | Notes / Key Differences                                                                                                                                           |
| :------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CREATE [OR REPLACE] FUNCTION func_name(p_param_name IN type) RETURN ret_type IS ... END [func_name]; /` | `CREATE [OR REPLACE] FUNCTION func_name(p_param_name type) RETURNS ret_type LANGUAGE plpgsql AS $$ ... $$;` | The `LANGUAGE plpgsql` clause is mandatory. The function body is delimited by dollar-quoting `$$ ... $$` or standard `AS 'code'`. Parameters typically lack `IN`. |
| Implicitly commits if DDL inside (Oracle)                                                                | No DDL (and no explicit `COMMIT`/`ROLLBACK`) allowed inside functions. (Critical: see last subsection)      | Functions run within the calling transaction's context.                                                                                                           |

**Example: Calculate Line Item Total for Order Detail (Northwind `order_details`)**

Let's create a function that computes the net value of an order detail line, factoring in discount.

```sql
-- Oracle PL/SQL Function
CREATE OR REPLACE FUNCTION get_order_line_net_value (
    p_unit_price  IN order_details.unit_price%TYPE,
    p_quantity    IN order_details.quantity%TYPE,
    p_discount    IN order_details.discount%TYPE
)
RETURN NUMBER IS
    l_net_value NUMBER;
BEGIN
    l_net_value := p_unit_price * p_quantity * (1 - p_discount);
    RETURN l_net_value;
END;
/

-- Test in Oracle:
SELECT od.order_id, od.product_id, od.unit_price, od.quantity, od.discount,
       get_order_line_net_value(od.unit_price, od.quantity, od.discount) AS net_value
FROM order_details od
WHERE od.order_id = 10248 AND od.product_id = 11;
```

```sql
-- PostgreSQL PL/pgSQL Function
CREATE OR REPLACE FUNCTION get_order_line_net_value (
    p_unit_price  order_details.unit_price%TYPE,
    p_quantity    order_details.quantity%TYPE,
    p_discount    order_details.discount%TYPE
)
RETURNS REAL -- unit_price, discount, quantity are REAL, SMALLINT, REAL types
LANGUAGE plpgsql
AS $$
DECLARE
    l_net_value REAL;
BEGIN
    l_net_value := p_unit_price * p_quantity * (1 - p_discount);
    RETURN l_net_value;
END;
$$;

-- Test in PostgreSQL:
SELECT od.order_id, od.product_id, od.unit_price, od.quantity, od.discount,
       get_order_line_net_value(od.unit_price, od.quantity, od.discount) AS net_value
FROM order_details od
WHERE od.order_id = 10248 AND od.product_id = 11;
```

**TL;DR:** PostgreSQL functions are stateless and generally intended for computations or data retrieval, with return values. The `LANGUAGE plpgsql` clause and dollar-quoting `AS $$ ... $$` are distinctive elements.

---

### Defining Procedures (`CREATE PROCEDURE...`) - New in PG 11+

Prior to PostgreSQL 11, `CREATE PROCEDURE` did not exist. For Oracle developers, this meant a significant paradigm shift for transactional logic. With PostgreSQL 11 and later, `CREATE PROCEDURE` was introduced to address this gap, explicitly allowing transaction control within procedural blocks.

| Oracle (PL/SQL)                                                                           | Postgres (PL/pgSQL)                                                                              | Notes / Key Differences                                                                                                            |
| :---------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------- |
| `CREATE [OR REPLACE] PROCEDURE proc_name(p_param_name IN type) IS ... END [proc_name]; /` | `CREATE [OR REPLACE] PROCEDURE proc_name(p_param_name type) LANGUAGE plpgsql AS $$ ... $$;`      | Very similar syntax to functions, but using `PROCEDURE` keyword and not requiring `RETURNS` clause. Parameters lack `IN` here too. |
| Call with: `EXEC proc_name();` or anonymous block `BEGIN proc_name(); END;`               | Call with: `CALL proc_name();` (Mandatory, `SELECT proc_name()` will not work)                   | PostgreSQL procedures *must* be invoked using the `CALL` statement.                                                                |
| Can include `COMMIT`/`ROLLBACK`/DDL                                                       | Can include `COMMIT`/`ROLLBACK`/DDL. (This is their primary distinction from `CREATE FUNCTION`s) | This restores familiar transaction management capabilities for DML/DDL operations.                                                 |

**Example: Log Product Price Change (Northwind `products`)**

Let's imagine a logging mechanism (e.g., an `audit_log` table not in current Northwind, so we'll simulate a `RAISE NOTICE`). A procedure to record a price adjustment would be suitable.

```sql
-- Oracle PL/SQL Procedure
CREATE OR REPLACE PROCEDURE adjust_product_price (
    p_product_id  IN products.product_id%TYPE,
    p_new_price   IN products.unit_price%TYPE
) IS
    l_old_price products.unit_price%TYPE;
BEGIN
    SELECT unit_price INTO l_old_price
    FROM products
    WHERE product_id = p_product_id;

    UPDATE products
    SET unit_price = p_new_price
    WHERE product_id = p_product_id;

    DBMS_OUTPUT.PUT_LINE('LOG: Product ' || p_product_id || ' price changed from ' || l_old_price || ' to ' || p_new_price || '.');
    COMMIT; -- Explicit commit
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
/

-- Test in Oracle:
-- SELECT unit_price FROM products WHERE product_id = 1; -- Should be 18
-- CALL adjust_product_price(1, 19.5); -- Change Chai price
-- SELECT unit_price FROM products WHERE product_id = 1; -- Should be 19.5
-- CALL adjust_product_price(1, 18); -- Change it back
```

```sql
-- PostgreSQL PL/pgSQL Procedure (PG 11+)
CREATE OR REPLACE PROCEDURE adjust_product_price (
    p_product_id  products.product_id%TYPE,
    p_new_price   products.unit_price%TYPE
)
LANGUAGE plpgsql
AS $$
DECLARE
    l_old_price products.unit_price%TYPE;
BEGIN
    SELECT unit_price INTO l_old_price
    FROM products
    WHERE product_id = p_product_id;

    UPDATE products
    SET unit_price = p_new_price
    WHERE product_id = p_product_id;

    RAISE NOTICE 'LOG: Product % price changed from % to %.', p_product_id, l_old_price, p_new_price;
    COMMIT; -- Explicit commit now allowed in procedures
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
$$;

-- Test in PostgreSQL:
-- SELECT unit_price FROM products WHERE product_id = 1; -- Should be 18
-- CALL adjust_product_price(1, 19.5); -- Change Chai price
-- SELECT unit_price FROM products WHERE product_id = 1; -- Should be 19.5
-- CALL adjust_product_price(1, 18); -- Change it back
```

**TL;DR:** For procedural logic that manages its own transactions or includes DDL, `CREATE PROCEDURE` (PG 11+) is the correct and most direct equivalent to Oracle's procedures. Remember to always use `CALL` to invoke them.

---

### Parameters: `IN`, `OUT`, `INOUT`

PostgreSQL functions and procedures support `IN`, `OUT`, and `INOUT` parameters, directly paralleling Oracle's functionality.

| Oracle (PL/SQL)                                                   | Postgres (PL/pgSQL)                                                                                         | Notes / Key Differences                                                                  |
| :---------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------- |
| `proc_name(p1 IN TYPE, p2 OUT TYPE, p3 IN OUT TYPE)`              | `func_proc_name(IN p1 TYPE, OUT p2 TYPE, INOUT p3 TYPE)`                                                    | Keywords (`IN`, `OUT`, `INOUT`) are present, but positioned *before* the parameter name. |
| `OUT` parameters for functions often dictate multi-value returns. | `OUT` parameters for functions often return a record or use `RETURNS TABLE(...)` or `RETURNS SETOF RECORD`. | This is where multi-value returns in functions come into play.                           |

**Example: Get Product Details by ID (Northwind `products`)**

Let's create a function that takes a product ID and returns the product name, unit price, and units in stock using `OUT` parameters.

```sql
-- Oracle PL/SQL Function with OUT parameters
CREATE OR REPLACE FUNCTION get_product_details_by_id (
    p_product_id  IN products.product_id%TYPE,
    p_product_name OUT products.product_name%TYPE,
    p_unit_price   OUT products.unit_price%TYPE,
    p_units_in_stock OUT products.units_in_stock%TYPE
)
RETURN NUMBER -- Function must return something, here just success indicator
IS
BEGIN
    SELECT product_name, unit_price, units_in_stock
    INTO p_product_name, p_unit_price, p_units_in_stock
    FROM products
    WHERE product_id = p_product_id;

    RETURN 1; -- Success
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 0; -- Failure
END;
/

-- Test in Oracle:
DECLARE
    l_name  products.product_name%TYPE;
    l_price products.unit_price%TYPE;
    l_stock products.units_in_stock%TYPE;
    l_status NUMBER;
BEGIN
    l_status := get_product_details_by_id(77, l_name, l_price, l_stock); -- Product: Original Frankfurter grüne Soße
    IF l_status = 1 THEN
        DBMS_OUTPUT.PUT_LINE('Oracle - Product: ' || l_name || ', Price: ' || l_price || ', Stock: ' || l_stock);
    ELSE
        DBMS_OUTPUT.PUT_LINE('Oracle - Product not found.');
    END IF;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL Function with OUT parameters
CREATE OR REPLACE FUNCTION get_product_details_by_id (
    p_product_id     products.product_id%TYPE,
    OUT p_product_name TEXT,             -- TEXT is common for variable length strings
    OUT p_unit_price   REAL,             -- REAL as per schema
    OUT p_units_in_stock SMALLINT      -- SMALLINT as per schema
)
RETURNS SETOF RECORD -- Or specifically: RETURNS TABLE(product_name TEXT, unit_price REAL, units_in_stock SMALLINT)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT product_name, unit_price, units_in_stock
    INTO p_product_name, p_unit_price, p_units_in_stock
    FROM products
    WHERE product_id = p_product_id;
    
    RETURN NEXT; -- Crucial for functions returning OUT parameters / SETOF
    RETURN; -- End of function results
END;
$$;

-- Test in PostgreSQL:
-- Calling a function with OUT parameters implicitly generates columns for its return.
SELECT p_product_name, p_unit_price, p_units_in_stock
FROM get_product_details_by_id(77); -- Product: Original Frankfurter grüne Soße

-- For multiple rows of different products in a setof function:
CREATE OR REPLACE FUNCTION get_products_by_category(
    p_category_id IN categories.category_id%TYPE
)
RETURNS TABLE (
    product_id_col SMALLINT,
    product_name_col VARCHAR(40),
    unit_price_col REAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT product_id, product_name, unit_price
    FROM products
    WHERE category_id = p_category_id;
END;
$$;

SELECT product_id_col, product_name_col, unit_price_col FROM get_products_by_category(1); -- Category: Beverages
```

**TL;DR:** `OUT`/`INOUT` parameters are syntactically similar but often require `RETURNS SETOF RECORD` or `RETURNS TABLE(...)` to explicitly define the "record" being returned in PostgreSQL functions. The `RETURN NEXT;` and `RETURN;` are fundamental for emitting results.

---

### Returning Multiple Rows (`RETURNS SETOF <type>` vs. Pipelined Functions, Implicit Cursors)

Oracle's `PIPELINED FUNCTION`s (used with `TABLE(...)` and `PIPE ROW`) and `REF CURSOR`s (explicit `OPEN...FETCH...CLOSE` or implicit `FOR` loops) are used to return sets of data. PostgreSQL's equivalent for returning sets of rows from a function is the `RETURNS SETOF` clause or `RETURNS TABLE(...)`.

| Oracle (PL/SQL)                                    | Postgres (PL/pgSQL)                                                    | Notes / Key Differences                                                                                                                                          |
| :------------------------------------------------- | :--------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `RETURN my_ref_cursor_var;` or `PIPE ROW(my_rec);` | `RETURN QUERY SELECT ...;` or `RETURN NEXT my_rec;`                    | `RETURN QUERY` is the most common and efficient for simply passing through a SQL query result set. `RETURN NEXT` is used for procedural construction of results. |
| Pipelined Functions                                | `RETURNS TABLE(...)`                                                   | Explicitly define column names and types returned in a table-like structure. Best practice for typed sets of results.                                            |
| `RETURNS SYS_REFCURSOR`                            | `RETURNS SETOF RECORD` (untyped) or `RETURNS SETOF table_name` (typed) | Typed results with `RETURNS SETOF table_name` provide better compile-time checks than untyped `SETOF RECORD`.                                                    |

**Example: Get Orders and Customer Info (Northwind `orders`, `customers`)**

Let's retrieve details for all orders placed after a specific date.

```sql
-- Oracle PL/SQL Pipelined Function (Simplified concept)
-- Often involves creating OBJECT types and TABLE types first
/*
CREATE TYPE order_info_obj AS OBJECT (
    order_id     NUMBER,
    order_date   DATE,
    company_name VARCHAR2(40)
);
/
CREATE TYPE order_info_tab AS TABLE OF order_info_obj;
/

CREATE OR REPLACE FUNCTION get_recent_orders (p_order_date_threshold DATE)
RETURN order_info_tab
PIPELINED
IS
BEGIN
    FOR rec IN (SELECT o.order_id, o.order_date, c.company_name
                FROM orders o JOIN customers c ON o.customer_id = c.customer_id
                WHERE o.order_date >= p_order_date_threshold
                ORDER BY o.order_date)
    LOOP
        PIPE ROW (order_info_obj(rec.order_id, rec.order_date, rec.company_name));
    END LOOP;
    RETURN;
END;
/

-- Test in Oracle:
SELECT * FROM TABLE(get_recent_orders(TO_DATE('1998-05-01', 'YYYY-MM-DD')));
*/

-- Oracle PL/SQL Function returning REF CURSOR (common pattern)
CREATE OR REPLACE FUNCTION get_recent_orders_ref_cursor (
    p_order_date_threshold IN DATE
) RETURN SYS_REFCURSOR IS
    v_ref_cursor SYS_REFCURSOR;
BEGIN
    OPEN v_ref_cursor FOR
        SELECT o.order_id, o.order_date, c.company_name
        FROM orders o JOIN customers c ON o.customer_id = c.customer_id
        WHERE o.order_date >= p_order_date_threshold
        ORDER BY o.order_date;
    RETURN v_ref_cursor;
END;
/

-- Test in Oracle (client specific):
-- In SQL Developer / SQL*Plus:
-- VAR RC REFCURSOR
-- EXEC :RC := get_recent_orders_ref_cursor(TO_DATE('1998-05-01', 'YYYY-MM-DD'));
-- PRINT RC;
```

```sql
-- PostgreSQL PL/pgSQL Function using RETURNS TABLE
CREATE OR REPLACE FUNCTION get_recent_orders_by_customer (
    p_customer_id customers.customer_id%TYPE
)
RETURNS TABLE (
    order_id        SMALLINT,
    order_date      DATE,
    shipped_date    DATE,
    total_freight   REAL -- Renaming for clarity if needed from freight
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT o.order_id, o.order_date, o.shipped_date, o.freight
    FROM orders o
    WHERE o.customer_id = p_customer_id
    ORDER BY o.order_date DESC; -- Order by latest orders first
END;
$$;

-- Test in PostgreSQL:
-- For Customer 'VINET' (order_id 10248 etc.)
SELECT order_id, order_date, shipped_date, total_freight FROM get_recent_orders_by_customer('VINET');
```

**TL;DR:** `RETURNS TABLE(...)` or `RETURNS SETOF <type>` are the canonical ways to return row sets from a function in PostgreSQL. `RETURN QUERY` is generally the most performant approach as it passes the query execution to the SQL engine. `RETURN NEXT` offers more fine-grained control for constructing the result set row by row procedurally.

---

### Transaction Behavior within Functions/Procedures

This is perhaps the most significant distinction in procedural language philosophy between Oracle and PostgreSQL.

| Oracle (PL/SQL)                                                                                                         | Postgres (PL/pgSQL)                                                                                                                         | Notes / Key Differences                                                                                                                                                                                       |
| :---------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Functions and Procedures can contain `COMMIT`, `ROLLBACK`, or DDL statements (DDL implies a `COMMIT` before and after). | **`CREATE FUNCTION` cannot contain `COMMIT`, `ROLLBACK`, or DDL.** This includes any DDL statements like `CREATE TABLE`, `DROP INDEX`, etc. | PostgreSQL functions run implicitly inside the *calling transaction*. They participate in and share the transactional state of the caller. Attempting transaction control statements will result in an error. |
| `PRAGMA AUTONOMOUS_TRANSACTION`                                                                                         | No direct equivalent.                                                                                                                       | Oracle's `AUTONOMOUS_TRANSACTION` allows sub-transactions independent of the main transaction. This is not natively available in PL/pgSQL. (Workarounds exist, e.g., dblink to same database).                |
| `CREATE PROCEDURE` (PG 11+)                                                                                             | **`CREATE PROCEDURE` allows `COMMIT`, `ROLLBACK`, and DDL.** (PG 11+ only)                                                                  | Procedures are specifically designed for this transactional independence. They must be invoked with `CALL`.                                                                                                   |

**Why this difference?**

*   **Functions as Query Building Blocks:** PostgreSQL's design envisions functions primarily as elements within SQL queries (like computed columns or WHERE clause predicates). To ensure transaction integrity, such query-embedded functions must not alter transaction state independently.
*   **Performance and Locking:** Oracle's approach can sometimes lead to implicit commits and DDL locking surprises, while PostgreSQL's stricter separation aims for more predictable transactional behavior.

**Example: Transactional Restrictions (Northwind concept)**

```sql
-- Oracle PL/SQL (Valid)
CREATE OR REPLACE PROCEDURE sample_transaction_oracle
IS
    v_new_shipper_id shippers.shipper_id%TYPE;
BEGIN
    INSERT INTO shippers (shipper_id, company_name, phone)
    VALUES (999, 'Temp Shipper Oracle', '555-1234');

    SELECT MAX(shipper_id) INTO v_new_shipper_id FROM shippers;

    -- DDL here implies an implicit commit before and after it.
    EXECUTE IMMEDIATE 'CREATE TABLE temp_oracle_log (message VARCHAR2(100))';
    INSERT INTO temp_oracle_log VALUES ('Shipper ' || v_new_shipper_id || ' created.');

    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Oracle: Transaction and DDL committed.');
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Oracle: Error and transaction rolled back.');
        RAISE;
END;
/
-- CALL sample_transaction_oracle();
```

```sql
-- PostgreSQL PL/pgSQL Function (INVALID - Will FAIL with error)
/*
CREATE OR REPLACE FUNCTION sample_transaction_function_pg()
RETURNS VOID -- For demonstration purposes, even if it has DML, functions are designed not to alter tx.
LANGUAGE plpgsql
AS $$
DECLARE
    v_new_shipper_id shippers.shipper_id%TYPE;
BEGIN
    INSERT INTO shippers (shipper_id, company_name, phone)
    VALUES (998, 'Temp Shipper PG-Func', '555-5678');
    
    SELECT MAX(shipper_id) INTO v_new_shipper_id FROM shippers;

    -- This will raise an error: "cannot have DDL or transaction control commands in a function"
    EXECUTE 'CREATE TABLE temp_pg_func_log (message TEXT)';
    
    -- This would also fail if EXECUTE was not used to work around error for DDL
    -- INSERT INTO temp_pg_func_log VALUES ('Shipper ' || v_new_shipper_id || ' created (func).');
    
    COMMIT; -- This will also raise an error: "transaction control statements are not allowed in a user-defined function"
    RAISE NOTICE 'PG Function: This line will likely not be reached if COMMIT/DDL are present.';
END;
$$;
-- SELECT sample_transaction_function_pg(); -- Will fail.
*/

-- PostgreSQL PL/pgSQL Procedure (VALID - PG 11+)
CREATE OR REPLACE PROCEDURE sample_transaction_procedure_pg()
LANGUAGE plpgsql
AS $$
DECLARE
    v_new_shipper_id shippers.shipper_id%TYPE;
BEGIN
    INSERT INTO shippers (shipper_id, company_name, phone)
    VALUES (997, 'Temp Shipper PG-Proc', '555-9012');
    
    SELECT MAX(shipper_id) INTO v_new_shipper_id FROM shippers;

    -- DDL is allowed here
    EXECUTE 'CREATE TABLE IF NOT EXISTS temp_pg_proc_log (log_message TEXT)'; -- Use IF NOT EXISTS for re-runnability
    INSERT INTO temp_pg_proc_log (log_message) VALUES ('Shipper ' || v_new_shipper_id || ' created (proc).');
    
    COMMIT; -- Allowed
    RAISE NOTICE 'PG Procedure: Transaction and DDL committed.';
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE EXCEPTION 'PG Procedure: Error and transaction rolled back: %', SQLERRM;
END;
$$;
-- CALL sample_transaction_procedure_pg();
-- Then you can check: SELECT * FROM shippers WHERE shipper_id=997; SELECT * FROM temp_pg_proc_log;
-- You might also try: DROP PROCEDURE sample_transaction_procedure_pg(); DROP TABLE temp_pg_proc_log;
```

**Critical Takeaway:** This is arguably the most critical difference for Oracle developers. **Never attempt `COMMIT`/`ROLLBACK` or DDL inside a PostgreSQL `CREATE FUNCTION`.** For such logic, either use `DO $$ ... $$;` blocks for ad-hoc scripts or, for persistent transactional units (PG 11+), explicitly create a `PROCEDURE` and invoke it with `CALL`. This requires re-evaluating the transaction boundaries in migrated PL/SQL code.