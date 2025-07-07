# **Postgres Point-In-Time Recovery (PITR)**

#### **1\. Prerequisites**

Before you can perform PITR, ensure:

- **Continuous Archiving** is enabled: PostgreSQL writes Write-Ahead Logs (WAL) to an archive directory for recovery.
- **Base Backup** is available: A consistent backup of your database.

---

#### **2\. Enable WAL Archiving**

To set up WAL archiving:

1.  Edit your `postgresql.conf` file:

    ```
    wal_level = replica                 # Required for WAL archiving
    archive_mode = on                   # Enable archiving
    archive_command = 'cp %p /path/to/archive/%f'  # Archive WALs

    ```

    Replace `/path/to/archive/` with your desired archive location.

2.  Reload the configuration:

    ```
    pg_ctl reload -D /path/to/data

    ```

---

#### **3\. Take a Base Backup**

Use the `pg_basebackup` tool or a similar method to take a base backup:

```
pg_basebackup -D /path/to/backup -F tar -z -X fetch -P

```

- `-D`: Target directory for the backup.
- `-F tar`: Creates a tarball.
- `-z`: Compresses the backup.
- `-X fetch`: Includes WAL files required to make the backup consistent.

---

#### **4\. Identify the Target Point-In-Time**

Determine the point in time you want to recover to:

- **Specific Timestamp**: E.g., `2025-01-21 15:00:00`
- **Transaction ID (XID)**: E.g., `123456`
- **Recovery Target Name**: If you’ve set a named restore point using:

  ```
  SELECT pg_create_restore_point('my_restore_point');

  ```

---

#### **5\. Prepare for Recovery**

1.  **Stop the PostgreSQL Server**:

    ```
    pg_ctl stop -D /path/to/data

    ```

2.  **Restore the Base Backup**:

    - Extract the base backup (if compressed):

      ```
      tar -xvf /path/to/backup/base.tar.gz -C /path/to/data

      ```

3.  **Remove Unnecessary Files**:

    - Remove files like `postmaster.pid` from the data directory.

4.  **Restore WAL Files**:

    - Ensure the archived WAL files are accessible in the specified archive directory.

---

#### **6\. Configure `recovery.conf`**

Create or edit the `postgresql.auto.conf` file in the data directory (PostgreSQL 12+ uses `postgresql.auto.conf`, not `recovery.conf`):

1.  Set the recovery parameters:

    ```
    restore_command = 'cp /path/to/archive/%f %p'  # Command to fetch WAL files
    recovery_target_time = '2025-01-21 15:00:00'  # Desired recovery time
    recovery_target_action = 'pause'              # Pause at the target

    ```

    Replace `recovery_target_time` with your target point.

2.  If using a restore point, set:

    ```
    recovery_target_name = 'my_restore_point'

    ```

---

#### **7\. Start the Server in Recovery Mode**

Start PostgreSQL, and it will enter recovery mode:

```
pg_ctl start -D /path/to/data

```

PostgreSQL will replay WAL files until it reaches the specified target.

---

#### **8\. Verify the Recovery**

- PostgreSQL will pause when it reaches the recovery target if `recovery_target_action = 'pause'`.
- Check the logs for successful recovery messages.

---

#### **9\. Complete the Recovery**

- If satisfied with the recovery state, finalize it:

  ```
  SELECT pg_wal_replay_resume();  -- If paused

  ```

- Promote the server to end recovery and make it writable:

  ```
  pg_ctl promote -D /path/to/data

  ```

PostgreSQL will create a new timeline after recovery.

---

### **10\. Post-Recovery Steps**

- Verify the database state.
- Backup the recovered database to ensure you have a new recovery point.
- Re-enable WAL archiving if needed.
