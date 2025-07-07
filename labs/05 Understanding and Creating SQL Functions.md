# Understanding and Creating SQL Functions

**Objective:**
This lab focuses on understanding and creating server-side SQL functions in PostgreSQL using the Northwind database. You will learn the basic function definition syntax, how to handle parameters, various return types (scalar, set of elements, table), and the concept of polymorphic functions, all while solving practical Northwind data challenges.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
    *   *(If not installed, please refer to the installation instructions from the previous interaction or a reliable source for Northwind PostgreSQL setup.)*
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line, DBeaver, VS Code with PostgreSQL extension).
4.  **Basic SQL Knowledge:** Familiarity with fundamental SQL syntax (`SELECT`, `FROM`, `WHERE`).

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run a simple query to confirm connectivity and data presence (e.g., `SELECT count(*) FROM customers;`). You should see a count reflecting the number of customers.

---

### **Lab Exercises:**

**Introduction to Tables Used:**
For this lab, we will primarily interact with the following Northwind tables:
*   `orders`: Contains general order information.
*   `order_details`: Stores individual product details for each order, including unit price, quantity, and discount.
*   `products`: Holds product names and their `discontinued` status.
*   `customers`: Contains customer details, including `company_name` and `country`.
*   `employees`: Stores employee names and their `reports_to` manager.

We will also create a temporary helper table named `ArchivedOrderDetails` for one of the exercises.

---

#### **Task 1: Basic Scalar Function - Calculate Order Line Item Net Total**

**Description:**
Create a SQL function named `calculate_line_item_net_total` that takes `unit_price` (real), `quantity` (smallint), and `discount` (real) as parameters. The function should calculate the net total amount for a single order line item, after applying the discount. The formula is `(unit_price * quantity) * (1 - discount)`. Reference the parameters by their names.

**SQL Query:**

```sql
CREATE OR REPLACE FUNCTION calculate_line_item_net_total(
    unit_price_param real,
    quantity_param smallint,
    discount_param real
)
RETURNS real AS $$
SELECT (unit_price_param * quantity_param) * (1 - discount_param);
$$ LANGUAGE SQL;

-- Test the function with example values from order_details for order_id 10248, product_id 11
SELECT calculate_line_item_net_total(14.0, 12, 0.0) AS net_total_example;
SELECT calculate_line_item_net_total(9.80, 10, 0.0) AS net_total_example;
SELECT calculate_line_item_net_total(34.7999992, 5, 0.0) AS net_total_example;

-- Apply to a real order detail
SELECT
    order_id,
    product_id,
    unit_price,
    quantity,
    discount,
    calculate_line_item_net_total(unit_price, quantity, discount) AS calculated_net_total
FROM
    order_details
WHERE
    order_id = 10248 AND product_id = 11;
```

**Expected Output/Explanation:**
The `CREATE FUNCTION` statement should execute successfully. The first test query (`SELECT calculate_line_item_net_total(10.00, 5, 0.1);`) should return `45.00`. The specific Northwind test should show `150` for product 11 in order 10248 (14 * 12 * (1-0) = 168).

**Learning Point:**
This task demonstrates the basic `CREATE FUNCTION` syntax, specifying a scalar `RETURNS` type, using the `LANGUAGE SQL` keyword, and referencing parameters by their defined names within the function body.

---

#### **Task 2: Scalar Function with Positional Parameters - Get Employee Full Name**

**Description:**
Create a SQL function named `get_employee_full_name` that takes `first_name` (character varying) and `last_name` (character varying) as parameters. This function should concatenate these into a single full name string (e.g., "Nancy Davolio"). This time, reference the parameters using positional notation (`$1`, `$2`).

**SQL Query:**

```sql
CREATE OR REPLACE FUNCTION get_employee_full_name(
    character varying,
    character varying
)
RETURNS character varying AS $$
SELECT $1 || ' ' || $2;
$$ LANGUAGE SQL;

-- Test the function with an example
SELECT get_employee_full_name('Nancy', 'Davolio') AS employee_name;

-- Apply to actual employee data
SELECT
    employee_id,
    get_employee_full_name(first_name, last_name) AS full_name,
    title
FROM
    employees
WHERE
    employee_id = 1;
```

**Expected Output/Explanation:**
The `CREATE FUNCTION` statement should execute successfully. The test query should return "Nancy Davolio". For `employee_id = 1`, the full name will be "Nancy Davolio".

**Learning Point:**
This task demonstrates an alternative way to reference function parameters using positional indicators (`$1`, `$2`, etc.), which is a common practice in older SQL code or when parameter names are not explicitly defined in the function signature. It also reinforces string concatenation with `||`.

---

#### **Task 3: Function Returning a Set of Elements - List Discontinued Product IDs**

**Description:**
Create a SQL function named `get_discontinued_product_ids` that takes no parameters. This function should return a set of `product_id`s (smallint) for all products in the `products` table that are currently marked as discontinued (`discontinued = 1`).

**SQL Query:**

```sql
CREATE OR REPLACE FUNCTION get_discontinued_product_ids()
RETURNS setof smallint AS $$
SELECT product_id
FROM products
WHERE discontinued = 1;
$$ LANGUAGE SQL;

-- Call the function to see the results
SELECT get_discontinued_product_ids();
```

**Expected Output/Explanation:**
The function creation should succeed. The `SELECT` statement will return a list of `product_id`s (e.g., 1, 2, 5, 9, 17, 24, 28, 29, 53) in a single column. The `setof` keyword indicates that the function will return zero or more rows of the specified type.

**Learning Point:**
This task introduces the `RETURNS setof type` clause, which is used when a function is expected to return multiple rows of a single data type. This is particularly useful for encapsulating queries that produce a list of identifiers or simple values.

---

#### **Task 4: Function Returning a Table - Get Customer Contact Details by Country**

**Description:**
Create a SQL function named `get_customer_contacts_by_country` that takes `p_country` (character varying) as a parameter. The function should return a table containing `customer_id` (character varying), `company_name` (character varying), and `contact_name` (character varying) for all customers located in the specified country.

**SQL Query:**

```sql
CREATE OR REPLACE FUNCTION get_customer_contacts_by_country(p_country character varying)
RETURNS TABLE (
    ret_customer_id character varying,
    ret_company_name character varying,
    ret_contact_name character varying
) AS $$
SELECT
    customer_id,
    company_name,
    contact_name
FROM
    customers
WHERE
    country = p_country;
$$ LANGUAGE SQL;

-- Call the function to retrieve customers from 'Germany'
SELECT * FROM get_customer_contacts_by_country('Germany');

-- You can also use it in a JOIN like a regular table (though not specifically required by task description)
-- SELECT o.order_id, co.ret_company_name
-- FROM orders o
-- JOIN get_customer_contacts_by_country('France') co ON o.customer_id = co.ret_customer_id
-- LIMIT 5;
```

**Expected Output/Explanation:**
The function should be created successfully. When called with 'Germany', the result will be a table-like output with `ret_customer_id`, `ret_company_name`, and `ret_contact_name` columns for customers like 'ALFKI', 'BLAUS', 'FRANK', etc.

**Learning Point:**
This task demonstrates `RETURNS TABLE (column_name type, ...)` which allows a SQL function to return a result set with multiple columns, similar to a regular `SELECT` query. This is a powerful feature for encapsulating complex data retrieval logic.

---

#### **Task 5: Function Returning a Table (with DML) - Archive Old Order Details**

**Description:**
First, create a temporary table named `ArchivedOrderDetails` with the same column structure as the `order_details` table. Then, create a SQL function named `archive_order_details_before_date` that takes `p_archive_date` (date) as a parameter. This function should:
1.  **Move** records: Delete all `order_details` records where the `order_date` (from the `orders` table) is strictly *before* `p_archive_date`.
2.  **Insert into Archive:** Insert the deleted records into the `ArchivedOrderDetails` temporary table.
3.  **Return:** Return a table containing the `order_id` (smallint) and `product_id` (smallint) of the *moved* records.

**SQL Query:**

```sql
-- 1. Create the temporary archive table
DROP TABLE IF EXISTS ArchivedOrderDetails;
CREATE TEMPORARY TABLE ArchivedOrderDetails (LIKE order_details INCLUDING ALL);

-- 2. Create the function
CREATE OR REPLACE FUNCTION archive_order_details_before_date(p_archive_date date)
RETURNS TABLE (archived_order_id smallint, archived_product_id smallint) AS $$
WITH deleted_records AS (
    DELETE FROM order_details od
    USING orders o
    WHERE od.order_id = o.order_id
      AND o.order_date < p_archive_date
    RETURNING od.* -- Return all columns of the deleted order_details record
)
INSERT INTO ArchivedOrderDetails
SELECT * FROM deleted_records
RETURNING order_id, product_id; -- Return only order_id and product_id for the function output
$$ LANGUAGE SQL;

-- Before calling: Check some existing data in order_details for 1996 orders
SELECT od.order_id, od.product_id, o.order_date
FROM order_details od JOIN orders o ON od.order_id = o.order_id
WHERE o.order_date < '1997-01-01'
LIMIT 5;

-- Call the function to archive orders before '1997-01-01'
SELECT * FROM archive_order_details_before_date('1997-01-01');

-- After calling: Verify some records are moved from order_details
SELECT od.order_id, od.product_id, o.order_date
FROM order_details od JOIN orders o ON od.order_id = o.order_id
WHERE o.order_date < '1997-01-01'
LIMIT 5; -- This should now return fewer or no records

-- After calling: Verify some records are in ArchivedOrderDetails
SELECT * FROM ArchivedOrderDetails LIMIT 5;
```

**Expected Output/Explanation:**
The function creation should succeed. The `SELECT * FROM archive_order_details_before_date('1997-01-01');` call will output a table listing the `order_id` and `product_id` of all `order_details` records that were moved from the `order_details` table to `ArchivedOrderDetails` (these should correspond to all orders from 1996). Subsequent checks of `order_details` for 1996 orders should show them missing, while `ArchivedOrderDetails` should contain them.

**Learning Point:**
This task demonstrates combining `DELETE ... RETURNING` with `INSERT INTO ... SELECT` within a single SQL function, using a Common Table Expression (CTE) (`WITH deleted_records AS (...)`) to manage the intermediate results. It shows how DML operations can be part of a function's logic and still return a result set.

---

#### **Task 6: Polymorphic SQL Function - Generic Null Value Replacement (NVL)**

**Description:**
Create a polymorphic SQL function named `nvl_northwind` that takes two parameters, both of `anyelement` type. The function should return the value of the first parameter if it is not `NULL`, otherwise, it should return the value of the second parameter.

**SQL Query:**

```sql
CREATE OR REPLACE FUNCTION nvl_northwind(val anyelement, default_val anyelement)
RETURNS anyelement AS $$
SELECT COALESCE(val, default_val);
$$ LANGUAGE SQL;

-- Test with an integer example
SELECT nvl_northwind(NULL::integer, 0::integer) AS result_int;
SELECT nvl_northwind(123::integer, 0::integer) AS result_int;

-- Test with a text example
SELECT nvl_northwind(NULL::text, 'N/A'::text) AS result_text;
SELECT nvl_northwind('Some text'::text, 'N/A'::text) AS result_text;

-- Apply to Northwind data: Display employee reports_to, replacing NULL with -1
SELECT
    employee_id,
    first_name,
    last_name,
    reports_to,
    nvl_northwind(reports_to, -1::smallint) AS managed_by
FROM
    employees;

-- Apply to Northwind data: Display customer fax, replacing NULL with 'No Fax'
SELECT
    customer_id,
    company_name,
    fax,
    nvl_northwind(fax, 'No Fax'::character varying) AS customer_fax_display
FROM
    customers
LIMIT 5;
```

**Expected Output/Explanation:**
The function creation should succeed.
*   `SELECT nvl_northwind(NULL::integer, 0::integer)` returns `0`.
*   `SELECT nvl_northwind(123::integer, 0::integer)` returns `123`.
*   `SELECT nvl_northwind(NULL::text, 'N/A'::text)` returns `'N/A'`.
*   The `employees` query will show `-1` for Andrew Fuller's `managed_by` column, as his `reports_to` is `NULL`.
*   The `customers` query will show `'No Fax'` for customers who do not have a fax number.

**Learning Point:**
This task demonstrates the use of the `anyelement` polymorphic type, allowing the function to work with any data type dynamically. It introduces the `COALESCE` function, a powerful built-in function for returning the first non-NULL expression in a list, making it ideal for implementing `NVL`-like behavior.

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **Function for Customer Details:** Create a SQL function that takes a `customer_id` (character varying) and returns a table with the customer's `company_name`, `contact_name`, `city`, and `country`.
2.  **Total Order Value Function:** Create a scalar SQL function `get_order_total_value` that takes an `order_id` (smallint) and returns the sum of all `net_total`s for that order, using the `calculate_line_item_net_total` function from Task 1 (you will need to aggregate these values).
3.  **Clean up Archive Table:** Drop the `ArchivedOrderDetails` table after you are done with the lab.

**Further Exploration:**
*   **`PL/pgSQL`:** Research when and why you would choose `PL/pgSQL` over a simple SQL function (e.g., for procedural logic, loops, explicit variable declaration, error handling, complex branching).
*   **Immutability, Stability, and Volatility:** Learn about `IMMUTABLE`, `STABLE`, and `VOLATILE` function properties and how they affect query optimization and when to apply them.
*   **Function Overloading:** Explore how PostgreSQL allows you to define multiple functions with the same name but different parameter data types or numbers of parameters.
*   **Dropping Functions:** Practice removing functions using `DROP FUNCTION function_name(param_types);` (e.g., `DROP FUNCTION calculate_line_item_net_total(real, smallint, real);`).

---

### **Conclusion:**

In this lab, you gained a solid understanding of creating and utilizing server-side SQL functions in PostgreSQL within the Northwind database context. You learned how to define functions that return scalar values, a set of elements, or an entire table. You practiced referencing parameters by name and position, and even created a polymorphic function for generic null value handling. These skills are fundamental for encapsulating reusable logic, improving code organization, and performing complex operations efficiently within your PostgreSQL database.