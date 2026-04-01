
Crocodile is a very easy Linux machine which showcases the dangers of misconfigured authentication and sensitive data exposure. A vulnerable FTP server instance is misconfigured to allow anonymous authentication and upon enumerating the server, sensitive files can be found containing cleartext credentials. Enumerating and fuzzing the website will reveal a hidden login endpoint where the previously acquired credentials can be used to gain access to the admin panel.

 sudo nmap 10.129.1.15 -p- -sV -sC                                                     
[sudo] password for kali:                                                               
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-31 15:53 -0400                       
Nmap scan report for 10.129.1.15                                                        
Host is up (0.056s latency).                                                            
Not shown: 65533 closed tcp ports (reset)                                               
PORT   STATE SERVICE VERSION                                                            
21/tcp open  ftp     vsftpd 3.0.3                                                       
| ftp-syst:                                                                             
|   STAT:                                                                                                                                                                        
| FTP server status:                                                                                                                                                             
|      Connected to ::ffff:10.10.15.96                                                                                                                                           
|      Logged in as ftp                                                                                                                                                          
|      TYPE: ASCII                                                                                                                                                               
|      No session bandwidth limit                                                                                                                                                
|      Session timeout in seconds is 300                                                                                                                                         
|      Control connection is plain text                                                                                                                                          
|      Data connections will be plain text                                                                                                                                       
|      At session startup, client count was 2                                                                                                                                    
|      vsFTPd 3.0.3 - secure, fast, stable                                                                                                                                       
|_End of status                                                                                                                                                                  
| ftp-anon: Anonymous FTP login allowed (FTP code 230)                                                                                                                           
| -rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist                                                                                                       
|_-rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd                                                                                                
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))                                                                                                                              
|_http-server-header: Apache/2.4.41 (Ubuntu)                                                                                                                                     
|_http-title: Smash - Bootstrap Business Template                                                                                                                                
Service Info: OS: Unix                          
Possiamo notare che la porta 21 ftp e aperta, e consente il login con anonymous, senza password. 
Vediamo aperta anche l porta 80, che gira su ubuntu/2.4.41

Per connetterci usiamo ftp 10.129.1.15, ci chiedera username, mettiamo anonymous, non verra ricbiesta password, siamo dentro.

Ora con ls vediamo i file, e ttrooviamo
ftp> ls                                                                                                                                                                          
229 Entering Extended Passive Mode (|||41249|)                                                                                                                                   
150 Here comes the directory listing.                                                                                                                                            
-rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist                                                                                                         
-rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd   

scarichiamo entrambi con get, e in altra cli leggiamo il contenuto

Leggendo il contenuto abbiamo trovato in allower userlist i name, e in  allowed.userlist.passwd le password associate.
Ora lanciamo gobuster per trovare informazioni
 gobuster dir -u http://10.129.1.15/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.1.15/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.htpasswd            (Status: 403) [Size: 276]
.htaccess            (Status: 403) [Size: 276]
.hta                 (Status: 403) [Size: 276]
assets               (Status: 301) [Size: 311] [--> http://10.129.1.15/assets/]
css                  (Status: 301) [Size: 308] [--> http://10.129.1.15/css/]
dashboard            (Status: 301) [Size: 314] [--> http://10.129.1.15/dashboard/]
fonts                (Status: 301) [Size: 310] [--> http://10.129.1.15/fonts/]
index.html           (Status: 200) [Size: 58565]
js                   (Status: 301) [Size: 307] [--> http://10.129.1.15/js/]
server-status        (Status: 403) [Size: 276]
Progress: 4750 / 4750 (100.00%)
===============================================================
Finished
===============================================================
Vediamo che possiamo essere reindirizzati in dashboard, andiamo e vediamo un login, tentiamo con le credenziali trovate admin e la password associata e siamo dentro.
Qua troviamo la flag.

task 1
What Nmap scanning switch employs the use of default scripts during a scan?
-sC
Task 2
What service version is found to be running on port 21?
vsftpd 3.0.3
Task 3
What FTP code is returned to us for the "Anonymous FTP login allowed" message?
230

Task 4

After connecting to the FTP server using the ftp client, what username do we provide when prompted to log in anonymously?
