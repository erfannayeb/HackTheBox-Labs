# Fawn

**Platform:** HackTheBox — Starting Point  
**Category:** Network Fundamentals  
**OS:** Linux  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed an FTP service misconfigured to allow anonymous 
login. After identifying the open port with Nmap, connecting via FTP 
as anonymous gave direct access to the filesystem where the flag was 
stored.

---

## Recon

```bash
nmap -sV -sC 10.129.X.X -oN nmap.txt
```

Nmap revealed port 21 open running FTP with anonymous login allowed. 
The default Nmap scripts automatically detected and reported the 
anonymous FTP misconfiguration.

21/tcp open  ftp  vsftpd 3.0.3
ftp-anon: Anonymous FTP login allowed (FTP code 230)
-rw-r--r-- 1 0 0 32 Jun 04 2021 flag.txt

Two important findings from the Nmap output — anonymous login is 
allowed, and a file called flag.txt is already visible in the 
directory listing.

---

## Enumeration
The Nmap output already revealed the flag file. Anonymous FTP 
requires no credentials — username is `anonymous` and password 
is either empty or any email address. No further enumeration 
was needed before attempting connection.

---

## Exploitation
Connected via FTP using anonymous credentials and downloaded 
the flag file directly.

```bash
ftp 10.129.X.X

# Name: anonymous
# Password: (press Enter or type any email)
# 230 Login successful.

ls -la
# -rw-r--r-- 1 0 0 32 Jun 04 2021 flag.txt

get flag.txt
# 226 Transfer complete.

quit
```

Then read the downloaded file locally:

```bash
cat flag.txt
```

---

## Post-Exploitation
Full access to the FTP server was achieved via anonymous login. 
In a real engagement the next steps would be checking write 
permissions to upload a webshell, looking for configuration files 
containing credentials, and checking if the FTP root directory 
overlaps with a web server root.

```bash
# Check write permissions
put test.txt

# List all files including hidden
ls -la
```

---

## Flag

---

## Key Lesson
This machine demonstrates anonymous FTP — one of the most common 
misconfigurations found during real penetration tests and bug bounty 
hunting. FTP was designed in an era before security was a priority 
and transmits everything including credentials in cleartext.

Anonymous FTP login means the server accepts any username of 
`anonymous` with no real password. This is sometimes intentionally 
configured for public file distribution but is extremely dangerous 
when left enabled on servers containing sensitive files.

In a real penetration test, Nmap's `ftp-anon` script automatically 
flags this misconfiguration. The immediate action is to connect, 
list all files, download everything, and check for write access. 
Write access to an FTP directory that overlaps with a web server 
root allows uploading a webshell and achieving remote code execution.

The fix is to disable anonymous FTP entirely if not needed, or 
restrict anonymous access to a dedicated read-only public directory 
with no sensitive files.
