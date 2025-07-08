## Variables & Data Types

Understanding variable declaration and data type handling is fundamental for transitioning from PL/SQL to PL/pgSQL. While many core concepts are similar, specific type names, behavior nuances, and idiomatic expressions differ.

### Declaring Variables

Variable declarations in PL/pgSQL share the `DECLARE` section with Oracle's PL/SQL, but PostgreSQL employs SQL-standard type names. It is best practice to match the exact data type of the underlying column if your variable directly corresponds to one.

| Oracle (PL/SQL)         | Postgres (PL/pgSQL)               | Notes / Key Differences                                                                                                                                       |
| :---------------------- | :-------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `v_id NUMBER;`          | `v_id INTEGER;`                   | PostgreSQL offers explicit integer sizes: `SMALLINT`, `INTEGER`, `BIGINT`.                                                                                    |
| `v_price NUMBER(10,2);` | `v_price NUMERIC(10,2);`          | `NUMERIC` and `DECIMAL` provide arbitrary precision. `REAL` is single-precision float; `DOUBLE PRECISION` is double. For financial data, prefer `NUMERIC`.    |
| `v_name VARCHAR2(100);` | `v_name VARCHAR(100);` or `TEXT;` | `VARCHAR(N)` behaves like `VARCHAR2(N)`. `TEXT` has no length limit and is generally idiomatic for strings.                                                   |
| `d_date DATE;`          | `d_date DATE;`                    | Oracle's `DATE` stores date and time; PostgreSQL `DATE` stores only date. `TIMESTAMP` (or `TIMESTAMP WITHOUT TIME ZONE`) is the equivalent for date and time. |
| `b_photo BLOB;`         | `b_photo BYTEA;`                  | `BYTEA` is PostgreSQL's native type for binary data.                                                                                                          |

**Example: Variable Declaration based on Northwind Schema**

```sql
-- Oracle PL/SQL
DECLARE
    -- From customers table
    l_customer_id         VARCHAR2(5)   := 'ALFKI';
    l_company_name        VARCHAR2(40);
    l_city                VARCHAR2(15);
    l_country             VARCHAR2(15);

    -- From orders table
    l_order_id            NUMBER        := 10248;
    l_order_date          DATE;
    l_freight             NUMBER; -- REAL in PG
    l_shipped_date        DATE;

    -- From order_details table
    l_product_id          NUMBER        := 11;
    l_quantity            NUMBER; -- SMALLINT in PG
    l_unit_price          NUMBER; -- REAL in PG

    -- From employees table
    l_employee_lastname   VARCHAR2(20);

    -- General-purpose numeric variable
    l_total_revenue       NUMBER(10,2);

BEGIN
    NULL; -- Placeholder
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    -- From customers table (customer_id CHARACTER VARYING(5), company_name CHARACTER VARYING(40), city CHARACTER VARYING(15), country CHARACTER VARYING(15))
    l_customer_id         VARCHAR(5)    := 'ALFKI';
    l_company_name        VARCHAR(40);
    l_city                VARCHAR(15);
    l_country             VARCHAR(15);

    -- From orders table (order_id SMALLINT, order_date DATE, freight REAL, shipped_date DATE)
    l_order_id            SMALLINT      := 10248;
    l_order_date          DATE;
    l_freight             REAL;
    l_shipped_date        DATE; -- If only date, otherwise TIMESTAMP

    -- From order_details table (product_id SMALLINT, quantity SMALLINT, unit_price REAL)
    l_product_id          SMALLINT      := 11;
    l_quantity            SMALLINT;
    l_unit_price          REAL;

    -- From employees table (last_name CHARACTER VARYING(20))
    l_employee_lastname   VARCHAR(20);

    -- General-purpose numeric variable
    l_total_revenue       NUMERIC(10,2); -- Prefer NUMERIC for financial precision

BEGIN
    -- Example: Fetching some data into declared variables from Northwind customers
    SELECT company_name, city, country
    INTO l_company_name, l_city, l_country
    FROM customers
    WHERE customer_id = l_customer_id;

    RAISE NOTICE 'Customer %: % from %, %', l_customer_id, l_company_name, l_city, l_country;
END $$;
```

**Type Mapping is Critical.** Always refer to PostgreSQL's actual type mapping (e.g., `psql \d <table>` or `pgAdmin`). Using `NUMERIC` for monetary values and precise integers like `SMALLINT`/`INTEGER` can prevent overflow or precision issues common in Oracle `NUMBER` usage if not specified carefully.

---

### Referencing Table Column Types (`%TYPE`)

Both Oracle PL/SQL and PostgreSQL PL/pgSQL provide the `%TYPE` attribute, which is highly recommended for creating variables whose data type matches a specific table column or another variable exactly. This practice enhances code maintainability and robustness by automatically adapting to schema changes.

| Oracle (PL/SQL)        | Postgres (PL/pgSQL)    | Notes / Key Differences        |
| :--------------------- | :--------------------- | :----------------------------- |
| `v_col tab.col%TYPE;`  | `v_col tab.col%TYPE;`  | Identical syntax and behavior. |
| `v_var var_name%TYPE;` | `v_var var_name%TYPE;` | Identical syntax and behavior. |

**Example: Using `%TYPE` in Northwind**

```sql
-- Oracle PL/SQL
DECLARE
    v_order_date  orders.order_date%TYPE;
    v_product_name products.product_name%TYPE;
    v_order_qty   order_details.quantity%TYPE;
    v_price       order_details.unit_price%TYPE;

    -- Declare based on another variable's type
    v_calculated_price v_price%TYPE;
BEGIN
    SELECT order_date
    INTO v_order_date
    FROM orders
    WHERE order_id = 10248;

    SELECT product_name, quantity, unit_price
    INTO v_product_name, v_order_qty, v_price
    FROM products p
    JOIN order_details od ON p.product_id = od.product_id
    WHERE od.order_id = 10248 AND od.product_id = 11;

    v_calculated_price := v_order_qty * v_price;

    DBMS_OUTPUT.PUT_LINE('Order 10248 on ' || TO_CHAR(v_order_date, 'YYYY-MM-DD'));
    DBMS_OUTPUT.PUT_LINE('  Product: ' || v_product_name || ', Qty: ' || v_order_qty || ', Unit Price: ' || v_price || ', Total: ' || v_calculated_price);
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    v_order_date  orders.order_date%TYPE;
    v_product_name products.product_name%TYPE;
    v_order_qty   order_details.quantity%TYPE;
    v_price       order_details.unit_price%TYPE;

    -- Declare based on another variable's type
    v_calculated_price v_price%TYPE;
BEGIN
    SELECT order_date
    INTO v_order_date
    FROM orders
    WHERE order_id = 10248;

    SELECT product_name, quantity, unit_price
    INTO v_product_name, v_order_qty, v_price
    FROM products p
    JOIN order_details od ON p.product_id = od.product_id
    WHERE od.order_id = 10248 AND od.product_id = 11;

    v_calculated_price := v_order_qty * v_price;

    RAISE NOTICE 'Order 10248 on %', v_order_date;
    RAISE NOTICE '  Product: %, Qty: %, Unit Price: %, Total: %', v_product_name, v_order_qty, v_price, v_calculated_price;
END $$;
```
**TL;DR:** The `%TYPE` attribute ensures data type compatibility and simplifies schema evolution in both database systems. Its direct portability is a convenience.

---

### Record Types (`RECORD` and `ROWTYPE`)

Both databases support fetching entire rows into a single variable, similar to row-level triggers. PostgreSQL offers both the untyped `RECORD` and the typed `%ROWTYPE`.

| Oracle (PL/SQL)             | Postgres (PL/pgSQL)         | Notes / Key Differences                                                                                                                                       |
| :-------------------------- | :-------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `v_row table_name%ROWTYPE;` | `v_row table_name%ROWTYPE;` | Direct equivalent for fully-typed row structures. Access columns: `v_row.column_name`.                                                                        |
| (Implicit cursor fetch)     | `v_row RECORD;`             | `RECORD` is an untyped generic record, inferred at runtime. More flexible but lacks compile-time type checking. Often used in `FOR` loops with query results. |

**Example: Fetching Row Data in Northwind**

```sql
-- Oracle PL/SQL
DECLARE
    emp_rec  employees%ROWTYPE;
BEGIN
    SELECT *
    INTO emp_rec
    FROM employees
    WHERE employee_id = 2; -- Andrew Fuller

    DBMS_OUTPUT.PUT_LINE('Employee ID: ' || emp_rec.employee_id);
    DBMS_OUTPUT.PUT_LINE('  Name: ' || emp_rec.first_name || ' ' || emp_rec.last_name);
    DBMS_OUTPUT.PUT_LINE('  Title: ' || emp_rec.title);
    DBMS_OUTPUT.PUT_LINE('  Hire Date: ' || TO_CHAR(emp_rec.hire_date, 'YYYY-MM-DD'));
END;
/
```

```sql
-- PostgreSQL PL/pgSQL (using %ROWTYPE)
DO $$
DECLARE
    emp_rec  employees%ROWTYPE;
BEGIN
    SELECT *
    INTO emp_rec
    FROM employees
    WHERE employee_id = 2; -- Andrew Fuller

    RAISE NOTICE 'Employee ID: %', emp_rec.employee_id;
    RAISE NOTICE '  Name: % %', emp_rec.first_name, emp_rec.last_name;
    RAISE NOTICE '  Title: %', emp_rec.title;
    RAISE NOTICE '  Hire Date: %', emp_rec.hire_date;
END $$;
```

**Example: Using `RECORD` for dynamic structures (PostgreSQL only)**

`RECORD` in PostgreSQL is particularly useful when the column structure of your query may not match an existing table row type, or when dealing with dynamic SQL.

```sql
-- PostgreSQL PL/pgSQL (using RECORD)
DO $$
DECLARE
    ord_prod_rec RECORD; -- Generic record type, columns defined by query result
BEGIN
    -- This query joins orders and products, creating a new structure
    SELECT
        o.order_id,
        c.company_name AS customer_name,
        p.product_name,
        od.quantity,
        od.unit_price
    INTO ord_prod_rec
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    JOIN order_details od ON o.order_id = od.order_id
    JOIN products p ON od.product_id = p.product_id
    WHERE o.order_id = 10250 AND p.product_id = 51; -- Example data: Order for Hanari Carnes, product 51

    RAISE NOTICE 'Order % for customer %: Product "%" (Qty: % @ $% / each)',
                 ord_prod_rec.order_id,
                 ord_prod_rec.customer_name,
                 ord_prod_rec.product_name,
                 ord_prod_rec.quantity,
                 ord_prod_rec.unit_price;
END $$;
```

**Watch Out For: `RECORD` Scope.** In PL/pgSQL, a `RECORD` variable's fields are only defined after it is used in a `SELECT INTO` or a `FOR` loop's `SELECT` statement. This means you cannot access its fields in the `DECLARE` section. Always initialize `RECORD` variables using an `INTO` clause.

---

### Arrays (`VARRAY`/`NESTED TABLE` vs. Native Arrays `[]`)

This is an area of significant divergence. Oracle offers sophisticated `VARRAY` (fixed-size) and `NESTED TABLE` (dynamic, sparse) collection types, along with associated `BULK COLLECT` and `FORALL` statements. PostgreSQL provides a simpler, more direct implementation of SQL-standard arrays.

| Oracle (PL/SQL)                                  | Postgres (PL/pgSQL)                                                                                | Notes / Key Differences                                                                                                      |
| :----------------------------------------------- | :------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------- |
| `TYPE my_arr IS VARRAY(10) OF VARCHAR2(50);`     | `my_array VARCHAR(50)[];`                                                                          | PostgreSQL arrays are directly typed (e.g., `INTEGER[]`, `TEXT[]`). Multi-dimensional arrays also supported (`INTEGER[][]`). |
| `TYPE my_tbl IS TABLE OF NUMBER;`                | `product_ids INT[];`                                                                               | Postgres arrays are more like Oracle's `VARRAY`s (dense) in basic usage, but resize dynamically.                             |
| Collection Methods (`COUNT`, `EXISTS`, `EXTEND`) | Array Functions/Operators (`array_length`, `unnest`, `@>`)                                         | Syntax and function names are different.                                                                                     |
| `FOR i IN my_arr.FIRST..my_arr.LAST`             | `FOR i IN array_lower(my_array,1)..array_upper(my_array,1)` or `FOREACH element IN ARRAY my_array` | Iteration is different. `FOREACH` is often simpler for array elements.                                                       |

**Example: Product IDs in an Array (Northwind context)**

```sql
-- Oracle PL/SQL (using VARRAY, conceptual for simplicity)
DECLARE
    TYPE product_id_array IS VARRAY(77) OF products.product_id%TYPE; -- Max 77 products in Northwind
    product_list product_id_array := product_id_array();
    
    -- Assuming a package or direct population:
    -- product_list.EXTEND(77);
    -- FOR i IN 1..77 LOOP
    --    SELECT product_id INTO product_list(i) FROM products WHERE ROWNUM = i ORDER BY product_id;
    -- END LOOP;
    
BEGIN
    -- For actual Northwind data, this might be populated by BULK COLLECT.
    -- For demonstration, manually add a few for context.
    product_list.EXTEND(3);
    product_list(1) := 1;   -- Chai
    product_list(2) := 2;   -- Chang
    product_list(3) := 77;  -- Original Frankfurter grüne Soße
    
    FOR i IN 1..product_list.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE('Oracle Array Item ' || i || ': Product ID = ' || product_list(i));
    END LOOP;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL (using native array)
DO $$
DECLARE
    product_ids_arr SMALLINT[]; -- Array of smallint (product_id type)
    p_id SMALLINT;
BEGIN
    -- Populate array directly (often using ARRAY_AGG from a query)
    product_ids_arr := ARRAY[1, 2, 77, 10]; -- Example: A few Northwind Product IDs

    RAISE NOTICE '--- Iterating product_ids_arr (basic FOR loop) ---';
    FOR i IN 1..array_length(product_ids_arr, 1) LOOP
        RAISE NOTICE 'Postgres Array Item %: Product ID = %', i, product_ids_arr[i];
    END LOOP;
    
    RAISE NOTICE '--- Iterating product_ids_arr (FOREACH loop) ---';
    FOREACH p_id IN ARRAY product_ids_arr LOOP
        RAISE NOTICE 'Product ID: %', p_id;
    END LOOP;
    
    -- Example of an array built from a query (fetching category IDs with over 15 products)
    DECLARE
        heavy_category_ids INTEGER[];
        cat_id             INTEGER;
    BEGIN
        SELECT ARRAY_AGG(category_id)
        INTO heavy_category_ids
        FROM products
        GROUP BY category_id
        HAVING COUNT(product_id) > 15;

        RAISE NOTICE '--- Categories with >15 products ---';
        FOREACH cat_id IN ARRAY heavy_category_ids LOOP
            SELECT category_name
            INTO v_product_name -- Reusing variable, this is bad practice, but demonstrates usage.
            FROM categories
            WHERE category_id = cat_id;
            RAISE NOTICE 'Category ID %: %', cat_id, v_product_name;
        END LOOP;
    END;
END $$;
```
**Watch Out For:** `NULL` elements in PostgreSQL arrays. While Oracle's collections can be sparse (meaning indices may exist but point to `NULL`), PostgreSQL arrays typically refer to dense arrays, where `NULL` is an actual element value. Use `array_remove(my_array, NULL)` if you need to clean out nulls.

---

### Casting Data Types (`TO_CHAR`/`TO_NUMBER` vs. `::` cast operator)

Oracle provides a suite of `TO_xxxx` functions (`TO_CHAR`, `TO_NUMBER`, `TO_DATE`). PostgreSQL uses a more generic `::` (double-colon) operator for casting, along with standard SQL `CAST()` syntax.

| Oracle (PL/SQL)             | Postgres (PL/pgSQL)                                                                | Notes / Key Differences                                                                                                          |
| :-------------------------- | :--------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------- |
| `TO_CHAR(num_var)`          | `num_var::TEXT` or `CAST(num_var AS TEXT)`                                         | Both are valid. `::` is more compact.                                                                                            |
| `TO_NUMBER(char_var)`       | `char_var::NUMERIC`                                                                | Specific numeric type often specified (`INTEGER`, `REAL`, `NUMERIC`).                                                            |
| `TO_DATE(char_var, format)` | `char_var::DATE` or `char_var::TIMESTAMP` (format inference or explicit `TO_DATE`) | PostgreSQL's parser can often infer simple date/time formats. Use `TO_DATE()`/`TO_TIMESTAMP()` for complex/non-standard formats. |
| `TO_CHAR(date_var, format)` | `TO_CHAR(date_var, format)`                                                        | `TO_CHAR` function exists in PostgreSQL for formatting dates/timestamps.                                                         |

**Example: Type Casting in Northwind Context**

```sql
-- Oracle PL/SQL
DECLARE
    v_order_id      orders.order_id%TYPE := 10249;
    v_order_date    orders.order_date%TYPE;
    v_string_id     VARCHAR2(10);
    v_num_string    VARCHAR2(10) := '42.40';
    v_parsed_number NUMBER;
BEGIN
    SELECT order_date INTO v_order_date FROM orders WHERE order_id = v_order_id;

    -- Casting NUMBER to VARCHAR2
    v_string_id := TO_CHAR(v_order_id);
    DBMS_OUTPUT.PUT_LINE('Order ID (as string): ' || v_string_id || ' (Length: ' || LENGTH(v_string_id) || ')');

    -- Casting DATE to VARCHAR2 with format
    DBMS_OUTPUT.PUT_LINE('Order Date formatted: ' || TO_CHAR(v_order_date, 'Mon DD, YYYY'));

    -- Casting VARCHAR2 to NUMBER
    v_parsed_number := TO_NUMBER(v_num_string);
    DBMS_OUTPUT.PUT_LINE('Parsed Number: ' || v_parsed_number || ' (Type check: ' || (CASE WHEN v_parsed_number = 42.4 THEN 'Success' ELSE 'Fail' END) || ')');
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    v_order_id      orders.order_id%TYPE := 10249;
    v_order_date    orders.order_date%TYPE;
    v_string_id     TEXT;
    v_num_string    TEXT := '42.40';
    v_parsed_number REAL; -- Matches order_details.unit_price type in Northwind
BEGIN
    SELECT order_date INTO v_order_date FROM orders WHERE order_id = v_order_id;

    -- Casting SMALLINT to TEXT
    v_string_id := v_order_id::TEXT;
    RAISE NOTICE 'Order ID (as string): % (Length: %)', v_string_id, LENGTH(v_string_id);
    
    -- Casting DATE to TEXT with format using TO_CHAR
    RAISE NOTICE 'Order Date formatted: %', TO_CHAR(v_order_date, 'Mon DD, YYYY');

    -- Casting TEXT to REAL
    v_parsed_number := v_num_string::REAL;
    RAISE NOTICE 'Parsed Number: % (Type check: %)', v_parsed_number, (CASE WHEN v_parsed_number = 42.4 THEN 'Success' ELSE 'Fail' END);

    -- Casting explicitly (alternative syntax)
    v_string_id := CAST(v_order_id AS TEXT);
    RAISE NOTICE 'Order ID (as string via CAST): %', v_string_id;

END $$;
```

**TL;DR: The `::` Operator is Ubiquitous.** Prefer the `::` operator for simplicity and conciseness when casting. For complex date/time formatting, `TO_CHAR` is still the function of choice, and `TO_DATE`/`TO_TIMESTAMP` for explicit string-to-date parsing with formats.

---

### NULL Handling (`IS NULL`/`IS NOT NULL`, `COALESCE`)

Standard SQL `IS NULL` and `IS NOT NULL` constructs function identically in both Oracle and PostgreSQL. The primary difference lies in the common functions for handling `NULL` values.

| Oracle (PL/SQL)             | Postgres (PL/pgSQL)                                     | Notes / Key Differences                                                         |
| :-------------------------- | :------------------------------------------------------ | :------------------------------------------------------------------------------ |
| `NVL(expr1, expr2)`         | `COALESCE(expr1, expr2, ...)`                           | `COALESCE` is standard SQL, supporting multiple expressions. Prefer `COALESCE`. |
| `NVL2(expr1, expr2, expr3)` | `CASE WHEN expr1 IS NOT NULL THEN expr2 ELSE expr3 END` | No direct `NVL2` equivalent. `CASE` statement provides full flexibility.        |

**Example: NULL Handling with Northwind Data (Customers region, Employees reports_to)**

Some `customers` in Northwind have `NULL` in their `region` column. `employees` have a `reports_to` column which can be `NULL` for the top-level manager.

```sql
-- Oracle PL/SQL
DECLARE
    v_customer_id       customers.customer_id%TYPE := 'ALFKI';
    v_company_name      customers.company_name%TYPE;
    v_region            customers.region%TYPE;
    v_display_region    VARCHAR2(50);
    
    v_emp_id            employees.employee_id%TYPE := 1;
    v_manager_id        employees.reports_to%TYPE;
    v_manager_info      VARCHAR2(100);

BEGIN
    -- Handle NULL region using NVL
    SELECT company_name, NVL(region, 'N/A')
    INTO v_company_name, v_display_region
    FROM customers
    WHERE customer_id = 'ALFKI'; -- Berlin, Region is NULL in Northwind data
    DBMS_OUTPUT.PUT_LINE('Customer ' || v_company_name || ' is in region: ' || v_display_region);

    SELECT company_name, NVL(region, 'Unknown Region')
    INTO v_company_name, v_display_region
    FROM customers
    WHERE customer_id = 'BOTTM'; -- Tsawassen, Region BC
    DBMS_OUTPUT.PUT_LINE('Customer ' || v_company_name || ' is in region: ' || v_display_region);

    -- NVL2 Example
    SELECT NVL2(reports_to, 'Reports to ' || reports_to, 'Top Manager')
    INTO v_manager_info
    FROM employees
    WHERE employee_id = 1; -- Nancy Davolio reports_to = 2
    DBMS_OUTPUT.PUT_LINE('Employee ID 1: ' || v_manager_info);

    SELECT NVL2(reports_to, 'Reports to ' || reports_to, 'Top Manager')
    INTO v_manager_info
    FROM employees
    WHERE employee_id = 2; -- Andrew Fuller reports_to = NULL
    DBMS_OUTPUT.PUT_LINE('Employee ID 2: ' || v_manager_info);

END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    v_customer_id       customers.customer_id%TYPE := 'ALFKI';
    v_company_name      customers.company_name%TYPE;
    v_region            customers.region%TYPE;
    v_display_region    TEXT; -- Use TEXT for flexible string handling
    
    v_emp_id            employees.employee_id%TYPE := 1;
    v_manager_id        employees.reports_to%TYPE;
    v_manager_info      TEXT; -- Use TEXT for flexible string handling

BEGIN
    -- Handle NULL region using COALESCE
    SELECT company_name, COALESCE(region, 'N/A')
    INTO v_company_name, v_display_region
    FROM customers
    WHERE customer_id = 'ALFKI'; -- Berlin, Region is NULL
    RAISE NOTICE 'Customer % is in region: %', v_company_name, v_display_region;

    SELECT company_name, COALESCE(region, 'Unknown Region')
    INTO v_company_name, v_display_region
    FROM customers
    WHERE customer_id = 'BOTTM'; -- Tsawassen, Region BC
    RAISE NOTICE 'Customer % is in region: %', v_company_name, v_display_region;

    -- Simulate NVL2 behavior using CASE
    SELECT CASE WHEN reports_to IS NOT NULL THEN 'Reports to ' || reports_to::TEXT ELSE 'Top Manager' END
    INTO v_manager_info
    FROM employees
    WHERE employee_id = 1; -- Nancy Davolio reports_to = 2
    RAISE NOTICE 'Employee ID 1: %', v_manager_info;

    SELECT CASE WHEN reports_to IS NOT NULL THEN 'Reports to ' || reports_to::TEXT ELSE 'Top Manager' END
    INTO v_manager_info
    FROM employees
    WHERE employee_id = 2; -- Andrew Fuller reports_to = NULL
    RAISE NOTICE 'Employee ID 2: %', v_manager_info;
END $$;
```

**TL;DR: `COALESCE` is the preferred `NULL` replacement function.** For complex `NVL2`-like logic, the standard `CASE` expression provides an equivalent and often more readable solution. Implicit conversion (like concatenating `reports_to` to `TEXT` with `::TEXT`) is usually helpful, or PostgreSQL attempts it automatically where sensible.

---