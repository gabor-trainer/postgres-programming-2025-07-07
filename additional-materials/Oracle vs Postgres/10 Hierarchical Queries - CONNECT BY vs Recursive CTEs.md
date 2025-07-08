## Hierarchical Queries: `CONNECT BY` vs. Recursive CTEs

Hierarchical queries are fundamental for traversing tree-structured data, such as organizational charts, bill-of-materials, or file systems. Oracle's `CONNECT BY` syntax is declarative and unique to Oracle, whereas PostgreSQL (and standard SQL) relies on the `WITH RECURSIVE` clause.

### Core Differences and Analogies

The primary distinction is in the construct used: a dedicated clause in Oracle versus a general-purpose CTE in PostgreSQL.

| Oracle `CONNECT BY`                     | Postgres Recursive CTE                                                                         | Notes / Key Differences                                                                                                                             |
| :-------------------------------------- | :--------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| `START WITH condition`                  | **Anchor Member** (first `SELECT` in CTE before `UNION ALL`)                                   | Defines the starting point(s) of the hierarchy (e.g., top-level managers, root folders).                                                            |
| `CONNECT BY PRIOR parent_id = child_id` | **Recursive Member** (second `SELECT` after `UNION ALL`) and its `JOIN` condition              | Defines the relationship between a parent row and its children. The recursive member refers to the CTE itself.                                      |
| `LEVEL` pseudocolumn                    | A column in the CTE to increment in the recursive member (`level + 1`)                         | Tracks the depth/level of a node in the hierarchy.                                                                                                  |
| `SYS_CONNECT_BY_PATH(expr, delimiter)`  | `ARRAY_AGG` then `ARRAY_TO_STRING()`, or manual string concatenation in the recursive member   | Constructs the path from the root to the current node. Prefer `ARRAY_AGG` for robust solutions.                                                     |
| `NOCYCLE` / `CONNECT_BY_IS_CYCLE`       | Manual cycle detection using `path_array @> ARRAY[node]` in `WHERE` clause of recursive member | Oracle's `NOCYCLE` implicitly handles cyclic data and identifies cycles. In PostgreSQL, this logic must be explicitly built into the recursive CTE. |

### Building a Hierarchy: Basic Traversal

The `employees` table in Northwind has `employee_id` and `reports_to`, which models a simple "reports-to" hierarchy. `Andrew Fuller` (Employee ID: 2) reports to nobody, making him the root.

#### Oracle: `START WITH ... CONNECT BY PRIOR`

```sql
-- Oracle PL/SQL: Simple Organizational Chart (Top-Down)
SELECT
    LPAD(' ', (LEVEL - 1) * 2) || first_name || ' ' || last_name AS org_chart,
    employee_id,
    reports_to,
    LEVEL AS depth
FROM
    employees
START WITH
    reports_to IS NULL -- Start with the employee(s) who report to no one (Andrew Fuller)
CONNECT BY PRIOR
    employee_id = reports_to -- Parent (PRIOR employee_id) is the manager of the current row's reports_to
ORDER SIBLINGS BY
    last_name, first_name; -- Sorts siblings within each level for consistent output
```

#### PostgreSQL: Recursive CTE

```sql
-- PostgreSQL PL/pgSQL: Simple Organizational Chart (Top-Down) using Recursive CTE
DO $$
BEGIN
    RAISE NOTICE '--- PostgreSQL Organizational Chart (Recursive CTE) ---';
    FOR rec IN
        WITH RECURSIVE emp_hierarchy AS (
            -- Anchor Member: Start with employees who report to nobody (CEO/Top Level)
            SELECT
                e.employee_id,
                e.first_name,
                e.last_name,
                e.reports_to,
                1 AS depth -- Initialize level for top-level employees
            FROM
                employees e
            WHERE
                e.reports_to IS NULL -- Andrew Fuller (ID: 2) is the root

            UNION ALL

            -- Recursive Member: Find employees who report to those in the current hierarchy set
            SELECT
                e.employee_id,
                e.first_name,
                e.last_name,
                e.reports_to,
                eh.depth + 1 AS depth -- Increment the level
            FROM
                employees e
            JOIN
                emp_hierarchy eh ON e.reports_to = eh.employee_id
        )
        -- Final Selection and Formatting
        SELECT
            LPAD('', (eh.depth - 1) * 2) || eh.first_name || ' ' || eh.last_name AS org_chart,
            eh.employee_id,
            eh.reports_to,
            eh.depth
        FROM
            emp_hierarchy eh
        ORDER BY
            eh.depth, eh.employee_id -- Consistent order for hierarchy
    LOOP
        RAISE NOTICE '% (ID: %, Reports To: %, Depth: %)',
            rec.org_chart, rec.employee_id, COALESCE(rec.reports_to::TEXT, 'N/A'), rec.depth;
    END LOOP;
END $$;
```
**Key Takeaway:** The core structure of a Recursive CTE consists of an **anchor member** (non-recursive query) and a **recursive member** (query referring to the CTE itself), combined with `UNION ALL`. The recursion continues until no new rows are generated by the recursive member.

---

### Constructing Hierarchy Paths

Obtaining the full path from the root to a specific node is a common hierarchical requirement.

#### Oracle: `SYS_CONNECT_BY_PATH`

```sql
-- Oracle PL/SQL: Hierarchy Path (Manager -> Employee)
SELECT
    employee_id,
    first_name || ' ' || last_name AS employee_name,
    LEVEL AS depth,
    SYS_CONNECT_BY_PATH(first_name || ' ' || last_name, ' -> ') AS hierarchy_path -- Build path with delimiter
FROM
    employees
START WITH
    reports_to IS NULL
CONNECT BY PRIOR
    employee_id = reports_to
ORDER SIBLINGS BY
    last_name, first_name;
```

#### PostgreSQL: Array Aggregation & `ARRAY_TO_STRING`

```sql
-- PostgreSQL PL/pgSQL: Hierarchy Path using Array in Recursive CTE
DO $$
BEGIN
    RAISE NOTICE '--- PostgreSQL Hierarchy Path (Recursive CTE) ---';
    FOR rec IN
        WITH RECURSIVE emp_hierarchy AS (
            -- Anchor Member: Initialize path with array containing the root's name
            SELECT
                e.employee_id,
                e.first_name,
                e.last_name,
                e.reports_to,
                1 AS depth,
                ARRAY[e.first_name || ' ' || e.last_name] AS path_names_array -- Path as an array of names
            FROM
                employees e
            WHERE
                e.reports_to IS NULL

            UNION ALL

            -- Recursive Member: Append current employee's name to the path array
            SELECT
                e.employee_id,
                e.first_name,
                e.last_name,
                e.reports_to,
                eh.depth + 1,
                eh.path_names_array || (e.first_name || ' ' || e.last_name) -- Concatenate new element to array
            FROM
                employees e
            JOIN
                emp_hierarchy eh ON e.reports_to = eh.employee_id
        )
        -- Final Selection: Convert array path to string
        SELECT
            eh.employee_id,
            eh.first_name || ' ' || eh.last_name AS employee_name,
            eh.depth,
            ARRAY_TO_STRING(eh.path_names_array, ' -> ') AS hierarchy_path -- Format the array into a string
        FROM
            emp_hierarchy eh
        ORDER BY
            eh.depth, eh.employee_id
    LOOP
        RAISE NOTICE 'ID: %, Name: %, Depth: %, Path: %',
            rec.employee_id, rec.employee_name, rec.depth, rec.hierarchy_path;
    END LOOP;
END $$;
```

**Key Takeaway:** PostgreSQL's approach involves propagating an array (or string) representing the path through the recursion. `ARRAY_AGG()` in the anchor/recursive members and `ARRAY_TO_STRING()` in the final selection provide robust path construction capabilities, surpassing `SYS_CONNECT_BY_PATH` in terms of flexibility (e.g., storing IDs, then names, then dates in the path, and manipulating as an array before final string conversion).

---

### Handling Cycles (`NOCYCLE` vs. Manual Cycle Detection)

Hierarchical data can sometimes contain cycles (e.g., employee A reports to B, B reports to C, and C reports back to A). This causes infinite loops in naive hierarchical queries.

| Oracle `NOCYCLE`                    | Postgres Recursive CTE Cycle Detection                                                   | Notes / Key Differences                                                                    |
| :---------------------------------- | :--------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------- |
| `CONNECT BY NOCYCLE ...`            | Requires explicit `WHERE` condition checking for cycles.                                 | Oracle provides a built-in mechanism.                                                      |
| `CONNECT_BY_IS_CYCLE` pseudocolumn. | Add a column to track `path_ids_array` and use `e.employee_id = ANY(eh.path_ids_array)`. | The responsibility falls to the CTE definition to prevent loops and identify cyclic nodes. |

**Example: Cycle Detection in PostgreSQL**

(Note: The Northwind `employees` table does not contain cycles, so this example is conceptual for demonstration of the technique rather than actually hitting a cycle.)

```sql
-- PostgreSQL PL/pgSQL: Recursive CTE with Cycle Detection (Conceptual Example)
DO $$
BEGIN
    RAISE NOTICE '--- PostgreSQL Cycle Detection (Conceptual Example) ---';
    -- Assume we have an employees_with_cycle table for testing
    -- CREATE TABLE employees_with_cycle (employee_id SMALLINT PRIMARY KEY, reports_to SMALLINT);
    -- INSERT INTO employees_with_cycle VALUES (1,'Alice',NULL), (2,'Bob',1), (3,'Carol',2), (4,'Dave',3), (5,'Eve',4), (6,'Frank',5);
    -- INSERT INTO employees_with_cycle VALUES (3,'Carol',4); -- Introducing a cycle: Carol reports to Dave, Dave reports to Carol
    
    FOR rec IN
        WITH RECURSIVE emp_hierarchy AS (
            -- Anchor Member: Initialize path with employee's ID to track visited nodes
            SELECT
                e.employee_id,
                e.first_name,
                e.last_name,
                e.reports_to,
                1 AS depth,
                ARRAY[e.employee_id] AS path_ids_array, -- Array to store visited employee_ids for cycle detection
                FALSE AS is_cycle -- Flag to identify if the current path leads to a cycle
            FROM
                employees e
            WHERE
                e.reports_to IS NULL -- Or wherever the root is

            UNION ALL

            -- Recursive Member: Check for cycles before adding new nodes
            SELECT
                e.employee_id,
                e.first_name,
                e.last_name,
                e.reports_to,
                eh.depth + 1,
                eh.path_ids_array || e.employee_id, -- Append current employee's ID to path
                (e.employee_id = ANY(eh.path_ids_array)) AS is_cycle -- Set is_cycle flag if current employee_id is already in the path
            FROM
                employees e
            JOIN
                emp_hierarchy eh ON e.reports_to = eh.employee_id
            WHERE NOT (e.employee_id = ANY(eh.path_ids_array)) -- STOP recursion on this branch if a cycle is detected
        )
        -- Final Selection
        SELECT
            eh.employee_id,
            eh.first_name || ' ' || eh.last_name AS employee_name,
            eh.depth,
            ARRAY_TO_STRING(eh.path_ids_array, ' -> ') AS path_of_ids,
            eh.is_cycle -- Output the cycle flag (will always be FALSE for normal runs if `WHERE` condition prevents extension)
        FROM
            emp_hierarchy eh
        ORDER BY
            eh.depth, eh.employee_id
    LOOP
        RAISE NOTICE 'ID: %, Name: %, Depth: %, Path: %, Cycle: %',
            rec.employee_id, rec.employee_name, rec.depth, rec.path_of_ids, rec.is_cycle;
    END LOOP;
END $$;
```

**Key Takeaway:** Unlike Oracle's built-in `NOCYCLE`, preventing infinite loops and identifying cycles in PostgreSQL recursive CTEs requires adding explicit logic. This typically involves including a column in the recursive CTE to track all parent IDs in the current path (`path_ids_array`). The recursion then stops a branch (`WHERE NOT (e.id = ANY(eh.path_ids_array))`) when a node in the path is encountered again. `is_cycle` flags can be managed similarly.

---

### Performance Considerations for Hierarchical Queries

*   **Indexing:** Proper indexing is paramount. An index on the foreign key column (`reports_to` in `employees`) and the primary key (`employee_id`) is essential for both Oracle `CONNECT BY` and PostgreSQL Recursive CTEs to perform efficiently.
*   **Cycles (PostgreSQL):** For very large datasets, ensure your cycle detection logic is efficient, as naive array manipulations (especially `ANY` comparisons for long paths) can become expensive. Sometimes a separate `(node_id, depth, path_array)` temporary table with unique constraints on path could be used in more advanced scenarios.
*   **Bottom-Up Queries:** While our examples are top-down (`START WITH reports_to IS NULL`), Recursive CTEs can also easily model bottom-up hierarchies by reversing the `JOIN` condition in the recursive member (`e.reports_to = eh.employee_id` to `e.employee_id = eh.reports_to`) and changing the anchor.
*   **Explain Analyze:** Always use `EXPLAIN ANALYZE` on your hierarchical queries to understand their execution plan and identify potential bottlenecks.