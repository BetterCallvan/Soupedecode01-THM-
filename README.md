<div align="center">

# 🥣 TryHackMe — SoupeDecode01

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat-square)
![OS](https://img.shields.io/badge/OS-Windows%20Server-0078D6?style=flat-square&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Category](https://img.shields.io/badge/Category-Active%20Directory-blue?style=flat-square)
![License](https://img.shields.io/badge/License-Educational%20Use-lightgrey?style=flat-square)

**Guest SMB → SID Brute-force → Password Spray → Kerberoasting → Pass-the-Hash → `NT AUTHORITY\SYSTEM`**

📖 Read the full writeup:&nbsp; **[🇷🇺 Русский](README_RU.md)** &nbsp;|&nbsp; **[🇬🇧 English](README_EN.md)**

</div>

---

## Attack Chain

```
Guest SMB Session
      │
      ▼
SID Brute-force (impacket-lookupsid)  ──►  users.txt
      │
      ▼
Password Spray (nxc, user=pass)  ──►  ybob317:ybob317
      │
      ▼
SMB Users Share  ──►  🚩 user.txt
      │
      ▼
Kerberoasting (GetUserSPNs)  ──►  service hashes
      │
      ▼
hashcat + rockyou  ──►  file_svc:Password123!!
      │
      ▼
SMB Backup Share  ──►  machine account NTLM hashes
      │
      ▼
Pass-the-Hash (FileServer$)  ──►  nt authority\system
      │
      ▼
🚩 root.txt
```

## Contents

| File | Description |
|---|---|
| [`README_RU.md`](README_RU.md) | Полный райтап на русском языке |
| [`README_EN.md`](README_EN.md) | Full writeup in English |

## Disclaimer

This writeup documents a walkthrough of a machine on [TryHackMe](https://tryhackme.com), a legal platform for learning cybersecurity in isolated lab environments. All techniques described here were applied against a system intentionally built for practice and are shared for educational purposes only.

