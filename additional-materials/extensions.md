# Extensions additional material

### **1\. Open-Source Extensions (Part of the PostgreSQL Ecosystem)**

These extensions are widely used and typically open-source but may require installation if not included in your PostgreSQL distribution.

#### **PostGIS**

- Purpose: Adds spatial and GIS capabilities to PostgreSQL, enabling advanced geographic queries.
- Use Cases: GIS applications, geospatial data analysis, location-based services.

#### **pg_cron**

- Purpose: A job scheduling tool that allows you to run scheduled tasks directly in PostgreSQL.
- Use Cases: Automating maintenance tasks like vacuuming, backups, or periodic data aggregation.

#### **pg_partman**

- Purpose: Helps manage and automate the creation and maintenance of table partitions.
- Use Cases: Managing large tables efficiently using time-based or ID-based partitioning.

#### **TimescaleDB**

- Purpose: A time-series database extension for PostgreSQL, optimized for storing and querying time-series data.
- Use Cases: IoT applications, financial data, monitoring metrics.

#### **PL/pgSQL Procedural Languages**

- Purpose: Support for additional procedural languages like `plperl`, `plpython`, and `plv8`.
- Use Cases: Extending the functionality of PostgreSQL with custom stored procedures.

#### **pg_stat_statements**

- Purpose: Tracks and aggregates SQL query execution statistics.
- Use Cases: Performance monitoring and optimization.

---

### Third-Party Extensions\*\*

#### **pgAudit**

- Purpose: Provides detailed logging and auditing capabilities for PostgreSQL.
- Use Cases: Compliance with regulatory requirements, security auditing.

#### **PostgreSQL TDE (Transparent Data Encryption)**

- Purpose: Adds support for encryption of data at rest in PostgreSQL.
- Use Cases: Security-sensitive environments requiring encryption.

#### **oracle_fdw**

- Purpose: Foreign Data Wrapper for connecting PostgreSQL to Oracle databases.
- Use Cases: Data migration, federated queries.

#### **powa (PostgreSQL Workload Analyzer)**

- Purpose: A real-time performance monitoring and optimization tool.
- Use Cases: Performance analysis and troubleshooting.

#### **pgBadger**

- Purpose: A log analyzer for PostgreSQL, generating detailed reports on query performance and errors.
- Use Cases: Database performance monitoring and log analysis.

---

### Explicitely commercial, mainly only bundled with a distribution, like EDB or managed service like AWS or Google or Azure Postgres\*\*

#### a) **Aurora PostgreSQL Extensions (AWS)**

- Examples: `aurora_stat_utils`, `aurora_pg_buffercache`.
- Purpose: Extensions specific to AWS Aurora PostgreSQL for performance tuning and monitoring.
- Use Cases: Cloud-native PostgreSQL on AWS.

#### b) **Heap / RDS (Amazon RDS) Specific Tools**

- Purpose: AWS RDS and Azure-specific extensions for backups, monitoring, and cloud integrations.
- Use Cases: Managed PostgreSQL services.

#### c) **EDB Postgres Advanced Server Extensions**

- Examples: `edb_wait_states`, `edb_audit`.
- Purpose: Enhanced features like wait-state analysis, security, and Oracle compatibility.
- Use Cases: Enterprise-level database requirements.

# Example: Installing PostGIS

- **Linux (via Package Managers)**: Many PostgreSQL distributions include PostGIS as an optional package. For example:

  - On Debian/Ubuntu:

    ```
    sudo apt install postgis postgresql-postgis-X.Y

    ```

    Replace `X.Y` with your PostgreSQL version (e.g., `15`).

  - On Red Hat/CentOS:

    ```
    sudo dnf install postgis postgresql15-postgis

    ```

---

### **2\. Enabling PostGIS**

1.  Connect to your PostgreSQL instance using `psql` or a client like pgAdmin.
2.  Create a database if you haven't already:

    ```
    CREATE DATABASE my_geodb;

    ```

3.  Enable PostGIS in the database:

    ```
    \c my_geodb
    CREATE EXTENSION postgis;

    ```

4.  Optionally, enable additional PostGIS features:

    ```
    CREATE EXTENSION postgis_topology;
    CREATE EXTENSION postgis_raster;

    ```

---

### **3\. Verifying PostGIS Installation**

Check if PostGIS is installed and active:

```
SELECT postgis_version();
```

### **1\. Babelfish**

**Babelfish** is a partial SQL Server compatibility extension. It allows to understand (partially) SQL Server's **T-SQL** syntax, features.

#### **Key Features of Babelfish**:

- Supports **T-SQL syntax** (e.g., `SELECT TOP`, `GO`, `@@IDENTITY`).
- Implements SQL Server-like behaviors, including:
  - Data types: `VARCHAR(MAX)`, `DATETIME`, `UNIQUEIDENTIFIER`, etc.
  - Functions: `ISNULL`, `CONVERT`, `DATEADD`, etc.
  - SQL Server's procedural language (`BEGIN...END`, `WHILE`, etc.).
- Supports **SQL Server wire protocol**, allowing SQL Server applications and tools (e.g., SQL Server Management Studio) to connect directly to PostgreSQL without modification.
- Replicates **Microsoft SQL Server system catalogs** for compatibility with SQL Server metadata queries.

#### **Use Cases**:

- Simplifies the migration of SQL Server applications to PostgreSQL by reducing the need to rewrite application code.
- Allows SQL Server applications to interact with PostgreSQL without modification.

#### **Installation**:

Babelfish is an open-source extension. Follow these steps to install it:

1.  Install PostgreSQL (Babelfish supports PostgreSQL 13 and 14).
2.  Clone and build Babelfish from its repository:

    ```
    git clone https://github.com/babelfish-for-postgresql/babelfish_extensions.git
    cd babelfish_extensions
    make
    sudo make install

    ```

3.  Enable Babelfish in your PostgreSQL instance:

    ```
    CREATE EXTENSION babelfishpg_tsql;

    ```

4.  Configure PostgreSQL for Babelfish by setting specific parameters (refer to Babelfish documentation).

#### **Documentation**:

- [Babelfish for PostgreSQL GitHub](https://github.com/babelfish-for-postgresql/babelfish_extensions)
-

### **Orafce**

Orafce is a PostgreSQL extension to create some level of compatibility with Oracle database functions, syntax, and features.

1.  **Oracle-Compatible Functions**:

    - Implements common Oracle functions that are not natively available in PostgreSQL:
      - **String Functions**: `DECODE`, `NVL`, `SUBSTRB`
      - **Date Functions**: `ADD_MONTHS`, `NEXT_DAY`, `MONTHS_BETWEEN`, `LAST_DAY`
      - **Math Functions**: `CEIL`, `FLOOR`, `MOD`, `TRUNC`, `ROUND`
      - **Conversion Functions**: `TO_CHAR`, `TO_DATE`, `TO_NUMBER`

2.  **Oracle-Compatible Packages**:

    - Adds support for Oracle-like packages, such as:
      - **DBMS_OUTPUT**: Mimics Oracle’s debugging/logging functionality.
      - **DBMS_RANDOM**: Provides random number generation similar to Oracle.
      - **UTL_FILE**: Basic file handling functionality.

3.  **Behavioral Compatibility**:

    - Provides Oracle-like behaviors for NULL handling, default values, and data type conversions.
    - Adds support for the Oracle `DUAL` table.

4.  **Performance Features**:

    - Includes enhancements that allow Oracle developers to optimize queries and procedural code in PostgreSQL.

5.  **PL/pgSQL Enhancements**:

    - Improves PostgreSQL’s PL/pgSQL procedural language to make it closer to Oracle's PL/SQL.
