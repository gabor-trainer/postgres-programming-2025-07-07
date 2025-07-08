## VI. Error Handling

### `EXCEPTION WHEN...THEN` Block Equivalent

Both procedural languages employ a structured `EXCEPTION` block for handling runtime errors. The syntax for catching specific error conditions or a generic `OTHERS` handler is quite similar, making direct migration of logic relatively straightforward.

| Oracle (PL/SQL)                                                          | Postgres (PL/pgSQL)                                                | Notes / Key Differences                                                                       |
| :----------------------------------------------------------------------- | :----------------------------------------------------------------- | :-------------------------------------------------------------------------------------------- |
| `EXCEPTION WHEN error_name THEN ... WHEN OTHERS THEN ... END;`           | `EXCEPTION WHEN condition_name THEN ... WHEN OTHERS THEN ... END;` | Keywords are identical. `condition_name` refers to SQLSTATE mnemonics or codes.               |
| Predefined Oracle exceptions (e.g., `NO_DATA_FOUND`, `DUP_VAL_ON_INDEX`) | SQLSTATE conditions (e.g., `NO_DATA_FOUND`, `UNIQUE_VIOLATION`)    | PostgreSQL uses standard SQLSTATE mnemonics or 5-character codes for specific error handling. |

**Example: Handling Missing Customer Data & Duplicate Category ID (Northwind)**

Let's attempt to fetch details for a non-existent customer and try to insert a duplicate category ID, handling both cases.

```sql
-- Oracle PL/SQL Error Handling
DECLARE
    v_customer_id       customers.customer_id%TYPE := 'NON_EXISTENT';
    v_company_name      customers.company_name%TYPE;
    v_error_message     VARCHAR2(200);

    v_dup_category_id   categories.category_id%TYPE := 1; -- Beverages
    v_dup_category_name categories.category_name%TYPE := 'Duplicate Name';
BEGIN
    DBMS_OUTPUT.PUT_LINE('Oracle: Starting Error Handling Demo...');

    -- Case 1: NO_DATA_FOUND
    BEGIN
        SELECT company_name INTO v_company_name
        FROM customers
        WHERE customer_id = v_customer_id;

        DBMS_OUTPUT.PUT_LINE('  Fetched Company: ' || v_company_name); -- This line should not be reached
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            DBMS_OUTPUT.PUT_LINE('  Handled: Customer ID ' || v_customer_id || ' not found.');
        WHEN OTHERS THEN
            v_error_message := SQLERRM;
            DBMS_OUTPUT.PUT_LINE('  Handled unexpected error in NO_DATA_FOUND block: ' || v_error_message);
    END;

    -- Case 2: DUP_VAL_ON_INDEX
    BEGIN
        INSERT INTO categories (category_id, category_name)
        VALUES (v_dup_category_id, v_dup_category_name);

        DBMS_OUTPUT.PUT_LINE('  Inserted duplicate category: ' || v_dup_category_name); -- This should not be reached
    EXCEPTION
        WHEN DUP_VAL_ON_INDEX THEN
            DBMS_OUTPUT.PUT_LINE('  Handled: Attempted to insert duplicate category ID ' || v_dup_category_id);
            ROLLBACK; -- Rollback this specific failed transaction (if in autonomous block)
        WHEN OTHERS THEN
            v_error_message := SQLERRM;
            DBMS_OUTPUT.PUT_LINE('  Handled unexpected error in DUP_VAL_ON_INDEX block: ' || v_error_message);
            ROLLBACK;
    END;
    
    DBMS_OUTPUT.PUT_LINE('Oracle: Demo Finished.');
    COMMIT; -- If not in an autonomous transaction
END;
/
```

```sql
-- PostgreSQL PL/pgSQL Error Handling
DO $$
DECLARE
    v_customer_id       customers.customer_id%TYPE := 'NON_EXISTENT';
    v_company_name      customers.company_name%TYPE;

    v_dup_category_id   categories.category_id%TYPE := 1; -- Beverages
    v_dup_category_name categories.category_name%TYPE := 'Duplicate Name PG';
BEGIN
    RAISE NOTICE 'Postgres: Starting Error Handling Demo...';

    -- Case 1: NO_DATA_FOUND (sets variables to NULL if not found, unless STRICT is used)
    BEGIN
        SELECT company_name INTO STRICT v_company_name -- Use STRICT to force a NO_DATA_FOUND error
        FROM customers
        WHERE customer_id = v_customer_id;

        RAISE NOTICE '  Fetched Company: %', v_company_name; -- This line should not be reached
    EXCEPTION
        WHEN NO_DATA_FOUND THEN -- Mnemonic for SQLSTATE '20000'
            RAISE NOTICE '  Handled: Customer ID % not found.', v_customer_id;
        WHEN OTHERS THEN
            RAISE NOTICE '  Handled unexpected error in NO_DATA_FOUND block: SQLSTATE % / SQLERRM %', SQLSTATE, SQLERRM;
    END;

    -- Case 2: UNIQUE_VIOLATION (Postgres equivalent of DUP_VAL_ON_INDEX)
    BEGIN
        INSERT INTO categories (category_id, category_name)
        VALUES (v_dup_category_id, v_dup_category_name);

        RAISE NOTICE '  Inserted duplicate category: %', v_dup_category_name; -- This should not be reached
    EXCEPTION
        WHEN UNIQUE_VIOLATION THEN -- Mnemonic for SQLSTATE '23505'
            RAISE NOTICE '  Handled: Attempted to insert duplicate category ID %.', v_dup_category_id;
            ROLLBACK; -- If this DO block implicitly started a transaction
        WHEN OTHERS THEN
            RAISE NOTICE '  Handled unexpected error in UNIQUE_VIOLATION block: SQLSTATE % / SQLERRM %', SQLSTATE, SQLERRM;
            ROLLBACK;
    END;
    
    RAISE NOTICE 'Postgres: Demo Finished.';
    ROLLBACK; -- Ensure database state is reset for demo re-runs (especially important in DO blocks)
END $$;
```

**TL;DR:** PostgreSQL's `EXCEPTION WHEN` syntax maps directly. Use `STRICT` with `SELECT ... INTO` to mimic Oracle's `NO_DATA_FOUND` error. PostgreSQL employs standard SQLSTATE codes (like `'23505'` or the mnemonic `UNIQUE_VIOLATION`) for specific error handling.

---

### Accessing Error Info (`SQLERRM`, `SQLCODE` vs. `SQLSTATE`, `SQLERRMSG`)

PostgreSQL provides specific variables within the `EXCEPTION` block to access detailed error information, akin to Oracle's `SQLERRM` and `SQLCODE`.

| Oracle (PL/SQL)                                          | Postgres (PL/pgSQL)                    | Notes / Key Differences                                                                                                                                 |
| :------------------------------------------------------- | :------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `SQLERRM` (error message)                                | `SQLERRM` (local variable `message`)   | Returns the primary error message string. `SQLERRMSG` can also be used as `SQLERRM`.                                                                    |
| `SQLCODE` (error number)                                 | `SQLSTATE` (local variable `sqlstate`) | Returns the 5-character SQLSTATE code (e.g., `'22012'` for division by zero). No direct numeric `SQLCODE` equivalent.                                   |
| Additional Info (e.g. `DBMS_UTILITY.FORMAT_ERROR_STACK`) | `GET STACKED DIAGNOSTICS ...`          | Provides access to more detailed error fields like `MESSAGE_TEXT`, `DETAIL`, `HINT`, `SCHEMA_NAME`, `TABLE_NAME`, etc. (Highly powerful for debugging!) |

**Example: Retrieving Error Details on Division By Zero (Northwind Data - simulated scenario)**

Let's intentionally trigger a division by zero and capture the error details.

```sql
-- Oracle PL/SQL Accessing Error Info
DECLARE
    v_numerator   NUMBER := 100;
    v_denominator NUMBER := 0;
    v_result      NUMBER;
    v_err_code    NUMBER;
    v_err_msg     VARCHAR2(500);
BEGIN
    v_result := v_numerator / v_denominator;
EXCEPTION
    WHEN ZERO_DIVIDE THEN
        v_err_code := SQLCODE;
        v_err_msg  := SQLERRM;
        DBMS_OUTPUT.PUT_LINE('Oracle: Caught ZERO_DIVIDE error. SQLCODE: ' || v_err_code || ', SQLERRM: ' || v_err_msg);
    WHEN OTHERS THEN
        v_err_code := SQLCODE;
        v_err_msg  := SQLERRM;
        DBMS_OUTPUT.PUT_LINE('Oracle: Caught other error. SQLCODE: ' || v_err_code || ', SQLERRM: ' || v_err_msg);
END;
/
```

```sql
-- PostgreSQL PL/pgSQL Accessing Error Info
DO $$
DECLARE
    v_numerator   NUMERIC := 100;
    v_denominator NUMERIC := 0; -- Use NUMERIC to ensure exact zero behavior, REAL/FLOAT might slightly differ.
    v_result      NUMERIC;
    v_sql_state   TEXT;
    v_sql_errm    TEXT;
    
    -- Additional diagnostics variables
    v_message_text  TEXT;
    v_detail_text   TEXT;
    v_hint_text     TEXT;
BEGIN
    v_result := v_numerator / v_denominator;
EXCEPTION
    WHEN DIVISION_BY_ZERO THEN -- Mnemonic for SQLSTATE '22012'
        v_sql_state := SQLSTATE; -- Automatic variable for SQLSTATE
        v_sql_errm  := SQLERRM;  -- Automatic variable for error message
        
        -- Get more detailed diagnostics
        GET STACKED DIAGNOSTICS 
            v_message_text = MESSAGE_TEXT,
            v_detail_text  = PG_EXCEPTION_DETAIL,
            v_hint_text    = PG_EXCEPTION_HINT;
            
        RAISE NOTICE 'Postgres: Caught DIVISION_BY_ZERO. SQLSTATE: %, SQLERRM: %', v_sql_state, v_sql_errm;
        RAISE NOTICE '  Message: %', v_message_text;
        RAISE NOTICE '  Detail: %', v_detail_text;
        RAISE NOTICE '  Hint: %', v_hint_text;
    WHEN OTHERS THEN
        RAISE NOTICE 'Postgres: Caught other error. SQLSTATE: %, SQLERRM: %', SQLSTATE, SQLERRM;
END $$;
```

**TL;DR:** PostgreSQL uses `SQLSTATE` and `SQLERRM` as special variables to retrieve error information. For comprehensive diagnostics, `GET STACKED DIAGNOSTICS` is indispensable, providing detailed contextual information.

---

### Reraising Exceptions (`RAISE` vs. `RAISE EXCEPTION`)

Both languages allow reraising the currently handled exception and explicitly raising new ones. PostgreSQL offers `RAISE EXCEPTION` for powerful custom error messaging.

| Oracle (PL/SQL)                              | Postgres (PL/pgSQL)                                                                     | Notes / Key Differences                                                                                                                             |
| :------------------------------------------- | :-------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| `RAISE;` (reraises current error)            | `RAISE;`                                                                                | Identical syntax for rethrowing the current exception. Useful in nested exception blocks.                                                           |
| `RAISE application_error_name;`              | `RAISE ERRCODE 'condition_code';`                                                       | For predefined/custom non-application errors.                                                                                                       |
| `RAISE_APPLICATION_ERROR(err_num, message);` | `RAISE EXCEPTION 'message' [USING ERRCODE = 'code', DETAIL = 'detail', HINT = 'hint'];` | PostgreSQL's `RAISE EXCEPTION` is highly versatile for creating rich, custom error messages. `ERRCODE` can be standard or custom (e.g., `'P0001'`). |

**Example: Invalid Employee ID and Custom Error (Northwind `employees`)**

Let's create a procedure to verify an employee ID. If the ID is invalid or outside an expected range (e.g., must be 1 to 9 based on our dataset), we'll raise custom errors.

```sql
-- Oracle PL/SQL Reraise & Custom Error
CREATE OR REPLACE PROCEDURE check_employee_id_oracle (p_employee_id IN employees.employee_id%TYPE)
IS
    v_employee_exists NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_employee_exists
    FROM employees
    WHERE employee_id = p_employee_id;

    IF v_employee_exists = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Employee with ID ' || p_employee_id || ' does not exist in Northwind.');
    END IF;

    IF p_employee_id > 9 OR p_employee_id < 1 THEN
        -- Using an Oracle system error code (conceptual, usually not recommended to re-use system codes)
        RAISE_APPLICATION_ERROR(-20002, 'Employee ID ' || p_employee_id || ' is out of typical Northwind range (1-9).');
    END IF;

    DBMS_OUTPUT.PUT_LINE('Oracle: Employee ' || p_employee_id || ' is valid.');
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Oracle: Caught error (re-raising). SQLCODE: ' || SQLCODE || ', SQLERRM: ' || SQLERRM);
        RAISE; -- Reraise the current exception
END;
/

-- Test in Oracle:
-- CALL check_employee_id_oracle(5);    -- Valid
-- CALL check_employee_id_oracle(999);  -- Employee does not exist
-- CALL check_employee_id_oracle(0);    -- Out of typical range (or employee not exist)
```

```sql
-- PostgreSQL PL/pgSQL Reraise & Custom Error
CREATE OR REPLACE PROCEDURE check_employee_id_pg (p_employee_id employees.employee_id%TYPE)
LANGUAGE plpgsql
AS $$
DECLARE
    v_employee_exists BOOLEAN;
BEGIN
    SELECT EXISTS (SELECT 1 FROM employees WHERE employee_id = p_employee_id) INTO v_employee_exists;

    IF NOT v_employee_exists THEN
        RAISE EXCEPTION 'Employee with ID % does not exist in Northwind.'
        USING HINT = 'Check the employee_id in the employees table.',
              ERRCODE = 'P0001'; -- Custom SQLSTATE for procedural exceptions (P-codes are for PL/pgSQL specific conditions)
    END IF;

    IF p_employee_id > 9 OR p_employee_id < 1 THEN
        RAISE EXCEPTION 'Employee ID % is out of typical Northwind range (1-9).'
        USING DETAIL = 'Expected ID between 1 and 9 for this context.',
              ERRCODE = '22003'; -- Using an SQLSTATE for "numeric_value_out_of_range"
    END IF;

    RAISE NOTICE 'Postgres: Employee % is valid.', p_employee_id;
EXCEPTION
    WHEN OTHERS THEN -- Catches any exception not handled by more specific blocks
        RAISE NOTICE 'Postgres: Caught error (re-raising). SQLSTATE: %, SQLERRM: %', SQLSTATE, SQLERRM;
        RAISE; -- Reraise the current exception
END;
$$;

-- Test in PostgreSQL:
-- CALL check_employee_id_pg(5);    -- Valid
-- CALL check_employee_id_pg(999);  -- Employee does not exist (custom error P0001)
-- CALL check_employee_id_pg(0);    -- Out of typical range (custom error 22003)
```

**TL;DR:** `RAISE;` offers direct reraising. For raising custom errors, `RAISE EXCEPTION` combined with the `USING` clause provides powerful and flexible control over the error message, detail, hint, and `SQLSTATE` code. Choose standard SQLSTATEs if appropriate, or use custom codes starting with 'P' for procedural language-specific errors.

---

### Custom Exceptions (`DEFINE EXCEPTION` vs. `RAISE EXCEPTION`)

Oracle supports declaring named exceptions (`MY_ERROR EXCEPTION; PRAGMA EXCEPTION_INIT(...)`). PostgreSQL, in contrast, handles "custom exceptions" more directly via the `RAISE EXCEPTION` statement by assigning a specific `ERRCODE`. You define the custom exception at the point of `RAISE` and then catch it by its associated `ERRCODE` or mnemonic.

| Oracle (PL/SQL)                            | Postgres (PL/pgSQL)                                                    | Notes / Key Differences                                                                                                       |
| :----------------------------------------- | :--------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------- |
| `MY_ERROR EXCEPTION;`                      | (No explicit type declaration)                                         | Custom exceptions are defined inline using `RAISE EXCEPTION`.                                                                 |
| `PRAGMA EXCEPTION_INIT(MY_ERROR, -20000);` | `USING ERRCODE = 'P0001';`                                             | PostgreSQL associates custom error conditions with SQLSTATE codes (starting with 'P' is typical for user-defined conditions). |
| `WHEN MY_ERROR THEN ...`                   | `WHEN SQLSTATE 'P0001' THEN ...` or `WHEN condition_mnemonic THEN ...` | Catch by the SQLSTATE code or, if a mnemonic exists for a P-code, use that.                                                   |

**Example: Invalid Discount Amount (Northwind `order_details`)**

Let's define a custom rule: discount should never exceed 50%. We'll enforce this using a custom exception.

```sql
-- Oracle PL/SQL Custom Exception
DECLARE
    -- Define a custom exception
    TOO_HIGH_DISCOUNT EXCEPTION;
    PRAGMA EXCEPTION_INIT(TOO_HIGH_DISCOUNT, -20003); -- Custom error number

    p_product_id  order_details.product_id%TYPE := 11;
    p_order_id    order_details.order_id%TYPE := 10248;
    new_discount  order_details.discount%TYPE; -- Trying with too high discount
BEGIN
    DBMS_OUTPUT.PUT_LINE('Oracle: Demonstrating Custom Exception');
    new_discount := 0.60; -- 60%

    IF new_discount > 0.5 THEN
        RAISE TOO_HIGH_DISCOUNT; -- Raise custom exception
    END IF;

    -- Update would occur here
    -- UPDATE order_details SET discount = new_discount WHERE product_id = p_product_id AND order_id = p_order_id;
    
    DBMS_OUTPUT.PUT_LINE('  Discount updated to: ' || (new_discount * 100) || '%');

EXCEPTION
    WHEN TOO_HIGH_DISCOUNT THEN
        DBMS_OUTPUT.PUT_LINE('  Caught Custom Error: Discount cannot exceed 50%.');
        -- Perform appropriate actions (log, rollback, etc.)
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('  Caught unexpected error: ' || SQLERRM);
END;
/
```

```sql
-- PostgreSQL PL/pgSQL Custom Exception
DO $$
DECLARE
    p_product_id  order_details.product_id%TYPE := 11;
    p_order_id    order_details.order_id%TYPE := 10248;
    new_discount  order_details.discount%TYPE; -- Trying with too high discount
BEGIN
    RAISE NOTICE 'Postgres: Demonstrating Custom Exception';
    new_discount := 0.60; -- 60%

    IF new_discount > 0.5 THEN
        -- Raise custom exception with a 'P' (procedural language specific condition) SQLSTATE
        RAISE EXCEPTION 'Discount amount of %% exceeds maximum allowed of 50%%.', (new_discount * 100)
        USING HINT = 'Ensure discount is not more than 0.50 (50%).',
              ERRCODE = 'P0002'; -- Another custom P-code
    END IF;
    
    -- Update would occur here
    -- UPDATE order_details SET discount = new_discount WHERE product_id = p_product_id AND order_id = p_order_id;

    RAISE NOTICE '  Discount updated to: %%.', (new_discount * 100);

EXCEPTION
    WHEN SQLSTATE 'P0002' THEN -- Catch by the custom SQLSTATE code
        RAISE NOTICE '  Caught Custom Error (SQLSTATE P0002): %', SQLERRM;
        RAISE NOTICE '  Hint: %', PG_EXCEPTION_HINT;
    WHEN OTHERS THEN
        RAISE NOTICE '  Caught unexpected error (SQLSTATE: %): %', SQLSTATE, SQLERRM;
END $$;
```

**TL;DR:** PostgreSQL handles custom exceptions by defining an `ERRCODE` within the `RAISE EXCEPTION` statement. You then catch these errors using `WHEN SQLSTATE 'Pxxxx'` or a recognized mnemonic if one is established. This pattern allows for clear separation and handling of application-specific errors within the PL/pgSQL environment.