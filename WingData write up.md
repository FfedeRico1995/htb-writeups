Let's start with enumeration, try to find open port and version.
❯ sudo nmap 10.129.20.226 -p- -sV -sC
[sudo] password for kali: 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-04 11:16 -0400
Nmap scan report for 10.129.20.226
Host is up (0.028s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey: 
|   256 a1:fa:95:8b:d7:56:03:85:e4:45:c9:c7:1e:ba:28:3b (ECDSA)
|_  256 9c:ba:21:1a:97:2f:3a:64:73:c1:4c:1d:ce:65:7a:2f (ED25519)
80/tcp open  http    Apache httpd 2.4.66
|_http-title: Did not follow redirect to http://wingdata.htb/
|_http-server-header: Apache/2.4.66 (Debian)
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Two port found who redirect, add to /etc/hosts with
echo "10.129.20.226 wingdata.htb" | sudo tee -a /etc/hosts
and then brute force the site with 
ffuf -u http://wingdata.htb -H "Host: FUZZ.wingdata.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -ac
![[Pasted image 20260404173029.png]]
![[Pasted image 20260404173214.png]]![[Pasted image 20260404173420.png]]
Found vulnerability CVE-2025-47812: Description

In Wing FTP Server before 7.4.4. the user and admin web interfaces mishandle '\0' bytes, ultimately allowing injection of arbitrary Lua code into user session files. This can be used to execute arbitrary system commands with the privileges of the FTP service (root or SYSTEM by default). This is thus a remote code execution vulnerability that guarantees a total server compromise. This is also exploitable via anonymous FTP accounts.

PoC ready on githyb with python3 CVE-2025-47812.py -u http://ftp.wingdata.htb -c whoami -v

![[Pasted image 20260404173933.png]]