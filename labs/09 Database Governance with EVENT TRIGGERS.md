### Database Governance with `EVENT TRIGGERS`

**Objective:**
This lab provides a detailed, hands-on exploration of PostgreSQL `EVENT TRIGGERS`. You will learn how to create event trigger functions, define triggers for DDL events (`ddl_command_start`, `ddl_command_end`, `sql_drop`), utilize special trigger context variables (`TG_EVENT`, `TG_TAG`, `CURRENT_USER`) and functions (`pg_event_trigger_ddl_commands()`, `pg_event_trigger_dropped_objects()`), and use `RAISE EXCEPTION` to enforce database policies. These concepts will be applied to implement real-world scenarios like DDL auditing, preventing critical object drops, enforcing naming conventions, and automating post-creation grants within the Northwind database.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
    *   *(If not installed, please refer to the installation instructions from the previous interaction or a reliable source for Northwind PostgreSQL setup.)*
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line, DBeaver, VS Code with PostgreSQL extension).
4.  **Basic SQL Knowledge:** Familiarity with fundamental SQL syntax.
5.  **Basic PL/pgSQL Knowledge:** Understanding of `DECLARE`, `BEGIN...END`, `SELECT INTO`, `IF` statements, `RAISE EXCEPTION`.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run a simple query to confirm connectivity and data presence (e.g., `SELECT count(*) FROM customers;`). You should see a count reflecting the number of customers.
3.  **Enable PL/pgSQL:** Ensure PL/pgSQL is active (`CREATE EXTENSION IF NOT EXISTS plpgsql;`).

---

### **Lab Exercises:**

**Introduction to Tables Used:**
*   `DDL_Audit_Log` (new table): To store audit trails of DDL commands.
*   `ProtectedTables` (new table): To list tables that cannot be dropped.
*   `Categories` (existing table): A critical Northwind table.
*   `Customers` (existing table): A critical Northwind table.

---

#### **Task 1: DDL Auditing (`ON ddl_command_end`)**

**Description:**
Northwind requires comprehensive auditing of all schema changes. Implement an event trigger that logs every `CREATE TABLE`, `ALTER TABLE`, and `DROP TABLE` command into a dedicated audit log table. The log should capture who executed the command, the command tag, and details about the affected objects.

**Application Context:** This is crucial for compliance, security monitoring, and troubleshooting unexpected schema modifications in a production environment.

**Steps:**
1.  Create the `DDL_Audit_Log` table.
2.  Create a PL/pgSQL event trigger function.
3.  Create an `EVENT TRIGGER` that fires `ON ddl_command_end`.

**SQL Query:**

```sql
-- 1. Create the DDL Audit Log table
DROP TABLE IF EXISTS DDL_Audit_Log CASCADE;
CREATE TABLE DDL_Audit_Log (
    log_id SERIAL PRIMARY KEY,
    event_type TEXT,        -- e.g., 'ddl_command_end'
    command_tag TEXT,       -- e.g., 'CREATE TABLE', 'ALTER TABLE', 'DROP TABLE'
    command_details JSONB,  -- Detailed info about the DDL command
    executed_by TEXT,
    executed_at TIMESTAMP DEFAULT NOW()
);

-- 2. Create the event trigger function
CREATE OR REPLACE FUNCTION log_ddl_commands()
RETURNS EVENT_TRIGGER AS $$
BEGIN
    INSERT INTO DDL_Audit_Log (event_type, command_tag, command_details, executed_by)
    VALUES (
        TG_EVENT,
        TG_TAG,
        (SELECT jsonb_agg(
            jsonb_build_object(
                'command', c.command,
                'object_type', c.object_type,
                'object_identity', c.object_identity,
                'schema_name', c.schema_name,
                'object_name', c.object_name
            )
        ) FROM pg_event_trigger_ddl_commands() AS c),
        CURRENT_USER
    );
END;
$$ LANGUAGE plpgsql;

-- 3. Create the event trigger
DROP EVENT TRIGGER IF EXISTS trg_ddl_audit;
CREATE EVENT TRIGGER trg_ddl_audit
ON ddl_command_end
EXECUTE FUNCTION log_ddl_commands();

-- --- Test Cases ---
-- Test 1: Create a new temporary table
CREATE TEMPORARY TABLE TestTable1 (id INT, name TEXT);
SELECT * FROM DDL_Audit_Log WHERE command_tag = 'CREATE TABLE' AND executed_by = CURRENT_USER ORDER BY executed_at DESC LIMIT 1;
DROP TABLE TestTable1;

-- Test 2: Alter an existing table (e.g., customers)
ALTER TABLE customers ADD COLUMN temp_notes TEXT;
SELECT * FROM DDL_Audit_Log WHERE command_tag = 'ALTER TABLE' AND executed_by = CURRENT_USER ORDER BY executed_at DESC LIMIT 1;
ALTER TABLE customers DROP COLUMN temp_notes;

-- Test 3: Drop a temporary table
CREATE TEMPORARY TABLE TestTable2 (id INT);
DROP TABLE TestTable2;
SELECT * FROM DDL_Audit_Log WHERE command_tag = 'DROP TABLE' AND executed_by = CURRENT_USER ORDER BY executed_at DESC LIMIT 1;
```

**Expected Output/Explanation:**
*   Each DDL command (CREATE, ALTER, DROP) will execute successfully.
*   After each DDL operation, querying `DDL_Audit_Log` will show a new entry corresponding to the command, detailing the object affected, command type, and who ran it.
*   `pg_event_trigger_ddl_commands()` provides detailed information about the DDL statement that fired the trigger.

**Learning Point:**
This task demonstrates `CREATE EVENT TRIGGER ON ddl_command_end`, which fires *after* a DDL command has successfully completed. It showcases the use of `TG_EVENT`, `TG_TAG`, `CURRENT_USER`, and `pg_event_trigger_ddl_commands()` to gather comprehensive auditing information.

---

#### **Task 2: Preventing Critical Table Drops (`ON sql_drop`)**

**Description:**
Northwind's core business relies heavily on the `customers` and `categories` tables. Implement an event trigger that prevents any attempt to `DROP TABLE` on these specific tables. If such an attempt is made, an informative error message should be raised.

**Application Context:** This is a critical security and data integrity measure to prevent accidental or malicious deletion of essential database objects.

**Steps:**
1.  Create a temporary table `ProtectedTables` to list tables that should not be dropped.
2.  Create a PL/pgSQL event trigger function.
3.  Create an `EVENT TRIGGER` that fires `ON sql_drop`.

**SQL Query:**

```sql
-- 1. Create a temporary table to define protected tables
DROP TABLE IF EXISTS ProtectedTables CASCADE;
CREATE TEMPORARY TABLE ProtectedTables (
    schema_name TEXT NOT NULL,
    table_name TEXT NOT NULL,
    PRIMARY KEY (schema_name, table_name)
);
INSERT INTO ProtectedTables (schema_name, table_name) VALUES
    ('public', 'customers'),
    ('public', 'categories');

-- 2. Create the event trigger function
CREATE OR REPLACE FUNCTION prevent_critical_table_drops()
RETURNS EVENT_TRIGGER AS $$
DECLARE
    obj_record RECORD;
BEGIN
    FOR obj_record IN SELECT * FROM pg_event_trigger_dropped_objects()
    LOOP
        IF obj_record.object_type = 'table' AND obj_record.schema_name = 'public' THEN
            -- Check if the dropped table is in our protected list
            IF EXISTS (SELECT 1 FROM ProtectedTables WHERE schema_name = obj_record.schema_name AND table_name = obj_record.object_name) THEN
                RAISE EXCEPTION 'Attempt to drop protected table "%"."%" was blocked by event trigger.',
                                obj_record.schema_name, obj_record.object_name;
            END IF;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- 3. Create the event trigger
DROP EVENT TRIGGER IF EXISTS trg_prevent_drop;
CREATE EVENT TRIGGER trg_prevent_drop
ON sql_drop
EXECUTE FUNCTION prevent_critical_table_drops();

-- --- Test Cases ---
-- Test 1: Try to drop a protected table (e.g., customers) - should FAIL
DROP TABLE customers;
-- This should result in an error: "Attempt to drop protected table "public"."customers" was blocked by event trigger."

-- Test 2: Try to drop another protected table (e.g., categories) - should FAIL
DROP TABLE categories;

-- Test 3: Try to drop a non-protected table (e.g., employees_backup - create it first) - should SUCCEED
CREATE TABLE employees_backup AS SELECT * FROM employees;
DROP TABLE employees_backup;
-- This should succeed.
```

**Expected Output/Explanation:**
*   Test 1 & 2: Will result in `ERROR: Attempt to drop protected table "public"."customers" was blocked by event trigger.` (or categories). The tables will NOT be dropped.
*   Test 3: Will successfully drop `employees_backup`.

**Learning Point:**
This task demonstrates `CREATE EVENT TRIGGER ON sql_drop`, which fires *before* an object is actually dropped but *after* the `DROP` command has been parsed. It utilizes `pg_event_trigger_dropped_objects()` to identify the objects intended for dropping and `RAISE EXCEPTION` to prevent the DDL command from completing.

---

#### **Task 3: Enforcing New Table Naming Convention (`ON ddl_command_start`)**

**Description:**
Northwind's DBA team wants to enforce a strict naming convention: all new tables created in the `public` schema must start with the prefix `nw_` (e.g., `nw_inventory_log`). Implement an event trigger to block any `CREATE TABLE` command that violates this rule.

**Application Context:** Ensures consistent schema organization, making it easier to identify custom tables and maintain the database.

**Steps:**
1.  Create a PL/pgSQL event trigger function.
2.  Create an `EVENT TRIGGER` that fires `ON ddl_command_start`.

**SQL Query:**

```sql
-- 1. Create the event trigger function
CREATE OR REPLACE FUNCTION enforce_table_naming_convention()
RETURNS EVENT_TRIGGER AS $$
DECLARE
    obj_record RECORD;
BEGIN
    -- Only check for CREATE TABLE commands
    IF TG_TAG = 'CREATE TABLE' THEN
        FOR obj_record IN SELECT * FROM pg_event_trigger_ddl_commands() WHERE command_tag = 'CREATE TABLE'
        LOOP
            IF obj_record.object_type = 'table' AND obj_record.schema_name = 'public' THEN
                -- Check if table name starts with 'nw_' (case-insensitive)
                IF obj_record.object_name !~* '^nw_.*' THEN
                    RAISE EXCEPTION 'Table name violation: New tables in public schema must start with "nw_". (Attempted: "%.%") ',
                                    obj_record.schema_name, obj_record.object_name;
                END IF;
            END IF;
        END LOOP;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 2. Create the event trigger
DROP EVENT TRIGGER IF EXISTS trg_enforce_naming;
CREATE EVENT TRIGGER trg_enforce_naming
ON ddl_command_start
EXECUTE FUNCTION enforce_table_naming_convention();

-- --- Test Cases ---
-- Test 1: Create a table with correct naming convention (should SUCCEED)
CREATE TABLE nw_new_products (product_id INT, name TEXT);
DROP TABLE nw_new_products; -- Clean up

-- Test 2: Create a table with incorrect naming convention (should FAIL)
CREATE TABLE inventory_updates (id INT, date DATE);
-- This should result in an error: "Table name violation: New tables in public schema must start with "nw_". (Attempted: "public"."inventory_updates")"

-- Test 3: Create a table in a different schema (should SUCCEED, as rule is for 'public')
CREATE SCHEMA IF NOT EXISTS temp_schema;
CREATE TABLE temp_schema.my_temp_data (value TEXT);
DROP TABLE temp_schema.my_temp_data;
DROP SCHEMA temp_schema;
```

**Expected Output/Explanation:**
*   Test 1 & 3: Will execute successfully.
*   Test 2: Will result in `ERROR: Table name violation: New tables in public schema must start with "nw_". (Attempted: "public"."inventory_updates")`, preventing table creation.

**Learning Point:**
This task demonstrates `CREATE EVENT TRIGGER ON ddl_command_start`, which fires *before* the DDL command is executed. This makes it ideal for pre-validation and preventing commands. It uses `pg_event_trigger_ddl_commands()` to inspect the incoming DDL statement and `ILIKE` (case-insensitive LIKE) for pattern matching.

---

#### **Task 4: Post-Table Creation Grants (`ON ddl_command_end`)**

**Description:**
Northwind has a standard reporting process. Whenever a new table is created in the `public` schema, it should automatically grant `SELECT` permissions on that table to the `public` role, simplifying access for reporting tools or general users.

**Application Context:** Automates common post-DDL tasks, ensuring consistent access control and reducing manual configuration.

**Steps:**
1.  Create a PL/pgSQL event trigger function.
2.  Create an `EVENT TRIGGER` that fires `ON ddl_command_end`.

**SQL Query:**

```sql
-- 1. Create the event trigger function
CREATE OR REPLACE FUNCTION auto_grant_select_on_new_tables()
RETURNS EVENT_TRIGGER AS $$
DECLARE
    obj_record RECORD;
    grant_sql TEXT;
BEGIN
    -- Only act on CREATE TABLE commands
    IF TG_TAG = 'CREATE TABLE' THEN
        FOR obj_record IN SELECT * FROM pg_event_trigger_ddl_commands() WHERE command_tag = 'CREATE TABLE'
        LOOP
            IF obj_record.object_type = 'table' AND obj_record.schema_name = 'public' THEN
                -- Construct the GRANT statement dynamically
                grant_sql := 'GRANT SELECT ON TABLE ' || quote_ident(obj_record.schema_name) || '.' || quote_ident(obj_record.object_name) || ' TO public;';
                RAISE NOTICE 'Executing: %', grant_sql; -- Log for verification
                EXECUTE grant_sql; -- Execute the generated SQL statement
            END IF;
        END LOOP;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 2. Create the event trigger
DROP EVENT TRIGGER IF EXISTS trg_auto_grant_select;
CREATE EVENT TRIGGER trg_auto_grant_select
ON ddl_command_end
EXECUTE FUNCTION auto_grant_select_on_new_tables();

-- --- Test Cases ---
-- Test 1: Create a test table
CREATE TABLE nw_reporting_data (report_id INT PRIMARY KEY, metric TEXT);

-- Verify: Check permissions on the newly created table
-- In psql, use: \dp nw_reporting_data
-- In pgAdmin, right-click table -> Properties -> Privileges tab
-- You should see "public = SELECT" granted.
-- Try connecting as a non-superuser (e.g., a new user with no specific grants) and select from this table.
-- For example:
-- CREATE ROLE test_user LOGIN PASSWORD 'test';
-- \c - test_user (in psql)
-- SELECT * FROM northwind.public.nw_reporting_data; -- This should succeed
-- \c - postgres (back to superuser)
-- DROP ROLE test_user;

DROP TABLE nw_reporting_data; -- Clean up
```

**Expected Output/Explanation:**
*   The `CREATE TABLE` command will succeed.
*   You should see a `NOTICE` message (if your client displays notices) indicating the `GRANT` statement being executed.
*   Verifying the table's privileges will show that `SELECT` has been automatically granted to `public`.

**Learning Point:**
This task demonstrates `CREATE EVENT TRIGGER ON ddl_command_end` used for post-DDL automation.
*   It highlights how to generate and `EXECUTE` dynamic SQL (`grant_sql`) within a PL/pgSQL function.
*   `quote_ident()` is used to properly quote identifiers (like table names) to handle cases with special characters or reserved words, making the dynamic SQL safe.
*   `RAISE NOTICE` is used to provide informational output during trigger execution.

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **More Complex Naming Convention:** Modify Task 3 to enforce different prefixes based on the object type (e.g., `tbl_` for tables, `fn_` for functions, `vw_` for views).
2.  **Prevent Schema Truncation:** Create an event trigger to prevent `TRUNCATE TABLE` operations on the `orders` or `order_details` tables. (`ON truncate` event).
3.  **Automatic Index Creation:** When a new table is created, automatically create a `GIN` index on its first `JSONB` or `HSTORE` column if detected. This would require more complex parsing of `pg_event_trigger_ddl_commands()` output to identify column types.
4.  **Notification on DDL:** Instead of logging to a table (Task 1), modify the `log_ddl_commands` function to use `NOTIFY 'ddl_alert', '...'` so that an application listening for notifications can be immediately aware of schema changes.

**Further Exploration:**
*   **Event Trigger Restrictions:** Understand the limitations of event triggers (e.g., they cannot modify the data of the command that fired them directly, `ddl_command_start` cannot query data outside `pg_event_trigger_ddl_commands`).
*   **Ordering Event Triggers:** Learn about `ORDER` clause in `CREATE EVENT TRIGGER` if you have multiple triggers for the same event and their execution order matters.
*   **System Catalogs:** Explore other PostgreSQL system catalog views (like `pg_class`, `pg_attribute`, `pg_roles`, `pg_namespace`) to gain deeper insights into database objects that can be manipulated or audited by event triggers.

---

### **Conclusion:**

In this comprehensive lab, you gained deep practical experience with PostgreSQL `EVENT TRIGGERS`. You learned to define trigger functions and implement triggers for various DDL events. You mastered using special trigger variables and functions to audit schema changes, prevent critical object deletions, enforce naming conventions, and automate post-creation grants. This lab demonstrated how event triggers are an indispensable tool for database governance, security, and automation in a real-world PostgreSQL environment like Northwind.