# Task 1 — Create a New User with Limited Privileges (SELECT Only)

---

## README.md (Task description)

```
Task 1: Create a New User with Limited Privileges (SELECT Only)

Log in as an administrator user on the ClickHouse server.

Identify or create a test database (e.g., analytics_db) for this exercise.

Create a test table with a few sample records to work with. Use generateRandom() to insert random records

Create a new user with a secure password.

Grant this user only SELECT privileges on the chosen database.

Test that the user can successfully run a SELECT query.

Attempt non-SELECT operations such as INSERT or ALTER to confirm that access is denied.

Document the behavior and confirm that permission boundaries are enforced correctly.

Repository layout for this task:

task-01-select-only/
├─ README.md        # This file (task description)
├─ solution.md      # Step-by-step solution and commands you executed
```

---

## solution.md (Step-by-step solution)

### 0. Prerequisites

* A running ClickHouse server (docker container `clickhouse-dba` in this lab).
* `clickhouse-client` available inside the container (the official image provides it).
* You are running the commands from the host using `docker exec -it clickhouse-dba clickhouse-client` for convenience.

> NOTE: If you use a different container name, replace `clickhouse-dba` below with your container name.

---

### 1. Admin: Connect to ClickHouse shell (from host)

```bash
# Opens the ClickHouse native client inside the container as the default/admin session
docker exec -it clickhouse-dba clickhouse-client
```

You should see the ClickHouse prompt (e.g. `:)`).

---

### 2. Create the test database

```sql
CREATE DATABASE IF NOT EXISTS analytics_db;
SHOW DATABASES;
```

Confirm `analytics_db` appears in `SHOW DATABASES`.

---

### 3. Create a test table

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

### 4. Insert random sample data (using numbers() + rand())

```sql
INSERT INTO user_events
SELECT
    number + 1 AS event_id,
    rand() % 100 + 1 AS user_id,
    arrayElement(['login', 'logout', 'click', 'purchase'], toUInt32(rand() % 4) + 1) AS event_type,
    now() - INTERVAL toInt32(rand() % 10000) SECOND AS event_time
FROM numbers(20);

-- Verify rows
SELECT count() AS rows FROM user_events;
SELECT * FROM user_events LIMIT 10;
```

*Notes:* `numbers(20)` generates 20 rows; adjust as needed. `arrayElement(..., index)` is 1-based.

---

### 5. Create a new user with a secure password

```sql
CREATE USER IF NOT EXISTS report_user IDENTIFIED BY 'report_pass_!23';
SHOW USERS;
```

---

### 6. Grant only SELECT privileges on analytics_db to the new user

```sql
GRANT SELECT ON analytics_db.* TO report_user;

-- Verify grants for the user
SHOW GRANTS FOR report_user;
```

Expected `SHOW GRANTS` output (similar):

```
GRANT SELECT ON analytics_db.* TO report_user
```

---

### 7. Test: log in as report_user and run SELECT

From the host shell (not inside the ClickHouse admin session):

```bash
docker exec -it clickhouse-dba clickhouse-client -u report_user --password report_pass_!23
```

Then run:

```sql
USE analytics_db;
SELECT * FROM user_events LIMIT 5;
```

**Expected result:** The SELECT returns rows (OK).

---

### 8. Test: Attempt INSERT (should be denied)

As `report_user` (same session):

```sql
INSERT INTO user_events VALUES (999, 1, 'test', now());
```

**Expected error:**

```
DB::Exception: Not enough privileges (or similar message depending on ClickHouse version)
```

Record the exact error message in your solution.md (use copy-paste).

---

### 9. Test: Attempt ALTER (should be denied)

```sql
ALTER TABLE user_events ADD COLUMN test_col String;
```

**Expected error:** `Not enough privileges`.

Undo attempt errors by reconnecting as admin if the table is unchanged.

---

### 10. Document the observed behavior (example text for your solution.md)

**Observations**

* The `report_user` can run `SELECT` queries on the `analytics_db` database as expected.
* Attempts to run `INSERT` or `ALTER` fail with a privilege error.
* `SHOW GRANTS FOR report_user` confirms the user only has `SELECT` on `analytics_db.*`.
* ClickHouse enforces database/table-level privileges correctly for SQL users and roles.

**Sample recorded evidence**

* `SHOW GRANTS FOR report_user;` → `GRANT SELECT ON analytics_db.* TO report_user`
* `INSERT INTO ...` attempted as `report_user` → error: `DB::Exception: Not enough privileges` (copy actual message)
* `ALTER TABLE ...` attempted as `report_user` → similar error.

---

### 11. Example Git commands to add this task to your repo

```bash
mkdir -p task-01-select-only
# create README.md and solution.md locally (or copy from this file)
# then:
git add task-01-select-only/README.md task-01-select-only/solution.md
git commit -m "Task 1: create select-only user and verify permissions"
git push
```

---

### 12. Troubleshooting tips

* If `report_user` still can INSERT/ALTER, re-check `SHOW GRANTS FOR report_user` and ensure there are no global or inherited grants (roles) that give extra rights.
* If authentication fails for `report_user`, confirm the user exists in `SHOW USERS` and that the password is correct (recreate user if necessary).
* If running inside a mounted `users.xml`, remember that XML-defined users can override SQL-created users; prefer SQL `CREATE USER` for lab users unless you intend to manage them via XML.

---

### 13. Optional: Commands used during the lab (compact summary)

```sql
CREATE DATABASE IF NOT EXISTS analytics_db;
USE analytics_db;
CREATE TABLE IF NOT EXISTS user_events (... ) ENGINE = MergeTree ORDER BY event_time;
INSERT INTO user_events SELECT ... FROM numbers(20);
CREATE USER report_user IDENTIFIED BY 'report_pass_!23';
GRANT SELECT ON analytics_db.* TO report_user;
SHOW GRANTS FOR report_user;
```

---

