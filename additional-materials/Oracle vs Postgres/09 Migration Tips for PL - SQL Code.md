## Migration Tips for PL/SQL Code

Migrating PL/SQL code to PL/pgSQL is not always a direct line-for-line translation. While syntax can be similar, differences in language philosophy, transactional behavior, and optimizer hints mean certain Oracle development patterns become "antipatterns" in PostgreSQL. This section highlights common pitfalls and offers strategies for effective code conversion.

### Common Antipatterns from Oracle in Postgres

Certain coding habits, common and sometimes necessary in Oracle PL/SQL, can lead to suboptimal performance, unexpected behavior, or even outright errors when ported directly to PostgreSQL PL/pgSQL.

#### 1. Excessive Row-by-Row Processing (Explicit Cursors and Iterative DML)

*   **Oracle Antipattern:** Over-reliance on explicit `OPEN...FETCH...CLOSE` loops or writing DML statements inside simple `FOR...LOOP`s without utilizing `BULK COLLECT` for fetching or `FORALL` for DML. While Oracle's `BULK COLLECT` and `FORALL` exist to mitigate this, developers sometimes neglect them for smaller datasets, or `FORALL` is conceptually harder for updates/deletes without array collections.

    ```sql
    -- Oracle PL/SQL Antipattern: Row-by-Row Stock Adjustment
    DECLARE
        v_category_id products.category_id%TYPE := 1; -- Beverages
        v_adj_qty     NUMBER := -5;
    BEGIN
        DBMS_OUTPUT.PUT_LINE('Oracle: Adjusting stock row-by-row...');
        FOR prod_rec IN (SELECT product_id FROM products WHERE category_id = v_category_id) LOOP
            UPDATE products
            SET units_in_stock = units_in_stock + v_adj_qty
            WHERE product_id = prod_rec.product_id;
        END LOOP;
        DBMS_OUTPUT.PUT_LINE('  Stock adjusted, one by one. (Inefficient for many rows)');
        ROLLBACK;
    END;
    /
    ```

*   **PostgreSQL Problem:** PL/pgSQL context-switching overhead is similar or potentially higher for frequent transitions between the procedural and SQL engines. Direct conversion of row-by-row logic is a major performance killer, especially for larger datasets. The core strength of relational databases is set-based operations.

*   **Migration Tip (The Golden Rule):** Always strive to convert procedural, row-by-row logic into a single, efficient, set-based SQL statement. For inserts from data, use `INSERT INTO ... SELECT ...`. For updates or deletes of multiple rows, use `UPDATE ... WHERE EXISTS` or `UPDATE ... FROM other_table`. 

    ```sql
    -- PostgreSQL Best Practice: Set-based Stock Adjustment
    DO $$
    DECLARE
        v_category_id products.category_id%TYPE := 1; -- Beverages
        v_adj_qty     SMALLINT := -5;
    BEGIN
        RAISE NOTICE 'Postgres: Adjusting stock with a single set-based UPDATE...';
        UPDATE products
        SET units_in_stock = units_in_stock + v_adj_qty
        WHERE category_id = v_category_id;
        RAISE NOTICE '  Stock adjusted in one statement.';
        ROLLBACK; -- Undo for demo
    END $$;
    ```
    This eliminates numerous context switches and allows the PostgreSQL query planner to fully optimize the DML operation.

#### 2. Overuse of `WHERE 1=1` for Optional Filters

*   **Oracle Antipattern:** In dynamic SQL, it's common to see `WHERE 1=1` followed by `AND` clauses to build optional filters without worrying about the initial `WHERE` keyword.

    ```sql
    -- Oracle PL/SQL Antipattern: Dynamic Query with WHERE 1=1
    DECLARE
        p_customer_city customers.city%TYPE := 'London';
        p_employee_id   employees.employee_id%TYPE := 7; -- Robert King
        
        v_sql_stmt VARCHAR2(1000);
        r_order_rec orders%ROWTYPE;
    BEGIN
        v_sql_stmt := 'SELECT * FROM orders WHERE 1=1';
        IF p_customer_city IS NOT NULL THEN
            v_sql_stmt := v_sql_stmt || ' AND ship_city = :city_param';
        END IF;
        IF p_employee_id IS NOT NULL THEN
            v_sql_stmt := v_sql_stmt || ' AND employee_id = :emp_id_param';
        END IF;

        DBMS_OUTPUT.PUT_LINE('Oracle: Dynamic query generated with 1=1: ' || v_sql_stmt);
        -- ... then EXECUTE IMMEDIATE using dynamic cursor for iteration (complex in basic demo)
    END;
    /
    ```

*   **PostgreSQL Problem:** While `WHERE 1=1` technically works, it's unnecessary and less explicit. PostgreSQL dynamic SQL (using positional parameters `$1, $2, ...` and `USING` clause) or directly constructing the `WHERE` clause without a dummy predicate are cleaner.

*   **Migration Tip:** Build dynamic `WHERE` clauses by checking for the *first* condition and prepending `WHERE` accordingly. For subsequent conditions, prepend `AND`. This results in more precise SQL.

    ```sql
    -- PostgreSQL Best Practice: Dynamic Query Construction
    DO $$
    DECLARE
        p_customer_city TEXT := 'London';
        p_employee_id   SMALLINT := 7; -- Robert King
        
        v_sql_stmt      TEXT;
        v_where_clauses TEXT[] := '{}'; -- An empty array for dynamic WHERE conditions
        v_param_values  JSONB := '{}'; -- Store parameter values as JSON for flexible binding
        v_param_counter INTEGER := 0;

        r_order_rec     RECORD;
    BEGIN
        -- Build the WHERE clause dynamically
        IF p_customer_city IS NOT NULL THEN
            v_param_counter := v_param_counter + 1;
            v_where_clauses := array_append(v_where_clauses, 'ship_city = $' || v_param_counter);
            v_param_values := jsonb_insert(v_param_values, '{' || v_param_counter || '}', to_jsonb(p_customer_city), true);
        END IF;
        IF p_employee_id IS NOT NULL THEN
            v_param_counter := v_param_counter + 1;
            v_where_clauses := array_append(v_where_clauses, 'employee_id = $' || v_param_counter);
            v_param_values := jsonb_insert(v_param_values, '{' || v_param_counter || '}', to_jsonb(p_employee_id), true);
        END IF;

        v_sql_stmt := 'SELECT order_id, order_date FROM orders';
        IF array_length(v_where_clauses, 1) IS NOT NULL THEN
            v_sql_stmt := v_sql_stmt || ' WHERE ' || array_to_string(v_where_clauses, ' AND ');
        END IF;
        v_sql_stmt := v_sql_stmt || ' ORDER BY order_date DESC LIMIT 3;';

        RAISE NOTICE 'Postgres: Dynamic query: %', v_sql_stmt;
        
        -- Execute the dynamic query using EXECUTE and dynamic parameters (advanced usage here)
        -- The actual parameter binding for arbitrary number of params is tricky with FOR..IN EXECUTE
        -- For a truly dynamic number of parameters you would construct it as `EXECUTE query USING val1, val2...`
        -- In simple cases you'd do: IF num_params=1 THEN EXECUTE query USING p1; ELSE EXECUTE query USING p1, p2; END IF;
        -- Or use EXECUTE FORMAT which can inject parameters directly, but is less safe against injection
        
        -- A simpler way for a demo with optional params would be to concatenate values for demo simplicity:
        -- NOT SAFE FOR PROD (SQL INJECTION RISK if inputs not trusted/sanitized)
        v_sql_stmt := 'SELECT order_id, order_date FROM orders';
        DECLARE
             first_condition_added BOOLEAN := FALSE;
        BEGIN
             IF p_customer_city IS NOT NULL THEN
                 v_sql_stmt := v_sql_stmt || ' WHERE ship_city = ''' || p_customer_city || '''';
                 first_condition_added := TRUE;
             END IF;
             IF p_employee_id IS NOT NULL THEN
                 IF first_condition_added THEN v_sql_stmt := v_sql_stmt || ' AND '; ELSE v_sql_stmt := v_sql_stmt || ' WHERE '; END IF;
                 v_sql_stmt := v_sql_stmt || 'employee_id = ' || p_employee_id;
             END IF;
             v_sql_stmt := v_sql_stmt || ' ORDER BY order_date DESC LIMIT 3;';

             RAISE NOTICE 'Postgres: (Simple Safe Dynamic Demo): %', v_sql_stmt;
             FOR r_order_rec IN EXECUTE v_sql_stmt LOOP
                RAISE NOTICE '  Order: %, Date: %', r_order_rec.order_id, r_order_rec.order_date;
             END LOOP;
        END;

    END $$;
    ```
    The advanced technique with `USING` and arbitrary parameters usually involves writing more complex helper code for `EXECUTE`, or accepting explicit arguments when creating the function/procedure. The core idea is to *not* use `1=1`.

#### 3. Over-Reliance on Implicit Type Conversions / Lack of Explicit Type Specifiers

*   **Oracle Antipattern:** Oracle often performs aggressive implicit conversions between `NUMBER`, `VARCHAR2`, and `DATE`, potentially hiding type mismatches or leading to unexpected results if format masks are ambiguous or omitted (e.g., `'10-JAN-2023'` can be parsed multiple ways depending on `NLS_DATE_FORMAT`). `NUMBER` as a general-purpose numeric type means precision can sometimes be unclear.

    ```sql
    -- Oracle PL/SQL Antipattern: Implicit Conversion Risk
    DECLARE
        v_numeric_str VARCHAR2(10) := '12.34';
        v_date_str    VARCHAR2(20) := '01/FEB/23';
        
        v_num_result  NUMBER;
        v_date_result DATE;
    BEGIN
        v_num_result := v_numeric_str; -- Implicit, might depend on NLS_NUMERIC_CHARACTERS
        DBMS_OUTPUT.PUT_LINE('Oracle: Implicit numeric: ' || v_num_result);

        v_date_result := v_date_str;  -- Implicit, highly depends on NLS_DATE_FORMAT
        DBMS_OUTPUT.PUT_LINE('Oracle: Implicit date: ' || v_date_result);
    END;
    /
    ```

*   **PostgreSQL Problem:** PostgreSQL's type system is stricter and adheres more closely to SQL standards. Implicit conversions are less common, and when they occur, might not behave exactly as in Oracle. Omitting explicit type specifiers or relying on locale-dependent date/number formats is a recipe for runtime errors or data corruption. Numerical types are more explicit (e.g. `REAL`, `NUMERIC`, `SMALLINT`).

*   **Migration Tip:** Be explicit with data types and use casting operators (`::` or `CAST()`) or formatting functions (`TO_CHAR()`, `TO_DATE()`, `TO_TIMESTAMP()`) to control conversions. Match PostgreSQL's precise numeric types to ensure correctness (e.g., `NUMERIC` for financial, `SMALLINT` for small integers).

    ```sql
    -- PostgreSQL Best Practice: Explicit Type Casting
    DO $$
    DECLARE
        v_numeric_str TEXT := '12.34';
        v_date_str    TEXT := '01-FEB-2023'; -- Standard format expected
        
        v_num_result  NUMERIC;
        v_date_result DATE;
    BEGIN
        v_num_result := v_numeric_str::NUMERIC; -- Explicit cast to numeric
        RAISE NOTICE 'Postgres: Explicit numeric: %', v_num_result;

        -- For string-to-date conversion, use TO_DATE with a format string or rely on ISO 8601 YYYY-MM-DD
        v_date_result := TO_DATE(v_date_str, 'DD-MON-YYYY');
        RAISE NOTICE 'Postgres: Explicit date: %', v_date_result;

        -- Even simple concatenations require explicit casting if mixing numeric and text
        RAISE NOTICE 'Product ID ' || 1::TEXT || ' stock: ' || 39::TEXT; -- Concatenating number types to text requires cast
    END $$;
    ```

#### 4. Relying on `DBMS_OUTPUT` for Production Logging / Error Reporting

*   **Oracle Antipattern:** `DBMS_OUTPUT.PUT_LINE` is often used for debugging, progress reporting, and sometimes even simple application logging in Oracle. For error reporting, relying solely on printing to `DBMS_OUTPUT` (without robust logging or exception handling) is common.

*   **PostgreSQL Problem:** `RAISE NOTICE` is the direct equivalent for debugging/console output, but for actual logging or error reporting in a production system, `RAISE NOTICE` should not be seen as a replacement for a structured logging system. Database messages (INFO, WARNING, ERROR) are configured by `log_min_messages` and may not reach the user unless `client_min_messages` is appropriately set. Errors are better handled with structured exceptions.

*   **Migration Tip:**
    *   For debugging messages, replace `DBMS_OUTPUT.PUT_LINE` with `RAISE NOTICE 'My message %', my_var;`.
    *   For structured logging, consider logging to PostgreSQL's native server logs (via `RAISE INFO 'my message';`), or integrate with an external logging solution, or store log data in dedicated audit/log tables.
    *   For controlled error handling and reporting to client applications, use `RAISE EXCEPTION 'My error: %', SQLERRM USING ERRCODE='my_custom_code';` and capture these errors in the application layer.

### Rethinking Global State (No Package Globals)

One of the most fundamental architectural shifts required when moving from Oracle PL/SQL is dealing with package-level global variables. Oracle packages can declare variables (e.g., counters, flags, configuration values, cached lookup data) that maintain their state for the entire user session, across multiple procedure/function calls. PostgreSQL PL/pgSQL functions and procedures do *not* have direct access to such persistent session-level global state mechanisms.

*   **Oracle Concept: Package Globals for Configuration & Caching**
    ```sql
    -- Oracle PL/SQL Package with Global State
    CREATE PACKAGE app_config AS
        g_debug_mode BOOLEAN := FALSE; -- Default is off
        g_tax_rate   CONSTANT NUMBER := 0.08;
        PROCEDURE set_debug_mode(p_mode BOOLEAN);
        FUNCTION get_tax_rate RETURN NUMBER;
    END app_config;
    /
    CREATE PACKAGE BODY app_config AS
        PROCEDURE set_debug_mode(p_mode BOOLEAN) IS
        BEGIN g_debug_mode := p_mode; END;
        FUNCTION get_tax_rate RETURN NUMBER IS
        BEGIN RETURN g_tax_rate; END;
    END app_config;
    /
    -- Later usage:
    -- BEGIN app_config.set_debug_mode(TRUE); END;
    -- IF app_config.g_debug_mode THEN DBMS_OUTPUT.PUT_LINE('Debugging!'); END IF;
    ```

*   **PostgreSQL Problem:** Direct translation is not possible. Each PL/pgSQL function/procedure call generally runs in its own scope and does not inherently share modifiable state with other function/procedure calls within the same session without explicit mechanisms. Attempting to manage `GLOBAL` state inside a PostgreSQL procedural code might hint that code logic needs to be revisited.

*   **Migration Tip Strategies:**

    1.  **Configuration Settings:**
        *   **`current_setting()` and `set_config()`:** For system-wide or session-level configurations, use PostgreSQL's `set_config()` function to set custom GUC (Grand Unified Configuration) variables within a session. These can be retrieved with `current_setting()`. They persist for the session duration or until explicitly reset. They are global within the current session, but readable *across all languages* and `SET`table from outside of PL/pgSQL.

        ```sql
        -- PostgreSQL Alternative for Global Configuration (Using Custom GUC)
        -- First, optionally define custom GUCs for persistence/documentation (not mandatory to declare in postgresql.conf first)
        -- postgresql.conf entry (requires restart): custom_variable_types.myapp.debug_mode = bool
        -- Or just use arbitrary names dynamically:
        
        -- Setter procedure
        CREATE OR REPLACE PROCEDURE set_debug_mode_pg(p_mode BOOLEAN)
        LANGUAGE plpgsql AS $$
        BEGIN
            PERFORM set_config('myapp.debug_mode', p_mode::TEXT, false); -- false = session-level
            RAISE NOTICE 'Debug mode set to: %', p_mode;
        END;
        $$;
        
        -- Getter function (immutable if parameter is always same string)
        CREATE OR REPLACE FUNCTION get_debug_mode_pg()
        RETURNS BOOLEAN
        LANGUAGE plpgsql STABLE -- Stable because current_setting does not change within a query plan execution
        AS $$
        BEGIN
            -- 'true' as third param returns NULL if setting is not set, instead of erroring
            RETURN current_setting('myapp.debug_mode', true)::BOOLEAN; 
        END;
        $$;

        -- Test in Postgres
        -- CALL set_debug_mode_pg(TRUE);
        -- SELECT get_debug_mode_pg();
        -- DO $$ BEGIN IF get_debug_mode_pg() THEN RAISE NOTICE 'Debugging is ON!'; END IF; END $$;
        -- SET myapp.debug_mode TO 'off'; -- Can also be set directly in client session
        -- SELECT get_debug_mode_pg();
        ```

        *   **Dedicated Configuration Table:** For more complex configurations or parameters that need to persist across sessions and be manageable via SQL. Access via `SELECT ...` queries.
            ```sql
            CREATE TABLE app_settings (
                setting_name  TEXT PRIMARY KEY,
                setting_value TEXT
            );
            INSERT INTO app_settings VALUES ('tax_rate', '0.08'), ('promo_active', 'TRUE');

            -- Function to get a setting value
            CREATE OR REPLACE FUNCTION get_setting(p_name TEXT)
            RETURNS TEXT
            LANGUAGE plpgsql STABLE -- It reads from a table, so it is stable
            AS $$
            DECLARE
                v_value TEXT;
            BEGIN
                SELECT setting_value INTO v_value FROM app_settings WHERE setting_name = p_name;
                RETURN v_value;
            END;
            $$;

            -- Usage:
            -- SELECT get_setting('tax_rate')::NUMERIC;
            ```

    2.  **Session-Specific Caching / Large Data Sharing:**
        *   **`TEMPORARY TABLES`:** For caching lookup data or intermediate results within a single session that would otherwise be package global arrays/collections. These tables are dropped automatically at session end (`ON COMMIT DROP` option available for `TEMPORARY` tables). They are perfect for staging data during complex batch operations in a session.
        *   **Passing as Parameters:** Explicitly pass required data/context between function calls as parameters. While sometimes verbose, it ensures clarity and functional purity.

### Replacing DDL in Procedures

Oracle PL/SQL routines (procedures, functions, triggers, anonymous blocks) can execute DDL (e.g., `CREATE TABLE`, `ALTER INDEX`, `TRUNCATE TABLE`, `CREATE OR REPLACE VIEW`). This DDL operation implicitly commits any pending transaction *before* and *after* the DDL statement, which is a common source of unexpected commits.

*   **Oracle Antipattern:** Using DDL within procedural blocks that are part of a larger transaction, potentially leading to partial commits.

    ```sql
    -- Oracle PL/SQL Antipattern: DDL within a procedure/function (implicitly commits)
    CREATE OR REPLACE PROCEDURE create_and_insert_log_oracle AS
    BEGIN
        -- Existing DML will be implicitly committed here by DDL.
        -- INSERT INTO some_table VALUES (1); 
        
        EXECUTE IMMEDIATE 'CREATE TABLE temp_daily_log_data (log_message VARCHAR2(200), log_timestamp DATE)';
        
        INSERT INTO temp_daily_log_data VALUES ('Log entry at ' || SYSDATE, SYSDATE);
        -- Any outstanding DML (e.g., a prior INSERT before CREATE TABLE)
        -- would have been committed by the CREATE TABLE.
        -- This INSERT here is part of a new transaction which will also implicitly commit.

        DBMS_OUTPUT.PUT_LINE('Oracle: temp_daily_log_data table created and row inserted.');
        
    EXCEPTION
        WHEN OTHERS THEN
            DBMS_OUTPUT.PUT_LINE('Oracle: Error ' || SQLERRM || ', potential partial commit state.');
            RAISE; -- Cannot rollback what's implicitly committed
    END;
    /
    -- Test with: BEGIN create_and_insert_log_oracle; END;
    -- Try calling after a main transaction has started but not committed.
    ```

*   **PostgreSQL Problem:** As discussed in Chapter IV (Functions & Procedures) and Chapter VI (Error Handling), PostgreSQL `CREATE FUNCTION`s cannot contain DDL statements at all. They must participate strictly within the calling transaction's snapshot. DDL *is* allowed within `CREATE PROCEDURE`s (PG 11+) and `DO` blocks.

*   **Migration Tip Strategies:**

    1.  **If the DDL is intended to alter persistent schema objects (e.g., `CREATE TABLE`, `ALTER TABLE` for permanent tables):**
        *   **Move to Deployment Scripts:** This is the most common and robust approach. Schema changes belong in migration scripts managed by tools (e.g., Liquibase, Flyway, or simple versioned SQL files), applied during deployment or schema upgrades, *outside* the application's runtime.
        *   **Encapsulate in a `CREATE PROCEDURE` (PG 11+):** If schema changes *must* be initiated from the database runtime (e.g., dynamic table creation for sharding, rarely recommended), wrap them in a `CREATE PROCEDURE`. Remember, `PROCEDURE`s are distinct, called via `CALL`, and are designed to handle their own transactional behavior (including implicit or explicit commits around DDL).
            ```sql
            -- PostgreSQL Best Practice: DDL in a PROCEDURE (PG 11+)
            CREATE OR REPLACE PROCEDURE create_and_insert_log_pg (p_table_name TEXT)
            LANGUAGE plpgsql AS $$
            DECLARE
                v_full_table_name TEXT := 'temp_logs.' || p_table_name;
            BEGIN
                -- This DDL is allowed here
                EXECUTE 'CREATE TABLE IF NOT EXISTS ' || v_full_table_name || ' (log_message TEXT, log_timestamp TIMESTAMP)';
                
                EXECUTE 'INSERT INTO ' || v_full_table_name || ' VALUES ($1, NOW())' USING 'Log entry for ' || p_table_name || ' at ' || NOW();

                RAISE NOTICE 'Postgres: % created and row inserted.', v_full_table_name;
                COMMIT; -- Explicit commit if you want this to be separate tx from caller.
                        -- Or let it be part of the outer transaction if the PROCEDURE is designed to participate.
            EXCEPTION
                WHEN OTHERS THEN
                    RAISE NOTICE 'Postgres: Error creating/inserting log: SQLSTATE % / SQLERRM %', SQLSTATE, SQLERRM;
                    ROLLBACK; -- Rollback this specific PROCEDURE transaction
                    RAISE; -- Reraise error to caller
            END;
            $$;

            -- Usage:
            -- CREATE SCHEMA IF NOT EXISTS temp_logs;
            -- CALL create_and_insert_log_pg('daily_report_log');
            -- SELECT * FROM temp_logs.daily_report_log;
            -- DROP TABLE temp_logs.daily_report_log;
            -- DROP SCHEMA temp_logs CASCADE;
            ```

    2.  **If the DDL is for temporary, session-specific objects (e.g., `CREATE GLOBAL TEMPORARY TABLE`):**
        *   **Use `CREATE TEMPORARY TABLE` with `ON COMMIT DROP`:** PostgreSQL's temporary tables are powerful, session-private, and can be configured to automatically drop at the end of a transaction or session. This makes them ideal for in-memory, short-lived staging or lookup tables. This DDL can occur directly within a PL/pgSQL function or procedure.
            ```sql
            -- PostgreSQL Best Practice: Temporary Table DDL in any PL/pgSQL routine
            CREATE OR REPLACE FUNCTION process_product_metrics(p_category_id products.category_id%TYPE)
            RETURNS TEXT
            LANGUAGE plpgsql AS $$
            DECLARE
                v_count INTEGER;
            BEGIN
                -- Create a temporary table that is automatically dropped at end of transaction
                CREATE TEMPORARY TABLE current_category_products (
                    product_id    products.product_id%TYPE,
                    product_name  products.product_name%TYPE
                ) ON COMMIT DROP; -- This DDL is allowed in function
                
                INSERT INTO current_category_products (product_id, product_name)
                SELECT product_id, product_name FROM products WHERE category_id = p_category_id;
                
                SELECT COUNT(*) INTO v_count FROM current_category_products;
                RAISE NOTICE 'Processed % products for category %.', v_count, p_category_id;

                -- Perform further operations on current_category_products
                -- SELECT ... FROM current_category_products;

                RETURN 'Category metrics processed.';
            END;
            $$;

            -- Usage:
            -- SELECT process_product_metrics(1);
            -- SELECT * FROM current_category_products; -- Will likely error or show nothing as table already dropped
            ```

**TL;DR:** The "DDL commits transaction" rule from Oracle means existing Oracle procedures often hide transaction boundaries. PostgreSQL strictly separates DDL capabilities (`PROCEDURE`/`DO` blocks) from `FUNCTION`s. Analyze the purpose of the DDL: for permanent schema changes, shift to migration scripts; for transactional isolation, convert to `PROCEDURE`s; for temporary work tables, utilize `TEMPORARY TABLE` within any procedural block.