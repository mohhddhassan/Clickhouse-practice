Task 1: Create a New User with Limited Privileges (SELECT Only)
1. Log in as an administrator user on the ClickHouse server.

2. Identify or create a test database (e.g., analytics_db) for this exercise.

3. Create a test table with a few sample records to work with. Use generateRandom() to insert random records

4. Create a new user with a secure password.

5. Grant this user only SELECT privileges on the chosen database.

6. Test that the user can successfully run a SELECT query.

7. Attempt non-SELECT operations such as INSERT or ALTER to confirm that access is denied.

8. Document the behavior and confirm that permission boundaries are enforced correctly.
