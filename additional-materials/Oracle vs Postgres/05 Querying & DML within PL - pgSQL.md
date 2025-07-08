## Querying & DML within PL/pgSQL

Procedural extensions in both Oracle and PostgreSQL frequently interact with data using standard SQL queries and Data Manipulation Language (DML) statements. While the core SQL syntax is largely identical, the PL/pgSQL environment introduces specific constructs for host variable interaction, result set handling, and dynamic execution.

### Fetching Single Rows (`SELECT ... INTO ...`)

Both Oracle PL/SQL and PostgreSQL PL/pgSQL employ the `SELECT ... INTO ...` statement to retrieve a single row's column values directly into declared local variables.

| Oracle (PL/SQL)                                         | Postgres (PL/pgSQL)                                                                                      | Notes / Key Differences                                                                 |
| :------------------------------------------------------ | :------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------- |
| `SELECT col1, col2 INTO var1, var2 FROM tab WHERE ...;` | `SELECT col1, col2 INTO var1, var2 FROM tab WHERE ...;`                                                  | Identical basic syntax.                                                                 |
| Raises `NO_DATA_FOUND` or `TOO_MANY_ROWS` on error.     | Sets `var` to `NULL` (and `FOUND` to false) if no row. Raises `TOO_MANY_ROWS` if multiple rows returned. | Explicitly check `FOUND` for no data. `STRICT` can force `NO_DATA_FOUND`-like behavior. |

**Example: Fetching Customer Contact Information (Northwind `customers`)**

Let's retrieve the `contact_name` and `contact_title` for a specific customer.

```sql
-- Oracle PL/SQL
DECLARE
    v_customer_id    customers.customer_id%TYPE := 'ANTON'; -- Antonio Moreno
    v_contact_name   customers.contact_name%TYPE;
    v_contact_title  customers.contact_title%TYPE;
BEGIN
    SELECT contact_name, contact_title
    INTO v_contact_name, v_contact_title
    FROM customers
    WHERE customer_id = v_customer_id;

    DBMS_OUTPUT.PUT_LINE('Oracle: Contact for ' || v_customer_id || ': ' || v_contact_name || ' (' || v_contact_title || ')');

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Oracle: Customer ' || v_customer_id || ' not found.');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('Oracle: Multiple customers found for ' || v_customer_id || ' (unexpected).');
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    v_customer_id    customers.customer_id%TYPE := 'ANTON'; -- Antonio Moreno
    v_contact_name   customers.contact_name%TYPE;
    v_contact_title  customers.contact_title%TYPE;
BEGIN
    SELECT contact_name, contact_title
    INTO v_contact_name, v_contact_title
    FROM customers
    WHERE customer_id = v_customer_id;

    IF NOT FOUND THEN
        RAISE NOTICE 'Postgres: Customer % not found.', v_customer_id;
    ELSE
        RAISE NOTICE 'Postgres: Contact for %: % (%)', v_customer_id, v_contact_name, v_contact_title;
    END IF;

EXCEPTION
    WHEN TOO_MANY_ROWS THEN
        RAISE NOTICE 'Postgres: Multiple customers found for % (unexpected).', v_customer_id;
        -- Use SELECT INTO STRICT ... to force a NO_DATA_FOUND-like error on zero rows:
        -- SELECT contact_name, contact_title INTO STRICT v_contact_name, v_contact_title FROM customers WHERE customer_id = 'NON_EXISTENT';
END $$;
```

**TL;DR:** While syntax is identical, remember to check `FOUND` after a `SELECT ... INTO ...` in PL/pgSQL to handle cases where no row is returned, or use `STRICT` for strict "must return one row" behavior similar to Oracle.

---

### Iterating Result Sets (Cursor-`FOR` loop preferred over explicit cursors generally)

For iterating over multiple rows, the cursor `FOR` loop is the recommended and most idiomatic construct in both Oracle and PostgreSQL. It abstracts away explicit cursor declaration, opening, fetching, and closing, leading to cleaner code.

| Oracle (PL/SQL)                                                                | Postgres (PL/pgSQL)                                | Notes / Key Differences                                                                                                                           |
| :----------------------------------------------------------------------------- | :------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| `FOR record_name IN (SELECT ...) LOOP ... END LOOP;`                           | `FOR record_name IN SELECT ... LOOP ... END LOOP;` | Direct syntax and semantic equivalent. The `record_name` variable is implicitly declared as a `RECORD` (PostgreSQL) or anonymous record (Oracle). |
| Explicit cursors (`CURSOR c IS SELECT...; OPEN c; FETCH c INTO ...; CLOSE c;`) | Explicit cursors (same pattern)                    | While supported, they are generally less common for simple iteration due to the cursor `FOR` loop's convenience.                                  |

**Example: Listing Orders for a Customer (Northwind `orders`, `order_details`)**

Let's list all orders and their associated details for a customer with many orders, like 'ERNSH' (Ernst Handel).

```sql
-- Oracle PL/SQL (Implicit Cursor FOR Loop)
DECLARE
    v_customer_id  orders.customer_id%TYPE := 'ERNSH'; -- Ernst Handel
BEGIN
    DBMS_OUTPUT.PUT_LINE('Oracle: Listing orders and details for customer ' || v_customer_id);
    
    FOR r_order IN (SELECT order_id, order_date, shipped_date, freight
                    FROM orders
                    WHERE customer_id = v_customer_id
                    ORDER BY order_id DESC)
    LOOP
        DBMS_OUTPUT.PUT_LINE('  Order ID: ' || r_order.order_id || ' Date: ' || TO_CHAR(r_order.order_date, 'YYYY-MM-DD'));
        FOR r_detail IN (SELECT product_id, quantity, unit_price, discount
                         FROM order_details
                         WHERE order_id = r_order.order_id)
        LOOP
            DBMS_OUTPUT.PUT_LINE('    - Product ' || r_detail.product_id || ': ' ||
                                 r_detail.quantity || ' @ ' || r_detail.unit_price ||
                                 ' (Disc: ' || r_detail.discount * 100 || '%)');
        END LOOP;
    END LOOP;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL (Cursor FOR Loop)
DO $$
DECLARE
    v_customer_id  orders.customer_id%TYPE := 'ERNSH'; -- Ernst Handel
    r_order        RECORD; -- Implicitly declared record for order iteration
    r_detail       RECORD; -- Implicitly declared record for order_details iteration
BEGIN
    RAISE NOTICE 'Postgres: Listing orders and details for customer %', v_customer_id;
    
    FOR r_order IN SELECT order_id, order_date, shipped_date, freight
                   FROM orders
                   WHERE customer_id = v_customer_id
                   ORDER BY order_id DESC
    LOOP
        RAISE NOTICE '  Order ID: % Date: %', r_order.order_id, r_order.order_date;
        FOR r_detail IN SELECT product_id, quantity, unit_price, discount
                        FROM order_details
                        WHERE order_id = r_order.order_id
        LOOP
            RAISE NOTICE '    - Product %: % @ % (Disc: %%)',
                         r_detail.product_id, r_detail.quantity, r_detail.unit_price, (r_detail.discount * 100);
        END LOOP;
    END LOOP;
END $$;
```

**TL;DR:** The `FOR IN SELECT` loop offers the most streamlined approach to iterate over query results in PL/pgSQL, directly comparable to its Oracle counterpart. This is often the preferred method due to its efficiency and reduced boilerplate code.

---

### Dynamic SQL (`EXECUTE IMMEDIATE` vs. `EXECUTE`)

Both environments provide a mechanism for executing SQL statements constructed as strings at runtime (dynamic SQL). This is useful for DDL, parameterized queries where structure varies, or queries with dynamic `ORDER BY`/`WHERE` clauses.

| Oracle (PL/SQL)                                    | Postgres (PL/pgSQL)                      | Notes / Key Differences                                                              |
| :------------------------------------------------- | :--------------------------------------- | :----------------------------------------------------------------------------------- |
| `EXECUTE IMMEDIATE sql_string;`                    | `EXECUTE sql_string;`                    | `EXECUTE` is the direct equivalent.                                                  |
| `EXECUTE IMMEDIATE sql_string INTO ... USING ...;` | `EXECUTE sql_string INTO ... USING ...;` | Parameters must be explicitly passed using `USING`, and output assigned with `INTO`. |

**Example: Dynamic Product Search (Northwind `products`)**

Let's construct a dynamic query to find products based on partial name and optional minimum stock level.

```sql
-- Oracle PL/SQL
DECLARE
    p_product_name_pattern products.product_name%TYPE := 'Cha%'; -- Chai, Chang
    p_min_stock_level      products.units_in_stock%TYPE := 30;
    
    v_sql_stmt             VARCHAR2(500);
    v_product_name         products.product_name%TYPE;
    v_unit_price           products.unit_price%TYPE;
BEGIN
    v_sql_stmt := 'SELECT product_name, unit_price FROM products WHERE product_name LIKE :name_pat';
    IF p_min_stock_level IS NOT NULL THEN
        v_sql_stmt := v_sql_stmt || ' AND units_in_stock >= :min_stock';
    END IF;
    
    DBMS_OUTPUT.PUT_LINE('Oracle: Dynamic Query: ' || v_sql_stmt);
    DBMS_OUTPUT.PUT_LINE('--- Results ---');

    IF p_min_stock_level IS NOT NULL THEN
        FOR rec IN (EXECUTE IMMEDIATE v_sql_stmt USING p_product_name_pattern, p_min_stock_level) LOOP
            v_product_name := rec.product_name;
            v_unit_price := rec.unit_price;
            DBMS_OUTPUT.PUT_LINE('  ' || v_product_name || ' (Price: ' || v_unit_price || ')');
        END LOOP;
    ELSE
        -- Without a strong reference, sometimes direct INTO usage or complex cursor loop needed for different # of USING clauses.
        -- Simpler way often for single value fetch:
        -- EXECUTE IMMEDIATE v_sql_stmt INTO v_product_name, v_unit_price USING p_product_name_pattern; -- This would require exact matches
        -- For multi-row: Requires dynamic cursor open/fetch, or different approach.
        DBMS_OUTPUT.PUT_LINE('Oracle: Cannot demonstrate multi-row dynamic query iteration without knowing parameters a priori via EXECUTE IMMEDIATE. Usually done with dynamic REF CURSOR.');
    END IF;

    -- Example for a simple SELECT INTO dynamic execution:
    -- SELECT contact_name INTO v_contact FROM CUSTOMERS WHERE customer_id = 'ALFKI';
    EXECUTE IMMEDIATE 'SELECT product_name FROM products WHERE product_id = :id' INTO v_product_name USING 1;
    DBMS_OUTPUT.PUT_LINE('Static query test via Dynamic: Product 1 is ' || v_product_name);
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    p_product_name_pattern products.product_name%TYPE := 'Cha%'; -- Chai, Chang
    p_min_stock_level      products.units_in_stock%TYPE := 30;
    
    v_sql_stmt             TEXT;
    r_product              RECORD; -- For iterating results
BEGIN
    v_sql_stmt := 'SELECT product_name, unit_price FROM products WHERE product_name ILIKE $1'; -- Use $1, $2, etc. placeholders
    IF p_min_stock_level IS NOT NULL THEN
        v_sql_stmt := v_sql_stmt || ' AND units_in_stock >= $2';
    END IF;
    
    RAISE NOTICE 'Postgres: Dynamic Query: %', v_sql_stmt;
    RAISE NOTICE '--- Results ---';

    -- Use EXECUTE within a FOR loop for multi-row dynamic query iteration
    IF p_min_stock_level IS NOT NULL THEN
        FOR r_product IN EXECUTE v_sql_stmt USING p_product_name_pattern, p_min_stock_level LOOP
            RAISE NOTICE '  % (Price: %)', r_product.product_name, r_product.unit_price;
        END LOOP;
    ELSE
        -- No min_stock_level filter
        FOR r_product IN EXECUTE v_sql_stmt USING p_product_name_pattern LOOP
            RAISE NOTICE '  % (Price: %)', r_product.product_name, r_product.unit_price;
        END LOOP;
    END IF;
END $$;
```

**TL;DR:** `EXECUTE` is the command for dynamic SQL in PL/pgSQL. For queries returning results, combine `EXECUTE` with `FOR record_name IN EXECUTE sql_string USING ... LOOP` for robust multi-row processing. This allows binding values and iterating through the results, which is a significant advantage for dynamic data retrieval.

---

### Placeholder Syntax (`USING` clause vs. `:bind_var`)

When building dynamic SQL with parameters, the syntax for placeholders differs significantly.

| Oracle (PL/SQL)       | Postgres (PL/pgSQL)                        | Notes / Key Differences                                                                     |
| :-------------------- | :----------------------------------------- | :------------------------------------------------------------------------------------------ |
| `WHERE col = :my_var` | `WHERE col = $1` or `WHERE col = $2`, etc. | PostgreSQL uses positional parameters ($1, $2, etc.) for `EXECUTE` and prepares statements. |
|                       | `USING var1, var2`                         | Both bind methods use `USING` clause after the query string.                                |

**Example (demonstrated in the Dynamic SQL section above):**
PostgreSQL uses `$1`, `$2`, etc. as positional placeholders which are then mapped to variables listed in the `USING` clause. Oracle uses named bind variables (e.g., `:my_var`).

**Watch Out For:** When dynamically executing DML, remember to use positional parameters `$1, $2, ...` and list the corresponding variables in the `USING` clause.

---

### DML (`INSERT`, `UPDATE`, `DELETE`)

Standard `INSERT`, `UPDATE`, and `DELETE` statements work as expected within PL/pgSQL, very similar to PL/SQL. Transaction handling depends on whether the DML is executed in a function or a procedure (as discussed in Chapter IV).

| Oracle (PL/SQL)                       | Postgres (PL/pgSQL)                   | Notes / Key Differences                                                                                                                                                                                                                                  |
| :------------------------------------ | :------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `INSERT INTO tab (...) VALUES (...);` | `INSERT INTO tab (...) VALUES (...);` | Identical syntax.                                                                                                                                                                                                                                        |
| `UPDATE tab SET col = val WHERE ...;` | `UPDATE tab SET col = val WHERE ...;` | Identical syntax.                                                                                                                                                                                                                                        |
| `DELETE FROM tab WHERE ...;`          | `DELETE FROM tab WHERE ...;`          | Identical syntax.                                                                                                                                                                                                                                        |
| No equivalent                         | `FOUND` / `NOT FOUND` after DML       | After DML operations (INSERT/UPDATE/DELETE), the global `FOUND` variable is set (`TRUE` if at least one row affected, `FALSE` otherwise). This is valuable for status checks. `GET DIAGNOSTICS variable = ROW_COUNT;` provides exact row count affected. |

**Example: Performing DML (Northwind `shippers` and `products`)**

```sql
-- Oracle PL/SQL DML example
DECLARE
    v_shipper_id      shippers.shipper_id%TYPE := 7; -- Assuming next ID not already used
    v_product_id      products.product_id%TYPE := 1;
    v_old_stock       products.units_in_stock%TYPE;
    v_affected_rows   NUMBER;
BEGIN
    DBMS_OUTPUT.PUT_LINE('Oracle: Starting DML Operations');

    -- INSERT
    INSERT INTO shippers (shipper_id, company_name, phone)
    VALUES (v_shipper_id, 'My New Shipping Co', '1-800-TEST-SHP');
    v_affected_rows := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('  Inserted ' || v_affected_rows || ' row into shippers.');

    -- UPDATE
    SELECT units_in_stock INTO v_old_stock FROM products WHERE product_id = v_product_id;
    UPDATE products
    SET units_in_stock = v_old_stock - 5 -- Reduce stock by 5
    WHERE product_id = v_product_id;
    v_affected_rows := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('  Updated ' || v_affected_rows || ' row in products. Old stock: ' || v_old_stock || ', New stock: ' || (v_old_stock - 5));

    -- DELETE
    DELETE FROM shippers WHERE shipper_id = v_shipper_id;
    v_affected_rows := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('  Deleted ' || v_affected_rows || ' row from shippers.');

    COMMIT; -- Explicit commit required
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Oracle: DML failed and rolled back: ' || SQLERRM);
END;
/
```

```sql
-- PostgreSQL PL/pgSQL DML example
DO $$
DECLARE
    v_shipper_id      shippers.shipper_id%TYPE := 7; -- Assuming next ID not already used
    v_product_id      products.product_id%TYPE := 1;
    v_old_stock       products.units_in_stock%TYPE;
    v_row_count       INTEGER; -- For GET DIAGNOSTICS
BEGIN
    RAISE NOTICE 'Postgres: Starting DML Operations';

    -- INSERT
    INSERT INTO shippers (shipper_id, company_name, phone)
    VALUES (v_shipper_id, 'My New Shipping Co', '1-800-TEST-SHP');
    GET DIAGNOSTICS v_row_count = ROW_COUNT;
    RAISE NOTICE '  Inserted % row(s) into shippers.', v_row_count;

    -- UPDATE
    SELECT units_in_stock INTO v_old_stock FROM products WHERE product_id = v_product_id;
    UPDATE products
    SET units_in_stock = v_old_stock - 5 -- Reduce stock by 5
    WHERE product_id = v_product_id;
    GET DIAGNOSTICS v_row_count = ROW_COUNT;
    RAISE NOTICE '  Updated % row(s) in products. Old stock: %, New stock: %', v_row_count, v_old_stock, (v_old_stock - 5);

    -- DELETE
    DELETE FROM shippers WHERE shipper_id = v_shipper_id;
    GET DIAGNOSTICS v_row_count = ROW_COUNT;
    RAISE NOTICE '  Deleted % row(s) from shippers.', v_row_count;

    -- Note: This is a DO block, so it's auto-committed if no explicit COMMIT/ROLLBACK within and no errors.
    -- For stored PROCEDURES, explicit COMMIT; might be present as shown in Chapter IV.
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE NOTICE 'Postgres: DML failed and rolled back: %', SQLERRM;
END $$;
```

**TL;DR:** DML statements (`INSERT`, `UPDATE`, `DELETE`) are nearly identical. Use `GET DIAGNOSTICS var = ROW_COUNT;` in PostgreSQL for the number of rows affected by the preceding DML statement, which maps directly to Oracle's `SQL%ROWCOUNT`.

---

### `RETURNING` Clause (Similar to `RETURNING INTO`)

Both Oracle and PostgreSQL provide a `RETURNING` clause (or `RETURNING INTO` in Oracle's PL/SQL context) to retrieve values of the affected row(s) after `INSERT`, `UPDATE`, or `DELETE` statements.

| Oracle (PL/SQL)                                                  | Postgres (PL/pgSQL)                                              | Notes / Key Differences                                                     |
| :--------------------------------------------------------------- | :--------------------------------------------------------------- | :-------------------------------------------------------------------------- |
| `INSERT/UPDATE/DELETE ... RETURNING col1, col2 INTO var1, var2;` | `INSERT/UPDATE/DELETE ... RETURNING col1, col2 INTO var1, var2;` | Syntax is largely identical and behavior matches for single row operations. |

**Example: Get ID/Updated Values on DML (Northwind `products`)**

Let's insert a new temporary category, update a product's stock and retrieve old/new stock, and then delete an order and get its details.

```sql
-- Oracle PL/SQL RETURNING INTO example
DECLARE
    v_new_category_id   categories.category_id%TYPE := 99; -- Temporary ID
    v_new_category_name categories.category_name%TYPE := 'Experimental Category';

    v_product_id        products.product_id%TYPE := 1;
    v_old_stock         products.units_in_stock%TYPE;
    v_new_stock         products.units_in_stock%TYPE;

    v_order_to_delete   orders.order_id%TYPE := 10248;
    v_deleted_cust_id   orders.customer_id%TYPE;
BEGIN
    DBMS_OUTPUT.PUT_LINE('Oracle: Demonstrating RETURNING INTO');

    -- INSERT with RETURNING INTO
    INSERT INTO categories (category_id, category_name)
    VALUES (v_new_category_id, v_new_category_name)
    RETURNING category_id INTO v_new_category_id; -- Return actual generated ID if using sequence.
    DBMS_OUTPUT.PUT_LINE('  Inserted new category with ID: ' || v_new_category_id);

    -- UPDATE with RETURNING INTO (new and old values)
    UPDATE products
    SET units_in_stock = units_in_stock - 2
    WHERE product_id = v_product_id
    RETURNING OLD units_in_stock, NEW units_in_stock INTO v_old_stock, v_new_stock; -- Oracle-specific syntax OLD/NEW for compound triggers also possible
    DBMS_OUTPUT.PUT_LINE('  Updated product ' || v_product_id || '. Old Stock: ' || v_old_stock || ', New Stock: ' || v_new_stock);

    -- DELETE with RETURNING INTO
    DELETE FROM orders
    WHERE order_id = v_order_to_delete
    RETURNING customer_id INTO v_deleted_cust_id;
    DBMS_OUTPUT.PUT_LINE('  Deleted order ' || v_order_to_delete || ' from customer: ' || v_deleted_cust_id);
    
    -- Clean up inserted category
    DELETE FROM categories WHERE category_id = v_new_category_id;
    DBMS_OUTPUT.PUT_LINE('  Cleaned up temporary category ' || v_new_category_id);

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Oracle: RETURNING INTO DML failed: ' || SQLERRM);
END;
/
```

```sql
-- PostgreSQL PL/pgSQL RETURNING INTO example
DO $$
DECLARE
    v_new_category_id   categories.category_id%TYPE := 9; -- Use ID 9, to not clash, if already used next to default range. Categories 1-8 are in DB
    v_new_category_name categories.category_name%TYPE := 'Experimental Category PG';

    v_product_id        products.product_id%TYPE := 1;
    v_old_stock         products.units_in_stock%TYPE;
    v_new_stock         products.units_in_stock%TYPE;

    v_order_to_delete   orders.order_id%TYPE := 10248; -- This order has details, which is FK dependent. DELETE will fail due to FK if CASCADE not defined.
                                                       -- For safer demo, let's pick an ID not involved in other table like suppliers (ID 997, 998, 999 above example).
    v_shipper_to_delete shippers.shipper_id%TYPE := 7;
    v_deleted_comp_name shippers.company_name%TYPE;
BEGIN
    RAISE NOTICE 'Postgres: Demonstrating RETURNING INTO';

    -- INSERT with RETURNING INTO
    INSERT INTO categories (category_id, category_name) -- Using fixed ID here as there's no auto-sequence by default
    VALUES (v_new_category_id, v_new_category_name)
    RETURNING category_id INTO v_new_category_id;
    RAISE NOTICE '  Inserted new category with ID: %', v_new_category_id;

    -- UPDATE with RETURNING INTO (PostgreSQL cannot directly get OLD and NEW values in single RETURNING. Usually uses before/after query or a trigger.)
    -- For simplicity, let's get just the new value or assume old value from pre-query
    SELECT units_in_stock INTO v_old_stock FROM products WHERE product_id = v_product_id; -- Get old value manually
    UPDATE products
    SET units_in_stock = v_old_stock - 2
    WHERE product_id = v_product_id
    RETURNING units_in_stock INTO v_new_stock; -- Return new value only
    RAISE NOTICE '  Updated product %. Old Stock: %, New Stock: %', v_product_id, v_old_stock, v_new_stock;

    -- Ensure we have a shipper ID to delete from prior example
    INSERT INTO shippers (shipper_id, company_name, phone) VALUES (v_shipper_to_delete, 'Temp Ship Co for Returning', '555-DEL');

    -- DELETE with RETURNING INTO (for single row delete)
    DELETE FROM shippers
    WHERE shipper_id = v_shipper_to_delete
    RETURNING company_name INTO v_deleted_comp_name;
    RAISE NOTICE '  Deleted shipper % (Company: %).', v_shipper_to_delete, v_deleted_comp_name;

    -- Important: Remember to roll back any persistent DML in DO blocks for consistent environment.
    ROLLBACK;
    RAISE NOTICE '  Postgres: DML operations rolled back for demo isolation.';
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE EXCEPTION 'Postgres: RETURNING INTO DML failed and rolled back: %', SQLERRM;
END $$;
```

**Watch Out For: Retrieving `OLD` and `NEW` in a single `RETURNING` statement on `UPDATE`.** PostgreSQL's `RETURNING` clause directly gives you the *final state* of columns. Obtaining both `OLD` and `NEW` values from a single `UPDATE ... RETURNING` is not natively supported like Oracle's `OLD/NEW` for `RETURNING INTO` unless used in contexts like triggers (where `OLD` and `NEW` refer to special row variables) or by performing an initial `SELECT` to get the old value before the `UPDATE`.

---