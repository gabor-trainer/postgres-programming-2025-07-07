## Fundamental Differences

### The PL/pgSQL Philosophy: Functions vs. Procedures

One of the most fundamental differences lies in how PostgreSQL (historically) and Oracle handle procedural blocks, especially regarding transaction management.

*   **Oracle PL/SQL Paradigm:** `FUNCTION` and `PROCEDURE` are distinct concepts. `PROCEDURE`s often contain DDL or DML with `COMMIT`/`ROLLBACK` statements and implicitly return `VOID`. `FUNCTION`s are typically expected to return a value and be side-effect free, though technically, DML and `COMMIT`/`ROLLBACK` *can* be used inside them (often leading to undesirable results if called from SQL queries, which don't expect implicit transactions). `FUNCTION`s can also return `TABLE` types (`PIPE ROW`).

*   **PostgreSQL PL/pgSQL Paradigm (Historical & Core Difference):**
    Historically, PostgreSQL primarily used `FUNCTION`s for *all* stored procedural code. These `FUNCTION`s were designed to be part of an *outer transaction* and **cannot** perform transaction control (`COMMIT`/`ROLLBACK`). Attempting to do so in a `FUNCTION` will result in an error (`transaction control statements are not allowed in a user-defined function`). This means if an Oracle `PROCEDURE` relied on committing within its logic, a direct translation to a PostgreSQL `FUNCTION` will fail.

*   **PostgreSQL `PROCEDURE`s (PG 11+):**
    Starting with PostgreSQL 11, the `CREATE PROCEDURE` command was introduced. These new `PROCEDURE`s *can* execute transaction control statements (`COMMIT`/`ROLLBACK`), explicitly matching Oracle's `PROCEDURE` behavior. `PROCEDURE`s always return `VOID` (i.e., nothing). They cannot be called directly from `SELECT` statements; use the `CALL` statement.

This means you often have to re-evaluate Oracle `FUNCTION`/`PROCEDURE` design for PostgreSQL, especially pre-PG11.

**Transaction Management is Contextual!**
- **PL/pgSQL `FUNCTION`:** Implicitly runs inside a subtransaction if called from a transaction. **Cannot `COMMIT`/`ROLLBACK` or execute DDL directly.** Designed for computed values/non-transactional logic.
- **PL/pgSQL `PROCEDURE` (PG 11+):** Designed to encapsulate independent transaction blocks. **Can `COMMIT`/`ROLLBACK` and execute DDL.** Must be invoked with `CALL`.

---

### Core Language Naming & Case Sensitivity

PostgreSQL data types generally follow SQL standards and common usage, contrasting with some of Oracle's proprietary types. Case sensitivity also has nuances.

| Oracle (PL/SQL)               | Postgres (PL/pgSQL)                | Notes / Key Differences                                                                                                                                                  |
| :---------------------------- | :--------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NUMBER`, `NUMBER(p,s)`       | `NUMERIC`, `INT`, `SMALLINT`, etc. | PostgreSQL offers distinct numeric types; choose appropriate precision.                                                                                                  |
| `VARCHAR2(N)`                 | `VARCHAR(N)`, `TEXT`               | `VARCHAR` has a length limit, `TEXT` does not. `TEXT` is generally preferred if no length constraint is needed.                                                          |
| `NVARCHAR2(N)`                | `VARCHAR(N)`, `TEXT`               | PostgreSQL inherently handles Unicode in `VARCHAR`/`TEXT` (if DB encoding is UTF8).                                                                                      |
| `DATE`                        | `DATE`, `TIMESTAMP`                | Oracle's `DATE` is date *and* time. PostgreSQL separates `DATE` (date only) and `TIMESTAMP` (date and time).                                                             |
| Identifiers (Tables, Columns) | Identifiers                        | Oracle: Case-insensitive by default. PostgreSQL: Case-insensitive *unless double-quoted*. Double-quoted identifiers are case-sensitive. Prefer lowercase without quotes. |

**Watch Out For:** Quoting in PostgreSQL makes identifiers case-sensitive. Avoid it for database objects unless strictly necessary, and always use standard snake_case (or kebab-case for columns, if preferred) in development.

**Example: Variable Declarations**

```sql
-- Oracle PL/SQL
DECLARE
    -- Northwind Customers
    v_customer_id       VARCHAR2(5)   := 'ALFKI';
    v_company_name      VARCHAR2(40);
    
    -- Northwind Employees
    v_employee_id       NUMBER        := 1;
    v_employee_lastname VARCHAR2(20);
    
    -- Northwind Products
    v_product_id        NUMBER        := 11;
    v_unit_price        NUMBER(10,2); -- Or just NUMBER
    v_units_in_stock    NUMBER;       -- Or just NUMBER
    
    d_order_date        DATE;

BEGIN
    -- Your logic here
    NULL;
END;
/
```

```sql
-- PostgreSQL PL/pgSQL
DO $$
DECLARE
    -- Northwind Customers
    v_customer_id       VARCHAR(5)   := 'ALFKI'; -- Or just TEXT if no length limit needed often.
    v_company_name      VARCHAR(40);
    
    -- Northwind Employees
    v_employee_id       SMALLINT     := 1; -- As per schema: employee_id is SMALLINT
    v_employee_lastname VARCHAR(20); -- As per schema: character varying(20)
    
    -- Northwind Products
    v_product_id        SMALLINT     := 11; -- As per schema: smallint
    v_unit_price        REAL;         -- As per schema: real
    v_units_in_stock    SMALLINT;     -- As per schema: smallint
    
    d_order_date        DATE;         -- For date only; TIMESTAMP if time is involved
    
BEGIN
    -- Your logic here
    NULL;
END $$;
```

---

### Block Structure & Anonymous Blocks (`DO` Block)

Anonymous blocks (or "unnamed" blocks) are very common for ad-hoc scripts or testing in both databases. The syntax varies.

| Oracle (PL/SQL)            | Postgres (PL/pgSQL)                            | Notes / Key Differences                                                                                                                                                  |
| :------------------------- | :--------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DECLARE...BEGIN...END; /` | `DO $$DECLARE...BEGIN...END $$;`               | The `DO $$` syntax uses "dollar quoting" (`$$`) to delimit the code block, simplifying escaping of single quotes within.                                                 |
|                            | `LANGUAGE plpgsql` is implied for `DO` blocks. | PostgreSQL always requires a `LANGUAGE` clause for stored functions/procedures, often `plpgsql`. `DO` blocks implicitly use the default procedural language (`plpgsql`). |

**Example: Simple Anonymous Block (Hello World)**

Let's fetch and display data from our Northwind `employees` table.

```sql
-- Oracle PL/SQL (Anonymous Block)
DECLARE
    v_emp_id      employees.employee_id%TYPE := 1;
    v_first_name  employees.first_name%TYPE;
    v_last_name   employees.last_name%TYPE;
BEGIN
    SELECT first_name, last_name
    INTO v_first_name, v_last_name
    FROM employees
    WHERE employee_id = v_emp_id;

    DBMS_OUTPUT.PUT_LINE('Oracle: Hello, ' || v_first_name || ' ' || v_last_name || '!');
    -- You need to set serveroutput on in SQL*Plus/SQL Developer for this to show.
END;
/
```

```sql
-- PostgreSQL PL/pgSQL (DO block - Anonymous Block)
DO $$
DECLARE
    v_emp_id       employees.employee_id%TYPE := 1;
    v_first_name   employees.first_name%TYPE;
    v_last_name    employees.last_name%TYPE;
BEGIN
    SELECT first_name, last_name
    INTO v_first_name, v_last_name
    FROM employees
    WHERE employee_id = v_emp_id;

    RAISE NOTICE 'Postgres: Hello, % %!', v_first_name, v_last_name;
    -- This message will appear as a notice in the client (e.g., psql, pgAdmin messages tab).
END $$;
```

**Concise Bullet Points for Explanation:**
*   PostgreSQL's `DO` block provides functionality directly analogous to Oracle's anonymous `DECLARE`/`BEGIN`/`END` blocks for ad-hoc script execution.
*   The `$$` syntax in PostgreSQL is "dollar quoting". It acts as a configurable delimiter, making it easy to embed code or strings containing single quotes without needing to escape them (`' '`). For instance, you could use `$ANYTAG$` to define your block if `$$` itself appears in the code, but `$$` is common for top-level blocks.
*   PostgreSQL uses `RAISE NOTICE` for console output, similar to `DBMS_OUTPUT.PUT_LINE`. Note the variadic arguments (`%`) in `RAISE NOTICE`.


