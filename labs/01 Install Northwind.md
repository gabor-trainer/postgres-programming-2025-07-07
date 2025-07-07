# Installing the Northwind database

First, ensure you have saved the provided Northwind SQL script to a known location on your computer, e.g., `C:\northwind.
sql`.

...or... download it from the [Northwind GitHub repository](https://raw.githubusercontent.com/pthom/northwind_psql/refs/heads/master/northwind.sql)

---

## 1. Install with pgAdmin

pgAdmin is a powerful graphical interface for managing PostgreSQL databases.

**Prerequisites:**
*   PostgreSQL server is running on your localhost (usually automatically after installation).
*   pgAdmin is installed and accessible.
*   You know the password for the `postgres` superuser (set during PostgreSQL installation).

**Steps:**

1.  **Open pgAdmin:**
    *   Search for "pgAdmin" in your Windows Start Menu and launch it.
    *   It might open in your web browser or as a standalone application, depending on the version.

2.  **Connect to Your PostgreSQL Server:**
    *   In the left-hand browser panel, you should see "Servers". If you've just installed PostgreSQL, there might already be a server listed (e.g., "PostgreSQL 1X" on `localhost`).
    *   If not, right-click on "Servers" and select **"Register" > "Server..."**.
    *   **General Tab:**
        *   **Name:** Give it a descriptive name, e.g., `My Local PostgreSQL`.
    *   **Connection Tab:**
        *   **Host name/address:** `localhost` (or `127.0.0.1`)
        *   **Port:** `5432` (this is the default, keep it unless you changed it).
        *   **Maintenance database:** `postgres` (this is the default database that always exists).
        *   **Username:** `postgres` (this is the default superuser).
        *   **Password:** Enter the password you set during PostgreSQL installation.
        *   Click **"Save"**.
    *   The server should now appear and connect successfully.

3.  **Create the Northwind Database:**
    *   Expand your connected server in the browser tree (e.g., `PostgreSQL 1X (localhost:5432)`).
    *   Right-click on **"Databases"**.
    *   Select **"Create" > "Database..."**.
    *   **General Tab:**
        *   **Database:** Type `northwind` (this will be the name of your new database).
        *   **Owner:** Select `postgres` (or your preferred user if you've created others).
    *   Click **"Save"**.
    *   You should now see `northwind` listed under "Databases".

4.  **Open the Query Tool for Northwind:**
    *   Right-click on the newly created **`northwind`** database.
    *   Select **"Query Tool"**. A new query editor window will open.

5.  **Load and Execute the SQL Script:**
    *   In the Query Tool, locate the **"Open File"** icon in the toolbar (it usually looks like a folder with an upward arrow, or a file icon with a plus sign, or sometimes just "File" > "Open File").
    *   Navigate to where you saved `northwind.sql` (e.g., `C:\northwind.sql`).
    *   Select the file and click **"Open"**. The entire script will be loaded into the query editor.
    *   Click the **"Execute/Play"** button (looks like a play triangle or a lightning bolt icon) in the Query Tool toolbar.
    *   Wait for the execution to complete. You should see a "Query returned successfully" message in the "Messages" tab at the bottom. There will be many `CREATE TABLE`, `INSERT`, `ALTER TABLE` messages.

6.  **Verify the Installation:**
    *   In the pgAdmin browser tree, expand the `northwind` database.
    *   Expand **"Schemas" > "public" > "Tables"**.
    *   You should now see all the Northwind tables (e.g., `categories`, `customers`, `orders`, `products`, etc.).
    *   To check data, right-click on any table (e.g., `customers`) and select **"View/Edit Data" > "All Rows"**. This will show you the data in the table.

---

## 2. Install with psql (Command Line)

psql is the official command-line interface for PostgreSQL.

**Prerequisites:**
*   PostgreSQL server is running.
*   You know the password for the `postgres` superuser.
*   The PostgreSQL `bin` directory (where `psql.exe` and `createdb.exe` are located) is either in your system's PATH environment variable, or you know its full path. A typical path is `C:\Program Files\PostgreSQL\1X\bin\` (replace `1X` with your PostgreSQL version, e.g., `16`).

**Steps:**

1.  **Open Command Prompt (or PowerShell):**
    *   Search for "cmd" or "PowerShell" in your Windows Start Menu and open it.

2.  **Navigate to the PostgreSQL bin Directory (if not in PATH):**
    *   If `psql` and `createdb` commands don't work directly, you'll need to navigate to their directory.
    *   Type: `cd "C:\Program Files\PostgreSQL\<YOUR_PG_VERSION>\bin"` (replace `<YOUR_PG_VERSION>` with your actual version, e.g., `16`).
        *   Example: `cd "C:\Program Files\PostgreSQL\16\bin"`
    *   Press Enter.

3.  **Create the Northwind Database:**
    *   Use the `createdb` command to create a new database.
    *   Type: `createdb -U postgres northwind`
        *   `-U postgres`: Specifies the user to connect as (the superuser `postgres`).
        *   `northwind`: The name of the new database.
    *   Press Enter.
    *   You will be prompted for the **Password for user postgres:**. Type your password and press Enter. (Note: The characters you type won't be displayed for security reasons).
    *   If successful, you'll just return to the command prompt without any error messages.

4.  **Load and Execute the SQL Script:**
    *   Use the `psql` command with the `-f` flag to execute the SQL file.
    *   Type: `psql -U postgres -d northwind -f "C:\northwind.sql"`
        *   `-U postgres`: Connects as the `postgres` user.
        *   `-d northwind`: Connects to the `northwind` database.
        *   `-f "C:\northwind.sql"`: Specifies the path to your Northwind SQL script. Make sure the path is correct and enclosed in double quotes if it contains spaces.
    *   Press Enter.
    *   You will be prompted for the **Password for user postgres:** again. Type your password and press Enter.
    *   The script will execute, and you will see many `CREATE TABLE`, `ALTER TABLE`, `INSERT`, and `ALTER TABLE` statements being printed to the console. This is normal.
    *   Wait for the command prompt to return, indicating the script has finished executing.

5.  **Verify the Installation:**
    *   To verify, you can connect to the `northwind` database using `psql` and check tables/data.
    *   Type: `psql -U postgres -d northwind`
    *   Press Enter.
    *   Enter your `postgres` password when prompted.
    *   You should now see the `northwind=#` prompt.
    *   To list tables, type: `\dt` and press Enter. You should see a list of Northwind tables.
    *   To check data in a table (e.g., `customers`), type: `SELECT count(*) FROM customers;` and press Enter. It should return the count of customers.
    *   To exit psql, type: `\q` and press Enter.

