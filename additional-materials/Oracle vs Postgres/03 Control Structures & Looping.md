## Control Structures & Looping

Procedural flow control in PL/pgSQL will feel familiar to PL/SQL developers. PostgreSQL offers `IF`/`CASE` statements for conditional logic and a variety of loop constructs. While syntax nuances exist, the underlying concepts for controlling program flow are largely consistent.

### Conditional Logic (`IF...ELSIF...ELSE...END IF;`)

Both PL/SQL and PL/pgSQL utilize the `IF...ELSIF...ELSE...END IF;` structure for conditional execution. The primary syntactic distinction is `THEN` vs. `THEN` (present in both) and the terminal `END IF;` (PL/pgSQL) vs. `END IF;` (PL/SQL).

| Oracle (PL/SQL)                              | Postgres (PL/pgSQL)                          | Notes / Key Differences                                         |
| :------------------------------------------- | :------------------------------------------- | :-------------------------------------------------------------- |
| `IF ... THEN ... ELSIF ... ELSE ... END IF;` | `IF ... THEN ... ELSIF ... ELSE ... END IF;` | Identical keywords, though capitalization conventions may vary. |

**Example: Classifying Products by Price (Northwind `products` table)**

Let's classify Northwind products based on their unit price.

```sql
-- Oracle PL/SQL
DECLARE
    p_product_id  products.product_id%TYPE := 77; -- Original Frankfurter grüne Soße
    p_unit_price  products.unit_price%TYPE;
    v_category    VARCHAR2(20);
BEGIN
    SELECT unit_price INTO p_unit_price
    FROM products
    WHERE product_id = p_product_id;

    IF p_unit_price < 10 THEN
        v_category := 'Budget';
    ELSIF p_unit_price BETWEEN 10 AND 50 THEN
        v_category := 'Standard';
    ELSE
        v_category := 'Premium';
    END IF;

    DBMS_OUTPUT.PUT_LINE('Product ' || p_product_id || ' (Price: ' || p_unit_price || ') is a ' || v_category || ' product.');

    -- Test another product
    p_product_id := 9; -- Mishi Kobe Niku
    SELECT unit_price INTO p_unit_price FROM products WHERE product_id = p_product_id;
    IF p_unit_price < 10 THEN v_category := 'Budget';
    ELSIF p_unit_price BETWEEN 10 AND 50 THEN v_category := 'Standard';
    ELSE v_category := 'Premium'; END IF;
    DBMS_OUTPUT.PUT_LINE('Product ' || p_product_id || ' (Price: ' || p_unit_price || ') is a ' || v_category || ' product.');
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    p_product_id  products.product_id%TYPE := 77; -- Original Frankfurter grüne Soße
    p_unit_price  products.unit_price%TYPE;
    v_category    TEXT;
BEGIN
    SELECT unit_price INTO p_unit_price
    FROM products
    WHERE product_id = p_product_id;

    IF p_unit_price < 10 THEN
        v_category := 'Budget';
    ELSIF p_unit_price BETWEEN 10 AND 50 THEN -- Note: Postgres REAL type has float precision, direct equality might be problematic, but BETWEEN works for ranges
        v_category := 'Standard';
    ELSE
        v_category := 'Premium';
    END IF;

    RAISE NOTICE 'Product % (Price: %) is a % product.', p_product_id, p_unit_price, v_category;

    -- Test another product
    p_product_id := 9; -- Mishi Kobe Niku
    SELECT unit_price INTO p_unit_price FROM products WHERE product_id = p_product_id;
    IF p_unit_price < 10 THEN v_category := 'Budget';
    ELSIF p_unit_price BETWEEN 10 AND 50 THEN v_category := 'Standard';
    ELSE v_category := 'Premium'; END IF;
    RAISE NOTICE 'Product % (Price: %) is a % product.', p_product_id, p_unit_price, v_category;
END $$;
```

**TL;DR:** `IF` statements are highly portable in syntax between PL/SQL and PL/pgSQL, allowing for direct translation of conditional logic.

---

### Simple Loops (`LOOP...EXIT...END LOOP;`)

The fundamental `LOOP` construct with an `EXIT WHEN` condition is structurally identical across both languages. This loop type is useful when the number of iterations is not predetermined or relies on an internal condition being met.

| Oracle (PL/SQL)                    | Postgres (PL/pgSQL)                | Notes / Key Differences                  |
| :--------------------------------- | :--------------------------------- | :--------------------------------------- |
| `LOOP ... EXIT WHEN ... END LOOP;` | `LOOP ... EXIT WHEN ... END LOOP;` | Identical syntax for basic loop control. |

**Example: Processing Customer Names until a Condition (Northwind `customers` table)**

Let's simulate processing a list of customers and stop after a certain number or condition. For demonstration, we'll iterate through company names from customers until we find one starting with 'Q'.

```sql
-- Oracle PL/SQL
DECLARE
    v_current_id  customers.customer_id%TYPE := 'ALFKI'; -- Starting from this ID, assuming alphabetical order or other mechanism for next.
    v_company_name customers.company_name%TYPE;
    v_loop_count NUMBER := 0;
BEGIN
    LOOP
        SELECT company_name INTO v_company_name
        FROM customers
        WHERE customer_id = v_current_id; -- Simplistic "next ID" logic here; ideally using proper cursors or ROWNUM

        DBMS_OUTPUT.PUT_LINE('Processing customer: ' || v_company_name);
        v_loop_count := v_loop_count + 1;

        IF SUBSTR(v_company_name, 1, 1) = 'Q' THEN
            DBMS_OUTPUT.PUT_LINE('Found company starting with Q! Exiting after ' || v_loop_count || ' iterations.');
            EXIT;
        END IF;

        -- For real data, we'd fetch the "next" customer. For this illustrative example, just break after a few
        IF v_loop_count >= 10 THEN -- Limit for demo to prevent infinite loop
             DBMS_OUTPUT.PUT_LINE('Exceeded loop count limit for demo.');
             EXIT;
        END IF;
        
        -- Simple way to get next company_name for demonstration purpose if it was sorted
        -- Note: this is NOT a robust way to iterate table in actual code
        SELECT customer_id INTO v_current_id FROM customers WHERE company_name > v_company_name ORDER BY company_name ASC FETCH FIRST 1 ROW ONLY;

    END LOOP;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    v_company_name customers.company_name%TYPE;
    v_loop_count   INTEGER := 0;
    -- A cursor is more appropriate for reliable iteration, but for simple LOOP example:
    -- We'll use a FOR loop with LIMIT and OFFSET for this demo to simulate progression.
    -- This is a very simplified example as proper single-row fetch requires order and unique condition
BEGIN
    RAISE NOTICE '--- Simple Loop Example ---';
    FOR i IN 1..20 LOOP -- Arbitrarily iterate over the first 20 customers by company_name alphabetically
        SELECT company_name
        INTO v_company_name
        FROM customers
        ORDER BY company_name
        OFFSET i - 1 -- Adjust offset for 0-based to 1-based loop
        LIMIT 1;

        EXIT WHEN NOT FOUND; -- If no row is returned by SELECT INTO, v_company_name will be NULL, and NOT FOUND will be true.

        RAISE NOTICE 'Processing customer (%): %', i, v_company_name;
        v_loop_count := v_loop_count + 1;

        IF SUBSTRING(v_company_name, 1, 1) = 'Q' THEN
            RAISE NOTICE 'Found company starting with Q! Exiting after % iterations.', v_loop_count;
            EXIT;
        END IF;
    END LOOP;
    IF v_loop_count = 0 THEN
        RAISE NOTICE 'No customers found or processed within loop limit.';
    END IF;
END $$;
```

**TL;DR:** The `LOOP...EXIT WHEN...END LOOP;` pattern is directly translatable. Be mindful of how you acquire "next" values when converting Oracle `SELECT INTO` within a simple loop if the original implicitly relied on sequential access of some kind (which is generally discouraged as not portable SQL anyway). `NOT FOUND` is Postgres' equivalent for checking if `SELECT INTO` returned rows.

---

### `FOR` Loops

PL/pgSQL provides robust `FOR` loop constructs including numeric ranges, iterating over query results (cursor loops), and `FOREACH` for arrays.

#### Numeric `FOR` Loop

Syntax is almost identical.

| Oracle (PL/SQL)                               | Postgres (PL/pgSQL)                           | Notes / Key Differences                                              |
| :-------------------------------------------- | :-------------------------------------------- | :------------------------------------------------------------------- |
| `FOR counter IN low..high LOOP ... END LOOP;` | `FOR counter IN low..high LOOP ... END LOOP;` | Identical syntax. `REVERSE` keyword also available for decrementing. |

**Example: Discount Iteration (Northwind product_ids)**

```sql
-- Oracle PL/SQL
DECLARE
    product_id_start products.product_id%TYPE := 70; -- Outback Lager
    product_id_end   products.product_id%TYPE := 77; -- Original Frankfurter grüne Soße
    new_price        products.unit_price%TYPE;
    original_price   products.unit_price%TYPE;
BEGIN
    DBMS_OUTPUT.PUT_LINE('--- Applying discount to Products 70-77 ---');
    FOR p_id IN product_id_start..product_id_end LOOP
        SELECT unit_price INTO original_price
        FROM products
        WHERE product_id = p_id;

        IF original_price IS NOT NULL THEN
            new_price := original_price * 0.95; -- 5% discount
            DBMS_OUTPUT.PUT_LINE('Product ' || p_id || ': Old Price = ' || original_price || ', New Price = ' || new_price);
            -- UPDATE products SET unit_price = new_price WHERE product_id = p_id; -- (If actually performing DML)
        ELSE
            DBMS_OUTPUT.PUT_LINE('Product ' || p_id || ': Not found or price is NULL.');
        END IF;
    END LOOP;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    product_id_start products.product_id%TYPE := 70; -- Outback Lager
    product_id_end   products.product_id%TYPE := 77; -- Original Frankfurter grüne Soße
    new_price        products.unit_price%TYPE;
    original_price   products.unit_price%TYPE;
BEGIN
    RAISE NOTICE '--- Applying discount to Products 70-77 ---';
    FOR p_id IN product_id_start..product_id_end LOOP
        SELECT unit_price INTO original_price
        FROM products
        WHERE product_id = p_id;

        IF FOUND THEN -- Check if SELECT INTO returned a row
            new_price := original_price * 0.95; -- 5% discount
            RAISE NOTICE 'Product %: Old Price = %, New Price = %', p_id, original_price, new_price;
            -- UPDATE products SET unit_price = new_price WHERE product_id = p_id; -- (If actually performing DML)
        ELSE
            RAISE NOTICE 'Product %: Not found or price is NULL (Postgres check: NOT FOUND)', p_id;
        END IF;
    END LOOP;
END $$;
```

**TL;DR:** Numeric `FOR` loops translate directly. Remember `FOUND` or `NOT FOUND` after `SELECT INTO` in PL/pgSQL to check if a row was returned.

---

#### Cursor `FOR` Loop (Implicit and Explicit)

This is one of the most critical looping constructs and highly used in Oracle. Thankfully, PL/pgSQL offers a very similar, robust, and often preferred way to process result sets.

| Oracle (PL/SQL)                             | Postgres (PL/pgSQL)                                                  | Notes / Key Differences                                                                                 |
| :------------------------------------------ | :------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| `FOR rec IN (SELECT...) LOOP ... END LOOP;` | `FOR rec IN SELECT... LOOP ... END LOOP;`                            | Direct translation; the record `rec` is implicitly declared as a `RECORD` variable. Highly recommended. |
| `FOR rec IN cur_name LOOP ... END LOOP;`    | (Less common directly for explicit cursor vars, implicit is favored) | Explicit cursors exist but are typically managed manually (see notes below).                            |

**Example: Listing Orders by Customer (Northwind `orders`, `customers` tables)**

```sql
-- Oracle PL/SQL (Implicit Cursor FOR Loop)
DECLARE
    v_customer_id customers.customer_id%TYPE := 'ANTON'; -- Antonio Moreno Taquería
BEGIN
    DBMS_OUTPUT.PUT_LINE('--- Orders for Customer ' || v_customer_id || ' ---');
    FOR order_rec IN (SELECT order_id, order_date, shipped_date, freight
                      FROM orders
                      WHERE customer_id = v_customer_id
                      ORDER BY order_date DESC)
    LOOP
        DBMS_OUTPUT.PUT_LINE('  Order ID: ' || order_rec.order_id || ', Date: ' || TO_CHAR(order_rec.order_date, 'YYYY-MM-DD') ||
                             ', Shipped: ' || NVL(TO_CHAR(order_rec.shipped_date, 'YYYY-MM-DD'), 'N/A') ||
                             ', Freight: ' || order_rec.freight);
    END LOOP;

    -- Using an explicit cursor in Oracle
    DBMS_OUTPUT.PUT_LINE(CHR(10) || '--- Orders for Customer (explicit cursor) ' || v_customer_id || ' ---');
    DECLARE
        CURSOR cur_customer_orders (p_customer_id customers.customer_id%TYPE) IS
            SELECT order_id, order_date, shipped_date, freight
            FROM orders
            WHERE customer_id = p_customer_id
            ORDER BY order_date DESC;
    BEGIN
        FOR order_rec IN cur_customer_orders(v_customer_id) LOOP
            DBMS_OUTPUT.PUT_LINE('  Order ID: ' || order_rec.order_id || ', Date: ' || TO_CHAR(order_rec.order_date, 'YYYY-MM-DD') ||
                                 ', Shipped: ' || NVL(TO_CHAR(order_rec.shipped_date, 'YYYY-MM-DD'), 'N/A') ||
                                 ', Freight: ' || order_rec.freight);
        END LOOP;
    END;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL (Cursor FOR Loop)
DO $$
DECLARE
    v_customer_id customers.customer_id%TYPE := 'ANTON'; -- Antonio Moreno Taquería
BEGIN
    RAISE NOTICE '--- Orders for Customer % ---', v_customer_id;
    FOR order_rec IN (SELECT order_id, order_date, shipped_date, freight
                      FROM orders
                      WHERE customer_id = v_customer_id
                      ORDER BY order_date DESC)
    LOOP
        RAISE NOTICE '  Order ID: %, Date: %, Shipped: %, Freight: %',
                     order_rec.order_id,
                     order_rec.order_date,
                     COALESCE(order_rec.shipped_date::TEXT, 'N/A'),
                     order_rec.freight;
    END LOOP;

    -- Postgres also supports explicit cursors, but FOR loop is generally preferred.
    -- For demonstration, here's how explicit cursor declaration looks:
    RAISE NOTICE E'\n--- Orders for Customer (explicit cursor) % ---', v_customer_id; -- E for escape sequences
    DECLARE
        my_explicit_cursor CURSOR FOR
            SELECT order_id, order_date, shipped_date, freight
            FROM orders
            WHERE customer_id = v_customer_id
            ORDER BY order_date DESC;
        order_rec orders%ROWTYPE; -- Or RECORD if less specific, but %ROWTYPE is explicit here
    BEGIN
        OPEN my_explicit_cursor;
        LOOP
            FETCH my_explicit_cursor INTO order_rec;
            EXIT WHEN NOT FOUND; -- Check if fetch was successful
            RAISE NOTICE '  Order ID: %, Date: %, Shipped: %, Freight: %',
                         order_rec.order_id,
                         order_rec.order_date,
                         COALESCE(order_rec.shipped_date::TEXT, 'N/A'),
                         order_rec.freight;
        END LOOP;
        CLOSE my_explicit_cursor;
    END;
END $$;
```

**TL;DR:** PostgreSQL's cursor `FOR` loop is a powerful and direct equivalent to Oracle's implicit cursor loop, and it is highly recommended for efficient set processing. It implicitly declares a `RECORD` variable (`order_rec` in the example) for each row. Explicit cursors are also available but generally not necessary for simple iteration due to the efficiency of `FOR IN SELECT` loops.

---

#### `FOREACH` Loop (PostgreSQL-specific for arrays)

PL/pgSQL's `FOREACH` loop is specifically designed for iterating over the elements of an array. While not a direct equivalent to `VARRAY` or `NESTED TABLE` *syntax*, it fulfills a similar role of iterating collection elements.

| Oracle (PL/SQL)                                                                                | Postgres (PL/pgSQL)                                    | Notes / Key Differences                            |
| :--------------------------------------------------------------------------------------------- | :----------------------------------------------------- | :------------------------------------------------- |
| `FOR i IN my_collection.FIRST..my_collection.LAST LOOP my_item := my_collection(i); END LOOP;` | `FOREACH element IN ARRAY my_array LOOP ... END LOOP;` | Simpler, more direct iteration for array elements. |

**Example: Iterating over a list of employee IDs (Northwind `employees` table)**

```sql
-- Oracle PL/SQL (conceptual, array population typically different)
DECLARE
    TYPE emp_id_arr IS VARRAY(9) OF employees.employee_id%TYPE;
    emp_ids emp_id_arr := emp_id_arr(1, 3, 5, 8); -- Example subset
    v_employee_first_name employees.first_name%TYPE;
    v_employee_last_name employees.last_name%TYPE;
BEGIN
    DBMS_OUTPUT.PUT_LINE('--- Processing specific Employee IDs (Oracle Concept) ---');
    FOR i IN emp_ids.FIRST..emp_ids.LAST LOOP
        SELECT first_name, last_name
        INTO v_employee_first_name, v_employee_last_name
        FROM employees
        WHERE employee_id = emp_ids(i);
        
        DBMS_OUTPUT.PUT_LINE('Employee: ' || v_employee_first_name || ' ' || v_employee_last_name || ' (ID: ' || emp_ids(i) || ')');
    END LOOP;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    emp_ids_list  SMALLINT[] := ARRAY[1, 3, 5, 8]; -- Nancy Davolio, Janet Leverling, Steven Buchanan, Laura Callahan
    current_emp_id employees.employee_id%TYPE;
    v_employee_first_name employees.first_name%TYPE;
    v_employee_last_name employees.last_name%TYPE;
BEGIN
    RAISE NOTICE '--- Processing specific Employee IDs (Postgres FOREACH) ---';
    FOREACH current_emp_id IN ARRAY emp_ids_list LOOP
        SELECT first_name, last_name
        INTO v_employee_first_name, v_employee_last_name
        FROM employees
        WHERE employee_id = current_emp_id;

        RAISE NOTICE 'Employee: % % (ID: %)', v_employee_first_name, v_employee_last_name, current_emp_id;
    END LOOP;
END $$;
```

**TL;DR:** When processing data already in a PL/pgSQL array, `FOREACH` is the idiomatic and clean way to iterate over its elements, conceptually mapping to `FOR i IN collection.FIRST..collection.LAST` loops in Oracle.

---

### `WHILE` Loops

Both PL/SQL and PL/pgSQL support `WHILE` loops with identical syntax, executing a block of code as long as a specified condition remains true.

| Oracle (PL/SQL)                      | Postgres (PL/pgSQL)                  | Notes / Key Differences |
| :----------------------------------- | :----------------------------------- | :---------------------- |
| `WHILE condition LOOP ... END LOOP;` | `WHILE condition LOOP ... END LOOP;` | Identical syntax.       |

**Example: Processing Remaining Stock (Northwind `products` table)**

Let's simulate selling units from a product's stock until a reorder level is met or stock runs out.

```sql
-- Oracle PL/SQL
DECLARE
    p_product_id  products.product_id%TYPE := 4; -- Chef Anton's Cajun Seasoning
    p_stock       products.units_in_stock%TYPE;
    p_reorder     products.reorder_level%TYPE;
    units_sold    NUMBER := 0;
BEGIN
    SELECT units_in_stock, reorder_level
    INTO p_stock, p_reorder
    FROM products
    WHERE product_id = p_product_id;

    DBMS_OUTPUT.PUT_LINE('--- Processing Product ' || p_product_id || ' Stock ---');
    DBMS_OUTPUT.PUT_LINE('Initial Stock: ' || p_stock || ', Reorder Level: ' || p_reorder);

    WHILE p_stock > p_reorder AND p_stock > 0 LOOP
        p_stock := p_stock - 1;
        units_sold := units_sold + 1;
        -- Simulate DML here: UPDATE products SET units_in_stock = p_stock WHERE product_id = p_product_id;
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Units Sold: ' || units_sold);
    DBMS_OUTPUT.PUT_LINE('Final Stock: ' || p_stock);
    IF p_stock <= p_reorder THEN
        DBMS_OUTPUT.PUT_LINE('Reorder triggered for product ' || p_product_id || '.');
    END IF;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    p_product_id  products.product_id%TYPE := 4; -- Chef Anton's Cajun Seasoning
    p_stock       products.units_in_stock%TYPE;
    p_reorder     products.reorder_level%TYPE;
    units_sold    INTEGER := 0; -- Changed from NUMBER to INTEGER for appropriate type
BEGIN
    SELECT units_in_stock, reorder_level
    INTO p_stock, p_reorder
    FROM products
    WHERE product_id = p_product_id;

    RAISE NOTICE '--- Processing Product % Stock ---', p_product_id;
    RAISE NOTICE 'Initial Stock: %, Reorder Level: %', p_stock, p_reorder;

    WHILE p_stock > p_reorder AND p_stock > 0 LOOP
        p_stock := p_stock - 1;
        units_sold := units_sold + 1;
        -- Simulate DML here: UPDATE products SET units_in_stock = p_stock WHERE product_id = p_product_id;
    END LOOP;

    RAISE NOTICE 'Units Sold: %', units_sold;
    RAISE NOTICE 'Final Stock: %', p_stock;
    IF p_stock <= p_reorder THEN
        RAISE NOTICE 'Reorder triggered for product %.', p_product_id;
    END IF;
END $$;
```

**TL;DR:** `WHILE` loops are directly transferable. Ensure loop termination conditions are correctly translated, especially concerning numeric or boolean comparisons based on PostgreSQL's specific data types (`REAL` vs `NUMERIC` for precision, for example).

---

### GOTO & Labels

Both languages support unconditional jumps using `GOTO` statements paired with labels. In both environments, its use is generally discouraged due to readability and maintainability concerns, favoring structured control flow whenever possible.

| Oracle (PL/SQL)                       | Postgres (PL/pgSQL)                   | Notes / Key Differences                            |
| :------------------------------------ | :------------------------------------ | :------------------------------------------------- |
| `<<label_name>> ... GOTO label_name;` | `<<label_name>> ... GOTO label_name;` | Syntax is essentially identical. Rarely necessary. |

**Example: Simulating an early exit (Northwind employee ID check)**

This example is illustrative and demonstrates syntax; such logic is usually better implemented with `IF`/`ELSIF` and `EXIT`.

```sql
-- Oracle PL/SQL
DECLARE
    v_employee_id employees.employee_id%TYPE := 6;
    v_is_manager BOOLEAN := FALSE;
BEGIN
    IF v_employee_id IS NULL THEN
        GOTO end_block;
    END IF;

    SELECT (reports_to IS NULL) INTO v_is_manager
    FROM employees
    WHERE employee_id = v_employee_id;

    IF v_is_manager THEN
        DBMS_OUTPUT.PUT_LINE('Employee ' || v_employee_id || ' is a top-level manager.');
        GOTO process_notes;
    END IF;

    DBMS_OUTPUT.PUT_LINE('Employee ' || v_employee_id || ' is not a top-level manager. Continue normal processing.');
    -- Normal flow would continue here
    GOTO end_block;

    <<process_notes>>
    DBMS_OUTPUT.PUT_LINE('Processing special notes for manager...');

    <<end_block>>
    DBMS_OUTPUT.PUT_LINE('Exiting GOTO demo.');
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    v_employee_id employees.employee_id%TYPE := 6;
    v_is_manager BOOLEAN := FALSE;
BEGIN
    IF v_employee_id IS NULL THEN
        GOTO end_block;
    END IF;

    SELECT (reports_to IS NULL) INTO v_is_manager
    FROM employees
    WHERE employee_id = v_employee_id;

    IF v_is_manager THEN
        RAISE NOTICE 'Employee % is a top-level manager.', v_employee_id;
        GOTO process_notes;
    END IF;

    RAISE NOTICE 'Employee % is not a top-level manager. Continue normal processing.', v_employee_id;
    -- Normal flow would continue here
    GOTO end_block;

    <<process_notes>>
    RAISE NOTICE 'Processing special notes for manager...';

    <<end_block>>
    RAISE NOTICE 'Exiting GOTO demo.';
END $$;
```

**The Use of `GOTO`!** While supported, excessive use of `GOTO` can lead to spaghetti code, just as in PL/SQL. Strive for structured alternatives like `IF`/`ELSIF`/`ELSE` and multiple `LOOP` types with `EXIT`.

---