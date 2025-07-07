# PostgreSQL JSON vs JSONB


### TL;DR:

1.  **`JSONB` is almost always preferred for storage and querying.**
    *   If you need to query the contents of your JSON data (e.g., filter by a value within the JSON, extract a specific key, check for containment), `JSONB` is the clear winner due to its query performance and indexing capabilities.
    *   The overhead of parsing on insert is negligible compared to the benefits during read operations for most applications.

2.  **Use `JSON` only in very specific, niche scenarios:**
    *   If you absolutely *must* preserve the exact original formatting, whitespace, or key order of the JSON string (e.g., for logging external API responses where the precise textual representation is critical, not just the data).
    *   If you're using the JSON column purely as a temporary staging area for input or output, and you plan to parse or process it elsewhere without performing complex queries directly within PostgreSQL.
    *   If you are receiving potentially invalid JSON and want to store it as-is, deferring validation to your application logic.

For typical application development where you want to store semi-structured data and query its contents efficiently within your database, **always choose `JSONB`**. It provides better performance, data integrity, and a richer set of optimized operators and functions for working with JSON data.


| Feature                     | `JSON` Data Type                                                                                               | `JSONB` Data Type                                                                              |
| :-------------------------- | :------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------- |
| **Storage Format**          | Stores an exact copy of the input JSON string as plain `TEXT`.                                                 | Stores JSON data in a decomposed binary format.                                                |
| **Parsing Time**            | Parses the JSON string *every time* it's accessed or queried.                                                  | Parses the JSON string *once* on input (insertion/update).                                     |
| **Query Performance**       | Slower for querying because it requires re-parsing the text for each operation.                                | Significantly faster for querying because it's already parsed and optimized for access.        |
| **Indexing**                | Limited indexing capabilities (e.g., `text_pattern_ops` on the whole column, but not on internal keys/values). | Supports powerful GIN indexes, allowing efficient querying of keys, values, and containment.   |
| **Storage Size**            | Can be larger due to preserving whitespace, original formatting, and duplicate keys.                           | Can be smaller due to optimization (no whitespace, internal sorting, no duplicate keys).       |
| **Input/Write Performance** | Slightly faster to insert because it's just saving a string.                                                   | Slightly slower to insert because it needs to parse and convert the JSON to its binary format. |
| **Key Order Preservation**  | Preserves the order of keys as they appeared in the original JSON string.                                      | Does **not** preserve key order; keys are typically sorted internally for efficiency.          |
| **Duplicate Keys**          | Allows duplicate keys in the input string (though standard behavior is to use the last value parsed).          | Eliminates duplicate keys, keeping only the last value provided for a given key.               |
| **Whitespace Preservation** | Preserves all whitespace from the original input.                                                              | Removes all insignificant whitespace.                                                          |
| **Data Validation**         | Validates only upon attempted access or parsing functions; invalid JSON might be stored.                       | Performs strict validation upon insertion; invalid JSON will throw an error immediately.       |

---

