# Connect to a PostgreSQL database using a password

### 1\. **Basic `psql` Command**

Use the `-U` option to specify the username, and the `-d` option for the database name:

```
psql -U username -d database_name

```

When prompted, enter the password for the user.

### 2\. **Provide the Password Non-Interactively**

To avoid being prompted for a password, you can set the `PGPASSWORD` environment variable. **Be cautious about security risks when using this method**:

```
PGPASSWORD='your_password' psql -U username -d database_name

```

### 3\. **Use a `.pgpass` File**

You can store the password securely in a `.pgpass` file in your home directory:

1.  Create the file `.pgpass` in your home directory:

    ```
    touch ~/.pgpass

    ```

2.  Add the connection details to the file in this format:

    ```
    hostname:port:database:username:password

    ```

    Example:

    ```
    localhost:5432:mydb:myuser:mypassword

    ```

3.  Secure the file by restricting permissions:

    ```
    chmod 600 ~/.pgpass

    ```

4.  Use `psql` as usual:

    ```
    psql -U username -d database_name

    ```

PostgreSQL will automatically read the password from the `.pgpass` file.

### 4\. **Connect with Full Connection String**

You can specify all connection details, including the password, in the connection string. Again, this exposes your password and is not recommended in scripts or shared environments:

```
psql "postgresql://username:password@localhost:5432/database_name"

```

### 5\. **Interactive Prompt for Password**

If you don't want to store your password or use `PGPASSWORD`, simply run `psql` without setting it:

```
psql -U username -d database_name
```
