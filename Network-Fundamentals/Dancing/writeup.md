# Dancing

**Platform:** HackTheBox — Starting Point  
**Category:** Network Fundamentals  
**OS:** Windows  
**Difficulty:** Very Easy  
**Date:** 2025  

---

## Summary
This machine exposed an SMB service misconfigured to allow null 
session access. After identifying open ports with Nmap, connecting 
to SMB shares without credentials revealed a share containing 
the flag.

---

## Recon

```bash
nmap -sV -sC 10.129.X.X -oN nmap.txt
```

Nmap revealed multiple ports open typical of a Windows machine. 
The most interesting findings were ports 135, 139, and 445 — 
all associated with SMB and Windows networking services.
```bash
135/tcp open  msrpc

139/tcp open  netbios-ssn

445/tcp open  microsoft-ds
```
Port 445 is the modern SMB port. This is the primary target for 
enumeration.

---

## Enumeration
Listed all available SMB shares without providing any credentials — 
known as a null session. This is a common misconfiguration where 
the server allows unauthenticated access to browse available shares.

```bash
smbclient -L 10.129.X.X -N
```

Output showed several shares:

ADMIN$       Disk    Remote Admin

C$           Disk    Default share

IPC$         IPC     Remote IPC

WorkShares   Disk

`ADMIN$` and `C$` are default Windows administrative shares that 
normally require credentials. `WorkShares` is a custom share that 
looked interesting — no comment and no obvious access restriction.

---

## Exploitation
Attempted to connect to each share without credentials. `ADMIN$` 
and `C$` denied access as expected. `WorkShares` allowed connection 
with no password.

```bash
smbclient \\\\10.129.X.X\\WorkShares -N
```

Once connected, browsed the share contents:

```bash
smb: \> ls

  Amy.J          D    0  Mon Mar 29 09:08:24 2021
  James.P        D    0  Thu Jun  3 09:38:03 2021

smb: \> cd James.P
smb: \James.P\> ls

  flag.txt       A   32  Mon Mar 29 09:26:57 2021

smb: \James.P\> get flag.txt
# getting file \James.P\flag.txt
```

---

## Post-Exploitation
The share contained two user directories — Amy.J and James.P. 
In a real penetration test both directories would be fully explored 
for sensitive files — documents, configuration files, scripts 
containing hardcoded credentials, and anything that could help 
move laterally through the network.

```bash
smb: \> cd Amy.J
smb: \Amy.J\> ls

# Check all files in both directories
smb: \> recurse ON
smb: \> ls
```

---

## Key Lesson
This machine demonstrates SMB null sessions — connecting to Windows 
file shares without providing any credentials. SMB is one of the 
most commonly abused protocols in real enterprise environments 
because it is present on virtually every Windows machine and is 
frequently misconfigured.

A null session occurs when a server allows anonymous connections. 
This was a widespread vulnerability in older Windows versions and 
is still found today on misconfigured systems. Even when a null 
session only gives read access, the information gathered — 
usernames, file names, internal documents — is extremely valuable 
for planning further attacks.

In a real penetration test, SMB enumeration is one of the first 
things to check after Nmap. Tools like `enum4linux` and 
`crackmapexec` automate the process of listing shares, users, 
and group memberships. Finding a world-readable share in a 
corporate environment often leads to credentials, internal 
documentation, or configuration files that enable lateral movement.

The fix is to disable null sessions via Group Policy, restrict 
share permissions to authenticated users only, and audit all 
SMB shares regularly to ensure no sensitive data is accessible 
without credentials.
