# HTB Writeups

Professional writeups for [Hack The Box](https://www.hackthebox.com/) machines, written as part of the **CPTS (Certified Penetration Testing Specialist)** certification path.

Each report covers the full attack chain: reconnaissance, enumeration, exploitation, privilege escalation, and lessons learned. Written in English, structured for readability and methodological clarity.

---

## 📁 Structure

```
htb-writeups/
├── Tier 0/
│   └── Meow/         Fawn/         Dancing/         Redeemer/         Explosion/
├── Tier 1/
│   └── Appointment/  Crocodile/    Sequel/           Responder/        Three/
├── Tier 2/
│   └── Archetype/    Oopsie/       Vaccine/
└── Easy/
    └── Cap/
```

---

## ✅ Completed Machines

### Starting Point — Tier 0
| Machine | Difficulty | Topics |
|---------|-----------|--------|
| Meow | ☁️ Very Easy | Telnet, default credentials |
| Fawn | ☁️ Very Easy | FTP, anonymous login |
| Dancing | ☁️ Very Easy | SMB, anonymous shares |
| Redeemer | ☁️ Very Easy | Redis, unauthenticated access |
| Explosion | ☁️ Very Easy | RDP, default credentials |

### Starting Point — Tier 1
| Machine | Difficulty | Topics |
|---------|-----------|--------|
| Appointment | ☁️ Very Easy | SQL Injection, web fundamentals |
| Crocodile | ☁️ Very Easy | FTP, web enumeration, credential reuse |
| Sequel | ☁️ Very Easy | MySQL, unauthenticated access |
| Responder | ☁️ Very Easy | LLMNR poisoning, WinRM, hash cracking |
| Three | ☁️ Very Easy | AWS S3, subdomain enumeration, web shells |

### Starting Point — Tier 2
| Machine | Difficulty | Topics |
|---------|-----------|--------|
| Archetype | 🟢 Easy | SMB, MSSQL, PowerShell, privilege escalation |
| Oopsie | 🟢 Easy | IDOR, web shells, SUID privilege escalation |
| Vaccine | 🟢 Easy | FTP, hash cracking, SQLi, sudo misconfiguration |

### Easy Machines
| Machine | Difficulty | Topics |
|---------|-----------|--------|
| Cap | 🟢 Easy | PCAP analysis, IDOR, Linux capabilities |

---

## 🗺️ Roadmap

This repo follows the **TJNull CPTS preparation list** — a curated set of retired HTB machines that map to the skills tested in the CPTS exam.

- [x] Starting Point Tier 0
- [x] Starting Point Tier 1
- [x] Starting Point Tier 2
- [ ] Easy machines (in progress)
- [ ] Medium machines
- [ ] Hard machines

---

## 🛠️ Methodology

Each writeup follows a structured format:

1. **Overview** — machine summary and key takeaways
2. **Reconnaissance** — port scanning, service enumeration
3. **Foothold** — initial access vector and exploitation
4. **Privilege Escalation** — local enumeration and escalation path
5. **Lessons Learned** — defensive perspective and key concepts

Tools used: `nmap`, `gobuster`, `ffuf`, `sqlmap`, `Metasploit`, `netcat`, `Impacket`, `CrackMapExec`, `Responder`, and others.

---

## 📜 Certification Target

**HTB CPTS** — Hack The Box Certified Penetration Testing Specialist

> Writeups in this repo are for **retired machines only**. Active and seasonal machine solutions are kept private in compliance with HTB rules.
