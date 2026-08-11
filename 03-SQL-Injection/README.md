# Project 04 — SQL Injection Testing

##  Overview

This project is part of my hands-on web application penetration testing practice.

For this assessment, I tested **OWASP Juice Shop** for SQL Injection vulnerabilities.

The main goal was to understand how SQL Injection works in a real application, identify an injection point, understand the SQL query being used by the backend, manipulate the query, identify the database, enumerate database information, and finally demonstrate the impact of the vulnerability.

I focused on understanding the reasoning behind each step instead of only relying on automated scanners or pre-built payloads.

### Target

- **Application:** OWASP Juice Shop
- **Environment:** Local lab
- **Target URL:** `http://127.0.0.1:3000`
- **Testing Type:** Black-box Web Application Security Testing
- **Vulnerability:** SQL Injection
- **Database:** SQLite

### Tools Used

- Burp Suite Community Edition
- Web Browser
- OWASP Juice Shop
- Kali Linux
- Burp Repeater


---

#  Reconnaissance

I started the assessment by running OWASP Juice Shop locally and exploring the application normally.

While browsing the application, I looked for functionality that accepted user-controlled input and communicated with the backend.

During the initial reconnaissance, I identified the product search functionality.

The relevant endpoint was:

```http
GET /rest/products/search?q=apple
```

<img width="850" height="251" alt="recon-sql" src="https://github.com/user-attachments/assets/f09bb333-c17d-49bd-b3b6-80bbb215440a" />

The q parameter immediately became interesting because it was being sent directly to the backend and was responsible for controlling the search query.

I intercepted the request using Burp Suite Repeater so I could modify the parameter and observe how the application responded.

The main things I wanted to determine were:
- Where user input was being accepted
- Which parameters influenced database queries
- How the server handled unexpected input
- Whether SQL-related errors were exposed
- Which database technology was being used

---

# Methodolgy

I used a manual black-box testing approach.

Instead of immediately using automated SQL Injection tools, I first tried to understand how the application processed the q parameter.
My testing process was:

Discover → Intercept → Modify → Observe → Understand → Exploit → Validate → Document

The SQL Injection testing was performed in several stages:
- Test normal search behavior
- Test SQL syntax handling
- Identify the SQL query context
- Perform boolean-based SQL Injection testing
- Identify the database technology
- Test UNION-based SQL Injection
- Enumerate database tables
- Enumerate the Users table schema
- Retrieve a controlled amount of user information
- Document the security impact

---

# SQL Injection Testing

**1. Normal Search Request**

I first captured a normal product search request.

Example:

```text
GET /rest/products/search?q=apple
```

The application returned the expected product search results.

This provided a baseline that I could compare against when modifying the q parameter.

<img width="1223" height="564" alt="normal-search" src="https://github.com/user-attachments/assets/1d4df1a2-2c2f-4ef6-90d5-e3b57ddff169" />

**2. SQL Syntax Testing**

I then modified the search parameter by adding a single quote.

The purpose of this test was not to exploit the application immediately.

I wanted to see whether the input was being interpreted as part of an SQL statement.

The test input was:

```text
apple'
```

The application returned an SQL-related error.

<img width="1228" height="558" alt="apple&#39;-parameter-search" src="https://github.com/user-attachments/assets/7cebfaaf-2b40-4508-ae22-dfdc2b86bf9c" />

The response exposed:
```text
SQLITE_ERROR
```

More importantly, the application exposed part of the SQL query being executed.

The query was similar to:
```http
SELECT * FROM Products
WHERE ((name LIKE '%apple%' OR description LIKE '%apple%')
AND deletedAt IS NULL)
ORDER BY name
```

This was an important finding because it showed that user-controlled input was being inserted into an SQL query.

At this point, I had strong evidence that the q parameter was vulnerable to SQL Injection.

The important part of the query was:

```text
name LIKE '%INPUT%'
```

This showed that my input was being placed inside a quoted SQL string.

For example, with:

```text
apple
```

the query became:

```text
name LIKE '%apple%'
```

When I supplied:

```text
apple'
```

the quote changed the structure of the SQL statement and caused the database to generate an error.

This helped me understand the SQL context before attempting further exploitation.

The key lesson from this stage was that SQL Injection payloads should be selected based on the actual SQL context rather than simply trying random payloads.

**3. Boolean-Based SQL Injection**

After confirming that the input could affect the SQL syntax, I tested whether I could influence the logic of the query.

The basic idea was to create a condition that would always evaluate to true.

The important SQL components were:

```text
'       → close the existing string
OR      → add another condition
1=1     → condition that is always true
--      → comment out the remaining SQL
```

payload was:

```text
' OR 1=1 --
```

<img width="1838" height="781" alt="{D1CFC9B7-AF6A-4229-9C40-516A60790A8C}" src="https://github.com/user-attachments/assets/8f005c99-1ef9-4da9-a68f-68a3c551efa8" />


The application returned a significantly different result compared with the normal search request.

This demonstrated that the SQL query logic could be influenced through user-controlled input.

**4. Database Identification**

After confirming that SQL Injection was possible, I wanted to determine which database technology was being used.

The earlier error already suggested SQLite because the application returned SQLITE ERROR

I then used a UNION-based query to retrieve the SQLite version.

The response returned:

```text
3.44.2
```

<img width="1556" height="638" alt="database-identify" src="https://github.com/user-attachments/assets/6929cb13-4e2c-49a4-b9af-eba3eeeaef65" />

This confirmed that the backend database was SQLite.

Identifying the database was important because different database technologies use different functions and system tables.

Instead of using generic SQL Injection payloads, I could now use SQLite-specific functionality.

**5. UNION-Based SQL Injection**

Once I understood the SQL context and database technology, I tested whether I could use a UNION SELECT statement.

The purpose of a UNION-based SQL Injection is to append another query to the application's original query and make the application return information from that query.

Before constructing the UNION query, I needed to understand the number of columns returned by the original query.

The product query used:
```text
SELECT * FROM Products
```

and the product response contained nine fields.

Therefore, the injected SELECT also needed to return nine columns.

I used NULL values for fields that were not important to the test.

After adjusting the query structure and encoding the request correctly, the UNION injection succeeded.

**6. Database Table Enumeration**

After confirming UNION-based SQL Injection, I wanted to understand what information existed in the database.

Because the application was using SQLite, I queried the SQLite metadata table sqplite_master

The response revealed tables including:
```text
Users
Addresses
Baskets
Products
BasketItems
Captchas
Cards
Challenges
Complaints
Deliveries
Feedbacks
Hints
ImageCaptchas
Memories
PrivacyRequests
Quantities
Recycles
SecurityQuestions
SecurityAnswers
Wallets
```

<img width="1543" height="640" alt="tables" src="https://github.com/user-attachments/assets/dc1807c8-cbe9-4590-97b2-ff65d45885f2" />

**7. Users Table Enumeration**

The Users table was particularly interesting because it contained authentication and account-related information.

Before retrieving any records, I first enumerated the table structure.

The discovered columns included:
```text
id
username
email
password
role
deluxeToken
lastLoginIp
profileImage
totpSecret
isActive
createdAt
updatedAt
deletedAt
```

<img width="1832" height="758" alt="user-table" src="https://github.com/user-attachments/assets/b6ac73c0-2424-4acb-9ab2-1c22936cbdb4" />

**8. User Data Extraction**

After identifying the structure of the Users table, I performed a controlled extraction of non-sensitive fields.

I retrieved:

```text
username
email
role
```
The extracted data included different types of application accounts and roles.

<img width="1544" height="652" alt="extracted-data" src="https://github.com/user-attachments/assets/f437fd4f-802a-40dc-a67b-ff933b6bf3af" />

This demonstrated the real security impact of the SQL Injection.

---

# Findings

SQLI-01 — SQL Injection

Severity: High

Status: Confirmed

Affected Endpoint:
```text
GET /rest/products/search?q=
```

Affected Parameter: q

**Description**

The product search endpoint was vulnerable to SQL Injection because user-controlled input was incorporated into the backend SQL query.

The application did not safely separate user input from SQL syntax.

I was able to:
- Trigger SQLite SQL errors
- Identify the SQL query structure
- Manipulate the query logic
- Identify the SQLite database
- Execute UNION-based SQL
- Enumerate database tables
- Enumerate the Users table schema
- Retrieve records from the Users table

---

# Impact

The vulnerability could allow an attacker to access database information outside the intended functionality of the application.

Depending on database permissions and the queries available to the attacker, SQL Injection can potentially result in:
- Unauthorized database access
- Sensitive information disclosure
- Database structure disclosure
- Account information disclosure
- Authentication data exposure
- Data modification
- In some environments, further compromise of the application or underlying system
- In this assessment, I successfully demonstrated database enumeration and extraction of user-re

---

# Remidation

- Use parameterized queries throughout the application
- Avoid dynamic SQL construction with user input
- Apply appropriate input validation
- Use least-privilege database accounts
- Avoid exposing database errors to users
- Implement centralized error handling
- Monitor suspicious SQL Injection patterns
- Perform security testing during development
- Retest the endpoint after remediation

---

