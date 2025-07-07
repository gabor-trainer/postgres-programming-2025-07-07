# Leveraging the `JSONB` Data Type for Flexible Data Models

**Objective:**
This lab will immerse you in the powerful `JSONB` data type in PostgreSQL. You will learn to create tables with `JSONB` columns, insert complex JSON documents, and perform advanced queries and updates on semi-structured data, simulating real-world scenarios such as tracking detailed product variants and comprehensive customer interaction logs within the Northwind context.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended). `JSONB` is a built-in data type and does not require enabling an extension.
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
*   `products`: We will extend product information by adding flexible variant data.
*   `customers`: We will store complex interaction histories for customers.
*   `ProductVariants` (new temporary table): To store varied attributes and options for Northwind products.
*   `CustomerInteractionLog` (new temporary table): To store semi-structured log entries for customer interactions.

---

#### **Task 1: Creating a Product Variants Table with JSONB**

**Description:**
Imagine Northwind wants to start tracking various configurations or "variants" for certain products (e.g., size, color, material, special edition details). These details are not standardized across all products. Create a new temporary table named `ProductVariants` to store this flexible data using a `JSONB` column.

The `ProductVariants` table should have the following columns:
*   `variant_id`: A unique identifier for each variant (auto-incrementing integer).
*   `product_id`: The ID of the base product (foreign key referencing `products`).
*   `variant_details`: A `JSONB` column to store a JSON object containing diverse variant attributes.

**SQL Query:**

```sql
DROP TABLE IF EXISTS ProductVariants CASCADE; -- CASCADE ensures dependent views/FKs are dropped too

CREATE TABLE ProductVariants (
    variant_id SERIAL PRIMARY KEY,
    product_id SMALLINT REFERENCES products(product_id),
    variant_details JSONB
);
```

**Expected Output/Explanation:**
The `CREATE TABLE` statement should execute successfully. No rows will be returned. You can verify the table creation by checking your SQL client's schema browser or by running `\dt ProductVariants` in `psql`.

**Learning Point:**
This task introduces the `JSONB` data type declaration in a `CREATE TABLE` statement. `JSONB` stores JSON data in a decomposed binary format, allowing for efficient indexing and faster processing compared to `JSON` (which stores as plain text).

---

#### **Task 2: Inserting JSONB Data (Objects and Arrays)**

**Description:**
Insert sample `JSONB` data into the `ProductVariants` table, demonstrating both simple JSON objects and nested structures/arrays. Link these variants to existing Northwind products.

*   For 'Chang' (product\_id 2):
    *   Variant 1: `{"size": "12 oz", "packaging": "bottle", "is_seasonal": false}`
    *   Variant 2: `{"size": "24 pack", "packaging": "carton", "promotions": ["Summer Sale", "Bulk Discount"]}`
*   For 'Tofu' (product\_id 14):
    *   Variant 1: `{"type": "firm", "certifications": ["Organic", "Non-GMO"]}`
    *   Variant 2: `{"type": "silken", "dietary_info": {"vegan": true, "gluten_free": true}}`

**SQL Query:**

```sql
INSERT INTO ProductVariants (product_id, variant_details) VALUES
    (2, '{"size": "12 oz", "packaging": "bottle", "is_seasonal": false}'),
    (2, '{"size": "24 pack", "packaging": "carton", "promotions": ["Summer Sale", "Bulk Discount"]}'),
    (14, '{"type": "firm", "certifications": ["Organic", "Non-GMO"]}'),
    (14, '{"type": "silken", "dietary_info": {"vegan": true, "gluten_free": true}}');

-- Verify insertions
SELECT * FROM ProductVariants;
```

**Expected Output/Explanation:**
You should see messages indicating the successful insertion of 4 rows. The `SELECT` statement will display these rows, with the `variant_details` column showing the JSON data. Note that `JSONB` might reorder keys or remove insignificant whitespace for efficiency.

**Learning Point:**
This task demonstrates how to insert JSON objects and arrays directly into a `JSONB` column using string literals cast with `::jsonb`. It showcases nesting objects and arrays within the `JSONB` structure.

---

#### **Task 3: Basic JSONB Querying - Extracting Values and Key Existence**

**Description:**
Query the `ProductVariants` table to extract specific values from the `JSONB` data and check for key existence.

1.  Retrieve `product_id` and the `size` of all product variants.
2.  Find all product variants that have a `promotions` key.
3.  Get the `packaging` for the variant with `variant_id` 1.

**SQL Query:**

```sql
-- 1. Get product_id and 'size' for all variants
SELECT
    product_id,
    variant_details ->> 'size' AS size_text -- ->> extracts as TEXT
FROM
    ProductVariants;

-- 2. Find variants with a 'promotions' key
SELECT
    product_id,
    variant_details
FROM
    ProductVariants
WHERE
    variant_details ? 'promotions'; -- ? checks for key existence

-- 3. Get 'packaging' for variant_id 1
SELECT
    variant_details ->> 'packaging' AS packaging_type
FROM
    ProductVariants
WHERE
    variant_id = 1;
```

**Expected Output/Explanation:**
1.  Returns `product_id` and the `size` (e.g., "12 oz", "24 pack") from each variant.
2.  Returns the variant for `product_id` 2 that has the "promotions" key.
3.  Returns "bottle" for `packaging_type`.

**Learning Point:**
This task introduces fundamental `JSONB` operators: `->>` to extract a JSON field as `TEXT`, and `?` to check for the existence of a top-level key.

---

#### **Task 4: Advanced JSONB Querying - Nested Paths and Containment**

**Description:**
Perform more complex queries involving nested JSON structures and checking for containment.

1.  Find all product variants that are "gluten_free".
2.  Find products that offer "Organic" certification.
3.  List variants that contain a `variant_details` object that includes `{"is_seasonal": false}`.

**SQL Query:**

```sql
-- 1. Find variants that are "gluten_free" (from nested 'dietary_info' object)
SELECT
    product_id,
    variant_details
FROM
    ProductVariants
WHERE
    variant_details #>> '{dietary_info, gluten_free}' = 'true'; -- #>> extracts nested path as TEXT

-- 2. Find products that offer "Organic" certification (from 'certifications' array)
SELECT
    product_id,
    variant_details
FROM
    ProductVariants
WHERE
    variant_details -> 'certifications' ? 'Organic'; -- -> extracts JSONB sub-element, then ? checks key/element existence

-- 3. List variants that contain {"is_seasonal": false}
SELECT
    product_id,
    variant_details
FROM
    ProductVariants
WHERE
    variant_details @> '{"is_seasonal": false}'::jsonb; -- @> checks if JSONB contains another JSONB
```

**Expected Output/Explanation:**
1.  Returns the variant for `product_id` 14 (silken tofu). ` #>>` is used for accessing values along a path (array of keys).
2.  Returns the variant for `product_id` 14 (firm tofu). This demonstrates combining `->` for sub-element extraction with `?` for element existence within a JSON array (treated as keys if array elements are strings).
3.  Returns the variant for `product_id` 2 (12 oz bottle). `@>` is a powerful operator for checking if one JSONB document contains another as a subset.

**Learning Point:**
This task introduces `JSONB` path operators (`#>>` for text extraction, `->` for JSONB object/array extraction) and the `@>` (contains) operator. It highlights how to query deeply nested or partially matched JSON structures.

---

#### **Task 5: Updating JSONB Data - Adding, Modifying, and Removing Keys**

**Description:**
Update the `JSONB` data in the `ProductVariants` table.

1.  For the `variant_id` 1 (Chai), add a new top-level key `origin` with value `India`.
2.  For the `variant_id` 2 (Chang), change the `packaging` value from `carton` to `can`.
3.  For the `variant_id` 3 (Tofu firm), remove the `certifications` array.

**SQL Query:**

```sql
-- 1. Add new key 'origin' to variant_id 1
UPDATE
    ProductVariants
SET
    variant_details = jsonb_set(variant_details, '{origin}', '"India"', true) -- true to create if not exists
WHERE
    variant_id = 1;

-- 2. Modify existing key 'packaging' for variant_id 2
UPDATE
    ProductVariants
SET
    variant_details = jsonb_set(variant_details, '{packaging}', '"can"', false) -- false to only modify existing
WHERE
    variant_id = 2;

-- 3. Remove 'certifications' array for variant_id 3
UPDATE
    ProductVariants
SET
    variant_details = variant_details - 'certifications'
WHERE
    variant_id = 3;

-- Verify changes
SELECT variant_id, variant_details FROM ProductVariants WHERE variant_id IN (1, 2, 3);
```

**Expected Output/Explanation:**
Each `UPDATE` statement should return "UPDATE 1". The final `SELECT` will show:
*   `variant_id` 1: `variant_details` now includes `"origin": "India"`.
*   `variant_id` 2: `packaging` is now `"can"`.
*   `variant_id` 3: The `certifications` key (and its array value) should be removed.

**Learning Point:**
This task demonstrates modifying `JSONB` data using `jsonb_set()` (for adding or changing values at a specific path) and the `-` operator (for removing top-level keys). `jsonb_set`'s last argument controls whether to create a new key if it doesn't exist.

---

#### **Task 6: Creating a Customer Interaction Log with JSONB Arrays of Objects**

**Description:**
Northwind wants to keep a flexible log of customer service interactions, where each interaction can have varying details. Create a new temporary table `CustomerInteractionLog`. Each log entry will contain a fixed set of fields (like customer_id, interaction_date) and a `JSONB` column to store an array of event objects for that specific interaction.

The `CustomerInteractionLog` table should have:
*   `log_id`: SERIAL primary key.
*   `customer_id`: FOREIGN KEY to `customers`.
*   `interaction_date`: DATE of the interaction.
*   `events`: `JSONB` column to store an array of event objects (e.g., `[{"type": "call", "duration": "5m"}, {"type": "email", "sentiment": "positive"}]`).

**SQL Query:**

```sql
DROP TABLE IF EXISTS CustomerInteractionLog CASCADE;

CREATE TABLE CustomerInteractionLog (
    log_id SERIAL PRIMARY KEY,
    customer_id VARCHAR(5) REFERENCES customers(customer_id),
    interaction_date DATE,
    events JSONB
);

-- Insert sample log data for ALFKI and ANATR
INSERT INTO CustomerInteractionLog (customer_id, interaction_date, events) VALUES
    ('ALFKI', '1997-01-20', '[
        {"type": "call", "agent": "Nancy Davolio", "duration_minutes": 10, "topic": "Order Status"},
        {"type": "email", "subject": "Order 10243 Update", "sent_by": "System"}
    ]'),
    ('ANATR', '1997-02-05', '[
        {"type": "chat", "agent": "Support Bot", "issue": "Payment Error", "resolution": "Redirected to human"},
        {"type": "call", "agent": "Andrew Fuller", "duration_minutes": 15, "feedback_rating": 4}
    ]'),
    ('ALFKI', '1997-03-15', '[
        {"type": "email", "subject": "Product Inquiry", "product_id": 11, "response_template": "A001"}
    ]');

-- Verify insertion
SELECT * FROM CustomerInteractionLog;
```

**Expected Output/Explanation:**
The table should be created and sample data inserted. The `events` column will display the JSON arrays containing multiple event objects.

**Learning Point:**
This task demonstrates creating a `JSONB` column to store an array of JSON objects, which is a common pattern for flexible logging or history tracking. It also shows inserting such complex JSON array literals.

---

#### **Task 7: Querying JSONB Arrays and Nested Array Elements**

**Description:**
Query the `CustomerInteractionLog` to extract information from the `events` array or filter based on its contents.

1.  Find all log entries where *any* event had `type` as 'call'.
2.  For `log_id` 1, list all `agent` names from its events.
3.  Find log entries where an event contains `{"issue": "Payment Error"}`.

**SQL Query:**

```sql
-- 1. Find log entries where any event was a 'call'
SELECT
    log_id,
    customer_id,
    interaction_date,
    events
FROM
    CustomerInteractionLog
WHERE
    events @> '[{"type": "call"}]'::jsonb; -- @> for array containment

-- 2. For log_id 1, list all 'agent' names from its events
SELECT
    log_id,
    event_obj ->> 'agent' AS agent_name -- Directly extract 'agent' from event_obj
FROM
    CustomerInteractionLog,
    jsonb_array_elements(events) AS event_obj -- event_obj is already a JSONB object
WHERE
    log_id = 1 AND event_obj ? 'agent'; -- Check if the 'agent' key exists within the current event_obj

-- 3. Find log entries where an event contains {"issue": "Payment Error"}
SELECT
    log_id,
    customer_id,
    interaction_date,
    events
FROM
    CustomerInteractionLog
WHERE
    events @> '[{"issue": "Payment Error"}]'::jsonb;
```

**Expected Output/Explanation:**
1.  Returns `log_id` 1 and 2 (both have a 'call' event). This shows using `@>` for checking if a JSONB array contains a specific element (even a partial match object).
2.  Returns 'Nancy Davolio' for `agent_name`. This demonstrates `jsonb_array_elements` to unnest the array and `jsonb_each_text` to extract keys from the resulting objects.
3.  Returns `log_id` 2 (ANATR's interaction). Similar to 1, but matching a more specific partial object.

**Learning Point:**
This task demonstrates querying JSONB arrays, specifically using `@>` for containment checks within arrays and `jsonb_array_elements` (a lateral join) to unnest array elements into individual rows, allowing for more granular querying and extraction within arrays.

---

### **Post-Lab Activities:**

**Optional Challenges:**
1.  **Combined Querying:** Join `CustomerInteractionLog` with the `customers` table. Find the company name of customers who had a 'chat' interaction.
2.  **JSONB Aggregation:** For each `product_id` in `ProductVariants`, list all unique `packaging` types found across all its variants. (Hint: Look into `jsonb_array_elements` combined with `DISTINCT` and `GROUP BY`).
3.  **Indexing `JSONB`:** Research `GIN` indexes for `JSONB` columns. Create a `GIN` index on `ProductVariants.variant_details` and `CustomerInteractionLog.events` and re-run some queries to see if performance improves (especially `WHERE` clauses using `?`, `?|`, `?&`, `@>`).
    *   Example: `CREATE INDEX idx_productvariants_details ON ProductVariants USING GIN (variant_details);`
4.  **JSONB_PRETTY:** Explore the `jsonb_pretty()` function to format your `JSONB` output for better readability.

**Further Exploration:**
*   **JSON vs JSONB:** Understand the trade-offs and when to use one over the other (hint: `JSONB` is almost always preferred for storage and querying).
*   **Other `JSONB` functions:** Explore `jsonb_object_keys()`, `jsonb_typeof()`, `jsonb_delete()`, and more.
*   **Nested Path Updates:** Experiment with `jsonb_set()` to update values within nested objects or arrays (e.g., updating `feedback_rating` for a specific event within the `events` array).
*   **Complex Filtering:** Try to find customers who had *both* a 'call' and an 'email' interaction on the *same day*.
*   **Dropping Temporary Tables:** Remember to `DROP TABLE ProductVariants;` and `DROP TABLE CustomerInteractionLog;` when you are done to clean up your database.

---

### **Conclusion:**

In this comprehensive lab, you gained hands-on experience with PostgreSQL's powerful `JSONB` data type. You successfully created tables with `JSONB` columns, inserted diverse JSON structures (objects and arrays of objects), and mastered various querying techniques using operators like `->>`, `#>>`, `?`, and `@>`. Furthermore, you practiced dynamic updates to `JSONB` data using `jsonb_set()` and the `-` operator. These skills are invaluable for modeling flexible and semi-structured data in real-world scenarios, such as managing product variants and customer interaction logs in a dynamic business environment like Northwind.