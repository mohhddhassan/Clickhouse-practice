## solution.md (Step-by-step solution)

---

### 0. Prerequisites

* A running ClickHouse server (docker container `clickhouse-dba` in this lab).
* `clickhouse-client` available inside the container.
* You are running commands from the host using `docker exec -it clickhouse-dba clickhouse-client`.

> NOTE: If your container name is different, replace `clickhouse-dba` accordingly.

---

### 1. Admin: Connect to ClickHouse shell (from host)

```bash
docker exec -it clickhouse-dba clickhouse-client
```

You should now see the ClickHouse prompt:

```
:)
```

---

### 2. Ensure the test database exists

```sql
CREATE DATABASE IF NOT EXISTS analytics_db;
SHOW DATABASES;
```

Confirm that `analytics_db` appears in the output.

---

### 3. Ensure a test table exists (for validation)

```sql
USE analytics_db;

CREATE TABLE IF NOT EXISTS user_events
(
    event_id UInt32,
    user_id UInt32,
    event_type String,
    event_time DateTime
)
ENGINE = MergeTree
ORDER BY event_time;

SHOW TABLES;
```

---

### 4. Insert sample data (if table is empty)

```sql
INSERT INTO user_events
SELECT
    number + 1 AS event_id,
    rand() % 100 + 1 AS user_id,
    arrayElement(['login','logout','click','purchase'], toUInt32(rand() % 4) + 1) AS event_type,
    now() - INTERVAL toInt32(rand() % 10000) SECOND AS event_time
FROM numbers(20);

-- Verify data
SELECT count() AS rows FROM user_events;
SELECT * FROM user_events LIMIT 5;
```

---

### 5. Create a reusable read-only role

```sql
CREATE ROLE IF NOT EXISTS analyst_readonly;
SHOW ROLES;
```

---

### 6. Grant read-only access to the role

```sql
GRANT SELECT ON analytics_db.* TO analyst_readonly;

-- Verify role privileges
SHOW GRANTS FOR ROLE analyst_readonly;
```

Expected output (similar):

```
GRANT SELECT ON analytics_db.* TO analyst_readonly
```

---

### 7. Create analyst users

```sql
CREATE USER IF NOT EXISTS analyst1 IDENTIFIED BY 'Analyst@123';
CREATE USER IF NOT EXISTS analyst2 IDENTIFIED BY 'Analyst@456';

SHOW USERS;
```

---

### 8. Assign the role to both users

```sql
GRANT analyst_readonly TO analyst1;
GRANT analyst_readonly TO analyst2;
```

---

### 9. Verify privilege inheritance for each user

```sql
SHOW GRANTS FOR analyst1;
SHOW GRANTS FOR analyst2;
```

Expected output (similar):

```
GRANT analyst_readonly TO analyst1
```

This confirms that both users inherit privileges from the role.

---

### 10. Test: Log in as analyst1 and run SELECT

From the **host terminal**:

```bash
docker exec -it clickhouse-dba clickhouse-client -u analyst1 --password
```

Enter password:

```
Analyst@123
```

Then run:

```sql
USE analytics_db;
SELECT * FROM user_events LIMIT 5;
```

 **Expected result:** Query executes successfully and returns rows.

---

### 11. Test: Attempt INSERT (should be denied)

As `analyst1`:

```sql
INSERT INTO user_events VALUES (9999, 1, 'test', now());
```

 **Expected error:**

```
DB::Exception: Not enough privileges
```

---

### 12. Test: Attempt ALTER (should be denied)

```sql
ALTER TABLE user_events ADD COLUMN test_col String;
```

**Expected error:**

```
DB::Exception: Not enough privileges
```

---

### 13. Repeat validation for analyst2 (optional but recommended)

```bash
docker exec -it clickhouse-dba clickhouse-client -u analyst2 --password
```

Password:

```
Analyst@456
```

Then:

```sql
USE analytics_db;
SELECT * FROM user_events LIMIT 5;
INSERT INTO user_events VALUES (8888, 2, 'fail_test', now());
```
---

### 14. Document the observed behavior

**Observations**

* A single role `analyst_readonly` was created and granted `SELECT` on `analytics_db.*`.
* Multiple users (`analyst1`, `analyst2`) inherited permissions automatically via role assignment.
* Both users were able to run `SELECT` queries successfully.
* Both users were blocked from executing `INSERT` and `ALTER`.
* This confirms that **role-based access control (RBAC) works correctly in ClickHouse**.

**Sample recorded evidence**

* `SHOW GRANTS FOR ROLE analyst_readonly;` → `GRANT SELECT ON analytics_db.* TO analyst_readonly`
* `SHOW GRANTS FOR analyst1;` → `GRANT analyst_readonly TO analyst1`
* `INSERT` attempt as analyst → `DB::Exception: Not enough privileges`
* `ALTER` attempt as analyst → same privilege error

---

### 15. Example Git commands to add this task to your repo

```bash
mkdir -p task-02-role-analyst-readonly
# Add README.md and this solution.md
git add task-02-role-analyst-readonly/README.md task-02-role-analyst-readonly/solution.md
git commit -m "Task 2: create analyst_readonly role and assign to multiple users"
git push
```

---

### 16. Troubleshooting tips

* If analysts can still INSERT/ALTER:

  ```sql
  SHOW GRANTS FOR analyst1;
  ```

  Check for accidental extra roles or direct grants.

* If login fails:

  * Recheck passwords.
  * Confirm user existence with:

    ```sql
    SHOW USERS;
    ```

* If privileges don’t reflect immediately:

  * Reconnect the user session.
  * Re-run `SHOW GRANTS`.

---

### 17. Optional: Commands used during the lab (compact summary)

```sql
CREATE DATABASE IF NOT EXISTS analytics_db;
USE analytics_db;

CREATE TABLE user_events (...) ENGINE = MergeTree ORDER BY event_time;
INSERT INTO user_events SELECT ... FROM numbers(20);

CREATE ROLE analyst_readonly;
GRANT SELECT ON analytics_db.* TO analyst_readonly;

CREATE USER analyst1 IDENTIFIED BY 'Analyst@123';
CREATE USER analyst2 IDENTIFIED BY 'Analyst@456';

GRANT analyst_readonly TO analyst1;
GRANT analyst_readonly TO analyst2;

SHOW GRANTS FOR ROLE analyst_readonly;
SHOW GRANTS FOR analyst1;
SHOW GRANTS FOR analyst2;
```

--
