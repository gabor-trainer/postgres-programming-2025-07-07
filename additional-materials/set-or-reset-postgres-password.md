# Set or reset the password for the `postgres` user:

---

### **1\. Switch to the `postgres` OS User**

The `postgres` database user is typically mapped to the `postgres` operating system user. To access it:

```
sudo -i -u postgres

```

---

### **2\. Access the PostgreSQL Shell**

Open the PostgreSQL interactive shell (`psql`):

```
psql

```

---

### **3\. Set or Reset the Password**

Run the following SQL command to set a new password for the `postgres` user:

```
ALTER USER postgres PASSWORD 'your_new_password';

```

Replace `your_new_password` with the desired password.

---

### **4\. Exit the PostgreSQL Shell**

After setting the password, exit the `psql` shell:

```
\q

```

---

### **5\. Test the Password**

You can now test the new password by connecting with the `postgres` user:

```
psql -U postgres -W

```

The `-W` flag forces a password prompt. Enter the password you just set.

---

### **Troubleshooting**

- If you encounter authentication issues, ensure the `pg_hba.conf` file is configured for `md5` or `scram-sha-256` authentication for the `postgres` user.
- Ensure the PostgreSQL service is reloaded after any changes to the `pg_hba.conf` file:

  ```
  sudo systemctl reload postgresql
  ```
