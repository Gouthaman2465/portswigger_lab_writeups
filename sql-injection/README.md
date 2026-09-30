# SQL Injection

PortSwigger Web Security Academy — SQL injection labs.

## 1. SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
[Lab link](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)

**Vulnerability:** User-controlled input is concatenated directly into a `WHERE` clause.
**Goal:** Display products outside the normal category filter, including unlisted ones.
**Steps:**
- Observed the backend query: `SELECT * FROM products WHERE category = 'Gifts' AND released = 1`.
- Intercepted the category-filter request and changed the value to `'+OR+1=1--`.
- The resulting query became `SELECT * FROM products WHERE category = '' OR 1=1-- AND released = 1`; the `--` comments out the rest, so the `OR 1=1` returns every row.

**Why it works:** Unsanitized input let the attacker close the intended string literal and inject their own logic into the query.
**Fix:** Use parameterized queries / prepared statements everywhere user input reaches SQL; never build queries by string concatenation.

---

## 2. SQL injection vulnerability allowing login bypass
[Lab link](https://portswigger.net/web-security/sql-injection/lab-login-bypass)

**Vulnerability:** Login query concatenates the username directly into a `WHERE` clause.
**Goal:** Log in as `administrator` without knowing the password.
**Steps:**
- Submitted a username of `administrator'--` with any password.
- The `--` commented out the password check portion of the query, so the login succeeded purely on username.

**Why it works:** Commenting out the rest of the query removes the password comparison entirely, so the query effectively becomes "does a user named administrator exist."
**Fix:** Parameterize the login query and never concatenate user-supplied credentials into SQL text.

---

## 3. SQL injection attack, querying the database type and version on Oracle
[Lab link](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle)

**Vulnerability:** UNION-based injection point in a query with a known column count.
**Goal:** Determine the Oracle database version.
**Steps:**
- Used the SQLi cheat sheet's Oracle version query and injected: `'+UNION+SELECT+BANNER,NULL+FROM+v$version--`.
- The response displayed the Oracle version string.

**Why it works:** `UNION SELECT` lets an attacker append arbitrary result rows from any accessible table (here, Oracle's `v$version`) into the application's own result set, as long as the column count and rough types line up (`NULL` pads out unused columns).
**Fix:** Parameterized queries prevent this entirely; least-privilege database accounts also limit what a successful injection can read.

---

## 4. SQL injection attack, querying the database type and version on MySQL and Microsoft
[Lab link](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)

**Vulnerability:** Same UNION-based injection point as lab 3, different database engine.
**Goal:** Determine the MySQL/MSSQL version.
**Steps:**
- Injected `'+UNION+SELECT+@@version,NULL--+`, including a trailing space required by MySQL after `--` to treat it as a comment.
- The response revealed the version string.

**Why it works:** Same root cause as lab 3 — unsanitized concatenation into SQL — with a database-specific version function (`@@version`).
**Fix:** Same as above: parameterized queries.

---

## 5. SQL injection attack, listing the database contents on non-Oracle databases
[Lab link](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)

**Vulnerability:** UNION-based injection, chained across multiple steps to enumerate schema and dump data.
**Goal:** Recover admin credentials from an unknown table.
**Steps:**
- Determined the query returns 2 columns: `'+UNION+SELECT+NULL,NULL--`.
- Confirmed both columns accept text: `'+UNION+SELECT+'abcf','sefg'--`.
- Enumerated table names via `information_schema.tables`: `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`, and identified a users-like table.
- Enumerated that table's columns via `information_schema.columns`.
- Extracted the username/password columns directly: `'+UNION+SELECT+username_col,+password_col+FROM+<table>--`, revealing the admin password, then logged in.

**Why it works:** Once a UNION injection point is confirmed, the database's own information schema can be queried the same way as application tables, letting an attacker map out and then dump arbitrary data.
**Fix:** Parameterized queries close this off entirely; additionally, don't grant the application's DB user read access to `information_schema` or other accounts' tables if avoidable.

---

## 6. SQL injection attack, listing the database contents on Oracle
[Lab link](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-oracle)

**Vulnerability:** Same as lab 5, targeting Oracle's schema views instead.
**Goal:** Recover admin credentials from an unknown table.
**Steps:**
- Confirmed column count/types using `FROM dual` (Oracle requires a table reference even for literal selects).
- Enumerated tables via `all_tables`, then columns via `all_tab_columns` for the identified table.
- Extracted the username and password columns directly, revealing valid admin credentials.

**Why it works:** Same as lab 5 — the injection point allows arbitrary reads via the database's own catalog views.
**Fix:** Parameterized queries; least-privilege DB accounts.

---

## 7. SQL injection UNION attack, determining the number of columns returned by the query
[Lab link](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns)

**Vulnerability:** UNION-based injection point; column count unknown at first.
**Goal:** Determine how many columns the original query returns.
**Steps:**
- Injected `'UNION+SELECT+NULL--` and increased the number of `NULL`s until the error disappeared.
- Three `NULL`s (`'UNION+SELECT+NULL,NULL,NULL--`) returned successfully, confirming a 3-column query.

**Why it works:** A `UNION SELECT` must match the original query's column count exactly, or the database raises an error — incrementing `NULL` values is a reliable way to discover that count via trial and error.
**Fix:** Parameterized queries prevent injection entirely, making this reconnaissance step moot.

---

## 8. SQL injection UNION attack, finding a column containing text
[Lab link](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text)

**Vulnerability:** Same injection point as lab 7, extended to find which columns accept string data.
**Goal:** Identify which column(s) in the UNION can hold a string value (needed to later extract text data).
**Steps:**
- With the column count known, substituted a string literal for each `NULL` in turn: `'UNION+SELECT+'abcdfefe',NULL,NULL--`.
- The column that returned the literal without error is a text-compatible column.

**Why it works:** Databases only accept a UNION if column types are compatible; testing each position individually reveals which ones can carry string output for later data extraction.
**Fix:** Parameterized queries.

---

## 9. SQL injection UNION attack, retrieving data from other tables
[Lab link](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables)

**Vulnerability:** UNION injection with a known target table name.
**Goal:** Retrieve all usernames and passwords from the `users` table.
**Steps:**
- Confirmed column count and text-compatible columns as in labs 7–8.
- Ran `'+UNION+SELECT+username,password+FROM+users--`, returning every row's credentials in the page output.
- Logged in with the recovered admin credentials.

**Why it works:** Once the injection point, column count, and compatible types are known, any accessible table — including one the application never intended to expose — can be read directly.
**Fix:** Parameterized queries; least-privilege DB accounts.

---

## 10. SQL injection UNION attack, retrieving multiple values in a single column
[Lab link](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)

**Vulnerability:** Same UNION injection point, but only one text-compatible output column is available.
**Goal:** Retrieve both username and password despite having only one usable output column.
**Steps:**
- Confirmed 2 columns total but only one accepts text.
- Concatenated both values into that single column: `'+UNION+SELECT+NULL,username||password+FROM+users--` (`||` is the concatenation operator).
- Parsed the combined string in the response to recover both values.

**Why it works:** String concatenation lets an attacker pack multiple pieces of data into a single output column when the query's shape otherwise limits how many values can be returned.
**Fix:** Parameterized queries.
