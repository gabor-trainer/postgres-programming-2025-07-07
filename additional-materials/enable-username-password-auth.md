Enable **username/password authentication** (`md5` or better: `scram-sha-256`) for the `postgres` user in PostgreSQL:

---

### **1\. Locate the `pg_hba.conf` File**

The `pg_hba.conf` file configures authentication methods for PostgreSQL connections. To find its location, run:

```
psql -U postgres -c "SHOW hba_file;"

```

Typical locations:

- `/var/lib/pgsql/data/pg_hba.conf`
- `/etc/postgresql/{version}/main/pg_hba.conf`

---

### **2\. Edit the `pg_hba.conf` File**

Modify the file to use `md5` or `scram-sha-256` authentication for the `postgres` user.

Open the file with a text editor:

```
sudo nano /var/lib/pgsql/data/pg_hba.conf

```

Look for a line similar to:

```
local   all             postgres                                peer

```

Change `peer` to `md5` (or `scram-sha-256` if your PostgreSQL supports it):

```
local   all             postgres                                md5

```

Explanation of fields:

- `local`: Refers to local connections (via UNIX socket).
- `all`: Applies to all databases.
- `postgres`: Specifies the `postgres` user.
- `md5`: Requires the user to authenticate with a password.

---

### **3\. Reload PostgreSQL Configuration**

After editing the file, reload the PostgreSQL service to apply changes:

```
sudo systemctl reload postgresql

```

---

### **4\. Set a Password for the `postgres` User**

If the `postgres` user does not already have a password, set one:

1.  Switch to the `postgres` OS user:

    ```
    sudo -i -u postgres

    ```

2.  Open the PostgreSQL prompt:

    ```
    psql

    ```

3.  Set a password for the `postgres` user:

    ```
    ALTER USER postgres PASSWORD 'your_password';

    ```

4.  Exit the prompt:

    ```
    \q

    ```

---

### **5\. Test the Connection**

Try connecting using the `postgres` user with the password:

```
psql -U postgres -W

```

The `-W` flag forces a password prompt. Enter the password you set to verify the setup.

---

### **Optional: Enable Password Authentication for All Users**

If you want all users to authenticate with a username and password, update the `pg_hba.conf` file:

1.  Modify the `local` line for all users:

    ```
    local   all             all                                     md5

    ```

2.  Reload PostgreSQL:

    ```
    sudo systemctl reload postgresql

    ```

Now, all users will require a password to connect locally.
