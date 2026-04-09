sudo nmap -sV -sC -p- -Pn -n -A -oA facts_scan 10.129.24.110                                                                                                                   
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-09 03:46 -0400                                                                                                                
Nmap scan report for 10.129.24.110                                                                                                                                               
Host is up (0.060s latency).                                                                                                                                                     
Not shown: 65532 closed tcp ports (reset)                                                                                                                                        
PORT      STATE SERVICE VERSION                                                                                                                                                  
22/tcp    open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)                                                                                             
| ssh-hostkey:                                                                                                                                                                   
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)                                                                                                                  
|_  256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)                                                                                                                
80/tcp    open  http    nginx 1.26.3 (Ubuntu)                                                                                                                                    
|_http-title: Did not follow redirect to http://facts.htb/                                                                                                                       
|_http-server-header: nginx/1.26.3 (Ubuntu)                                                                                                                                      
54321/tcp open  http    Golang net/http server                                                                                                                                   
|_http-server-header: MinIO                                                                                                                                                      
|_http-title: Site doesn't have a title (application/xml).                                                                                                                       
| fingerprint-strings:                                                                                                                                                           
|   FourOhFourRequest:                                                                                                                                                           
|     HTTP/1.0 400 Bad Request                                                                                                                                                   
|     Accept-Ranges: bytes                                                                                                                                                       
|     Content-Length: 303                                                                                                                                                        
|     Content-Type: application/xml                                                                                                                                              
|     Server: MinIO                                                                                                                                                              
|     Strict-Transport-Security: max-age=31536000; includeSubDomains                                                                                                             
|     Vary: Origin                                                                                                                                                               
|     X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8                                                                                               
|     X-Amz-Request-Id: 18A4A10171F4BDC2                                                                                                                                         
|     X-Content-Type-Options: nosniff                                                                                                                                            
|     X-Xss-Protection: 1; mode=block                                                                                                                                            
|     Date: Thu, 09 Apr 2026 07:53:00 GMT                                                                                                                                        
|     <?xml version="1.0" encoding="UTF-8"?>                                                                                                                                     
|     <Error><Code>InvalidRequest</Code><Message>Invalid Request (invalid argument)</Message><Resource>/nice ports,/Trinity.txt.bak</Resource><RequestId>18A4A10171F4BDC2</Reques
tId><HostId>dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8</HostId></Error>                                                                                    
|   GenericLines, Help, RTSPRequest, SSLSessionReq:                                                                                                                              
|     HTTP/1.1 400 Bad Request                                                                                                                                                   
|     Content-Type: text/plain; charset=utf-8                                                                                                                                    
|     Connection: close                                                                                                                                                          
|     Request                                                                                                                                                                    
|   GetRequest:                                                                                                                                                                  
|     HTTP/1.0 400 Bad Request
|     Accept-Ranges: bytes
|     Content-Length: 276
|     Content-Type: application/xml
|     Server: MinIO
|     Strict-Transport-Security: max-age=31536000; includeSubDomains
|     Vary: Origin
|     X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
|     X-Amz-Request-Id: 18A4A0FDB0320543
|     X-Content-Type-Options: nosniff
|     X-Xss-Protection: 1; mode=block
|     Date: Thu, 09 Apr 2026 07:52:44 GMT
|     <?xml version="1.0" encoding="UTF-8"?>
|     <Error><Code>InvalidRequest</Code><Message>Invalid Request (invalid argument)</Message><Resource>/</Resource><RequestId>18A4A0FDB0320543</RequestId><HostId>dd9025bab4ad464
b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8</HostId></Error>
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Vary: Origin
|     Date: Thu, 09 Apr 2026 07:52:44 GMT
|_    Content-Length: 0
Device type: general purpose|router
Running: Linux 5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 5.0 - 5.14, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 23/tcp)
HOP RTT      ADDRESS
1   59.38 ms 10.10.14.1
2   59.27 ms 10.129.24.110

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 421.45 seconds

ffuf -u http://facts.htb/FUZZ \                                                                                                                                                  
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt \                                                                                                            
  -fs 11110,11119,11125,11128,11134,11122,11131,11113,11116,11140,11137                                                                                                          
                                                                                                                                                                                 
        /'___\  /'___\           /'___\                                                                                                                                          
       /\ \__/ /\ \__/  __  __  /\ \__/                                                                                                                                          
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\                                                                                                                                         
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/                                                                                                                                         
         \ \_\   \ \_\  \ \____/  \ \_\                                                                                                                                          
          \/_/    \/_/   \/___/    \/_/                                                                                                                                          
                                                                                                                                                                                 
       v2.1.0-dev                                                                                                                                                                
________________________________________________                                                                                                                                 
                                                                                                                                                                                 
 :: Method           : GET                                                                                                                                                       
 :: URL              : http://facts.htb/FUZZ                                                                                                                                     
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt                                                                                      
 :: Follow redirects : false                                                                                                                                                     
 :: Calibration      : false                                                                                                                                                     
 :: Timeout          : 10                                                                                                                                                        
 :: Threads          : 40                                                                                                                                                        
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500                                                                                                      
 :: Filter           : Response size: 11110,11119,11125,11128,11134,11122,11131,11113,11116,11140,11137                                                                          
________________________________________________                                                                                                                                 
                                                                                                                                                                                 
400                     [Status: 200, Size: 6685, Words: 993, Lines: 115, Duration: 1372ms]                                                                                      
404                     [Status: 200, Size: 4836, Words: 832, Lines: 115, Duration: 1325ms]                                                                                      
500                     [Status: 200, Size: 7918, Words: 1035, Lines: 115, Duration: 1304ms]                                                                                     
admin                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1366ms]                                                                                             
admin.cgi               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1371ms]                                                                                             
admin.php               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1293ms]                                                                                             
admin.pl                [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 1285ms]                                                                                             
ajax                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 1663ms]                                                                                             
captcha                 [Status: 200, Size: 5419, Words: 26, Lines: 17, Duration: 1924ms]                                                                                        
en                      [Status: 200, Size: 11109, Words: 1328, Lines: 125, Duration: 2280ms]                                                                                    
error                   [Status: 500, Size: 7918, Words: 1035, Lines: 115, Duration: 1820ms]                                                                                     
page                    [Status: 200, Size: 19593, Words: 3296, Lines: 282, Duration: 1882ms]                                                                                    
post                    [Status: 200, Size: 11308, Words: 1414, Lines: 152, Duration: 1661ms]                                                                                    
robots.txt              [Status: 200, Size: 99, Words: 12, Lines: 2, Duration: 1784ms]                                                                                           
robots                  [Status: 200, Size: 33, Words: 2, Lines: 1, Duration: 1881ms]                                                                                            
rss                     [Status: 200, Size: 183, Words: 20, Lines: 9, Duration: 2042ms]                                                                                          
search                  [Status: 200, Size: 19187, Words: 3276, Lines: 272, Duration: 1930ms]                                                                                    
sitemap                 [Status: 200, Size: 3508, Words: 424, Lines: 130, Duration: 1808ms]                                                                                      
sitemap.gz              [Status: 500, Size: 7918, Words: 1035, Lines: 115, Duration: 1954ms]                                                                                     
sitemap.xml             [Status: 200, Size: 3508, Words: 424, Lines: 130, Duration: 2090ms]                                                                                      
up                      [Status: 200, Size: 73, Words: 4, Lines: 1, Duration: 2047ms]                                                                                            
welcome                 [Status: 200, Size: 11966, Words: 1481, Lines: 130, Duration: 1916ms]                                                                                    
:: Progress: [4750/4750] :: Job [1/1] :: 21 req/sec :: Duration: [0:03:42] :: Errors: 0 ::                                                                                       
                                                                                                       

 echo "10.129.244.96   facts.htb" | sudo tee -a /etc/hosts
10.129.244.96   facts.htb
![[Pasted image 20260408214812.png]]
Troviamo che possiamo andare in http://facts.htb/admin/ e provar a registrarci, lo facciamo, accediamo e siamo in http://facts.htb/admin/dashboard come admin.
![[Pasted image 20260408220429.png]]https://www.offsec.com/blog/cve-2024-46986/
![[Pasted image 20260408221802.png]]
da qua posso fare upload e sfruttare exploit
Con f12 dalla pagina admin copiato in storage, cookies, http://facts.htb il 
_factsapp_session: h7iGP3KgweBt%2BlgKhXHN6cwvS8SLOO41AnBkqq1vYg%2BafD4%2BfC%2FISoqkjNpbrGXREBf6r6lSgZe2Lc%2Fzuh2RQ1GwyC2F%2FFkk3fOHBuVrabaC3fbpQWXSv2Sff1btBP4xC9zO6T8RXfJSHMr7XtezqhnaLDuaHJhfgj0VLQd9pZZaCApjh9uGUNuslsBWGlYamOUEEtmh3HIO7upBIRMezc1mEC3gFIY%2BtvbPwSEPd8wFV45yMfyY1Su5PBpj0qrYtX1Ole5%2BhYUNvAiZKxQSbI0CKU4AAn9ODws4K3zHkoJwXPZv919BM7fDGyeSNfS50Ge3nptw0b6ooZzi8FvGI0oTOabCGHqlE7yvr1mO5%2FksQgKK%2FYuCVKw%3D--zrak81EH8bGAgdIO--3z5wJd6mXWnu33Xgvi%2B0Cw%3D%3D
questo o quello di auth token?
e auth_token: YBx9Nli-bGDFuJAYxJZdcw&Mozilla%2F5.0+%28X11%3B+Linux+x86_64%3B+rv%3A140.0%29+Gecko%2F20100101+Firefox%2F140.0&10.10.15.60

versione patchata, quindi nulla
