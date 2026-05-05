# Crocodile

**Platform:** HackTheBox — Starting Point  
**Category:** Network Fundamentals  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed both an FTP service with anonymous login 
enabled and a web server. Anonymous FTP access revealed two files 
containing usernames and passwords. These credentials were reused 
on the web application's admin login page discovered via directory 
enumeration, granting administrator access and the flag.

---

## Recon

```bash
nmap -sV -sC 10.129.X.X -oN nmap.txt
```

Nmap revealed two open ports — FTP on port 21 with anonymous 
login allowed, and a web server on port 80.
```
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r-- allowed.userlist
| -rw-r--r-- allowed.userlist.passwd
80/tcp open  http    Apache httpd 2.4.41
```
Two files immediately visible in the FTP directory listing — 
`allowed.userlist` and `allowed.userlist.passwd`. These names 
strongly suggest credential files.

---

## Enumeration

**Step 1 — Download credential files from FTP**

Connected anonymously and downloaded both files:

```bash
ftp 10.129.X.X

# Name: anonymous
# Password: (press Enter)

ls -la
# allowed.userlist
# allowed.userlist.passwd

get allowed.userlist
get allowed.userlist.passwd
quit
```

Read the contents of both files:

```bash
cat allowed.userlist
# aron
# pwnmeow
# egotisticalsw
# admin

cat allowed.userlist.passwd
# root
# Supersecretpassword1
# @BaASD&9032123sADS
# rKXM59ESxesUFHAd
```

Four usernames and four corresponding passwords in order.

**Step 2 — Enumerate the web application**

Browsed to the web server on port 80. The main page did not 
reveal an obvious login form. Ran Gobuster to find hidden pages:

```bash
gobuster dir -u http://10.129.X.X -w /usr/share/wordlists/dirb/common.txt -x php,html -t 50
```

Gobuster found an admin login page:
```
/login.php   (Status: 200)
/dashboard   (Status: 302)
```
---

## Exploitation
Browsed to the login page at `http://10.129.X.X/login.php` and 
tried each username and password combination from the files 
downloaded via FTP. The credentials matched positionally — 
first username with first password, second with second, and 
so on.

The combination `admin` / `rKXM59ESxesUFHAd` granted access 
to the admin dashboard.
The flag was displayed directly on the admin dashboard after 
successful login.

---

## Post-Exploitation
In a real penetration test, gaining access to an admin dashboard 
opens multiple avenues for further exploitation — looking for 
file upload functionality to upload a webshell, checking for 
command injection in admin features, reading application source 
code or configuration files, and pivoting to other services 
using any additional credentials found.

```bash
# Continue directory enumeration inside the admin panel
gobuster dir -u http://10.129.X.X/dashboard/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak -t 50 \
  -c "session=YOURCOOKIE"
```

---

## Key Lesson
This machine demonstrates credential reuse — one of the most 
common and impactful vulnerabilities found during real penetration 
tests. The attack chain required chaining two separate 
misconfigurations together: anonymous FTP access exposing 
credential files, and those same credentials being valid on 
the web application.

Neither vulnerability alone would have been sufficient — the FTP 
files needed a login page to use them on, and the login page 
needed credentials to be useful. This is a fundamental concept 
in penetration testing called attack chaining — combining multiple 
low-severity findings into a high-impact exploit path.

The lesson about credential reuse extends far beyond this machine. 
In real environments, credentials found in one place — config 
files, FTP servers, Git repositories, backup files — are almost 
always worth trying on every other service. Password reuse is 
extremely common even among technical users.

Directory enumeration with Gobuster was also essential here. 
The admin login page was not linked from anywhere on the main 
site — without running Gobuster it would never have been found. 
This reinforces the importance of always running directory 
enumeration on any web server encountered during a pentest, 
regardless of how simple the main page appears.

The fix involves disabling anonymous FTP access, never storing 
credentials in files accessible without authentication, using 
unique passwords for every service, and restricting admin panels 
to internal networks or VPN access only.
