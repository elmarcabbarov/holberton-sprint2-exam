i# 📊 Log Analysis — Tam Cheatsheet

> İmtahan qaydaları: `grep cut find xargs du df ls sort wc uniq tr tee` — yalnız bunlar.  
> İcazəli: `|` pipe, `>` `>>` `<` redirect, `*` `?` wildcard.  
> **İcazəsiz:** `awk`, `sed`, `if`, `for`, `while`, `\` (line continuation).  
> Bütün əmrlər **tək sətirdə** yazılmalıdır.

---

## 📑 Məzmun

1. [Alətlər & Flag-lar — Tam Cədvəl](#1-alətlər--flag-lar--tam-cədvəl)
2. [Regex Əsasları (grep üçün)](#2-regex-əsasları-grep-üçün)
3. [IP Ünvanı Tapma & Sayma](#3-ip-ünvanı-tapma--sayma)
4. [Təkrar Sətirlərlə İş (uniq)](#4-təkrar-sətirlərlə-iş-uniq)
5. [Fayl & Qovluq Axtarışı (find + xargs)](#5-fayl--qovluq-axtarışı-find--xargs)
6. [Sıralama & Sayma Kombinasiyaları](#6-sıralama--sayma-kombinasiyaları)
7. [Sahə (Field) Çıxarma — cut](#7-sahə-field-çıxarma--cut)
8. [Disk & Fayl Məlumatları (du, df, ls)](#8-disk--fayl-məlumatları-du-df-ls)
9. [Output İdarəetmə (tee, redirect)](#9-output-idarəetmə-tee-redirect)
10. [Holberton Task-larının Həlli](#10-holberton-task-larının-həlli)
11. [Log Formatları & Sahə Xəritəsi](#11-log-formatları--sahə-xəritəsi)
12. [Hücum İzlərini Tapma](#12-hücum-izlərini-tapma)
13. [Quick Reference — Bütün Komandalar Bir Yerdə](#13-quick-reference--bütün-komandalar-bir-yerdə)

---

## 1. Alətlər & Flag-lar — Tam Cədvəl

### `grep` — Mətn Axtarışı

```bash
grep [flags] 'pattern' file
```

| Flag | Mənası | Nümunə |
|---|---|---|
| `-o` | Yalnız **uyğun hissəni** çap et (bütün sətiri deyil) | `grep -o '[0-9]*' file.log` |
| `-E` | **Extended regex** — `+`, `{n}`, `\|` kimi simvollar üçün | `grep -E '[0-9]{1,3}\.[0-9]{1,3}'` |
| `-r` | **Recursive** — qovluq daxilindəki bütün faylları axtar | `grep -r 'error' logs/` |
| `-v` | **İnvert** — uyğun olmayan sətirləri çap et | `grep -v '200' access.log` |
| `-c` | **Count** — uyğun sətirlərin **sayını** çap et | `grep -c 'ERROR' syslog.log` |
| `-n` | **Line number** — sətir nömrəsini göstər | `grep -n 'fail' auth.log` |
| `-w` | **Whole word** — tam söz uyğunluğu (substring deyil) | `grep -w '404' access.log` |
| `-h` | **Hide filename** — recursive-də fayl adını gizlət | `grep -rh 'ERROR' logs/` |
| `-i` | **Case-insensitive** — böyük/kiçik hərfi fərqləndirmə | `grep -i 'error' file.log` |
| `-l` | **List files** — yalnız uyğun fayl adlarını çap et | `grep -rl 'ERROR' logs/` |

> 🧠 **Ən çox lazım olan kombinasiya:** `-oE` birlikdə → pattern-in yalnız uyğun hissəsini extended regex ilə çıxar. `-rh` birlikdə → qovluqda axtar, fayl adını göstərmə.

---

### `sort` — Sıralama

```bash
sort [flags] file
```

| Flag | Mənası | Nümunə |
|---|---|---|
| `-u` | **Unique** — eyni sətirləri bir dəfə göstər | `sort -u file.log` |
| `-r` | **Reverse** — azalma sırasına görə | `sort -r file.log` |
| `-n` | **Numeric** — rəqəmsal sıralama (string deyil) | `sort -n counts.txt` |
| `-rn` | Rəqəmsal + tərsinə — **ən böyükdən** başla | `sort -rn counts.txt` |

> 🧠 **`uniq`-dən əvvəl həmişə `sort` et** — `uniq` yalnız **ardıcıl** təkrarları tapır!

---

### `uniq` — Təkrar Sətir Analizi

```bash
sort file | uniq [flags]
```

| Flag | Mənası | Nümunə |
|---|---|---|
| `-c` | **Count** — hər sətrin neçə dəfə təkrarlandığını say | `sort file.log \| uniq -c` |
| `-d` | **Duplicates only** — yalnız **təkrar** olan sətirləri göstər | `sort file.log \| uniq -d` |
| `-u` | **Unique only** — yalnız **bir dəfə** görünən sətirləri göstər | `sort file.log \| uniq -u` |
| `-cd` | Təkrarları **say** + yalnız onları göstər | `sort file.log \| uniq -cd` |

> ⚠️ **Kritik fərq:**  
> `uniq -d` → yalnız **dublikat** olan sətirləri göstərir (bir nüsxəsini)  
> `uniq -u` → yalnız **unikal** (heç təkrarlanmayan) sətirləri göstərir  
> `uniq -c` → **hər** sətri sayı ilə birlikdə göstərir  

---

### `cut` — Sahə Çıxarma

```bash
cut -d 'delimiter' -f field_number file
```

| Flag | Mənası | Nümunə |
|---|---|---|
| `-d X` | **Delimiter** — sahələri ayıran simvol | `-d ' '` (boşluq), `-d ':'` (iki nöqtə) |
| `-f N` | **Field** — N-ci sahəni götür | `-f1` (birinci), `-f2` (ikinci) |
| `-f N-M` | N-dən M-ə qədər sahələr | `-f1-3` |
| `-f N,M` | N və M sahələri (vergüllə) | `-f1,3` |

> 🧠 **`cut` boşluq delimiter-ini fərqli işlədir:** Bir boşluq ilə ayırır, çoxlu boşluq = boş sahə sayılır. Buna görə Apache log-larında birinci sahə (IP) üçün `-d' ' -f1` işləyir.

---

### `find` — Fayl Axtarışı

```bash
find path/ [flags]
```

| Flag | Mənası | Nümunə |
|---|---|---|
| `-type f` | Yalnız **fayllar** | `find logs/ -type f` |
| `-type d` | Yalnız **qovluqlar** | `find logs/ -type d` |
| `-name 'pattern'` | Ad əsasında axtarış | `find logs/ -name '*.log'` |
| `-name '*.log'` | Wildcard ilə bütün `.log` fayllar | `find . -name '*.log'` |

---

### `xargs` — Nəticəni Arqument Kimi Ver

```bash
find ... | xargs command
```

`find`-ın nəticəsini (fayl siyahısını) başqa əmrin arqumenti kimi ötürür.

```bash
# find ilə tapılan bütün faylları grep ilə axtar
find logs/ -type f | xargs grep 'ERROR'

# -h ilə fayl adını gizlət
find logs/ -type f | xargs grep -h 'ERROR'
```

---

### `wc` — Sayma

```bash
wc [flags] file
```

| Flag | Mənası | Nümunə |
|---|---|---|
| `-l` | **Lines** — sətir sayı | `wc -l file.log` |
| `-w` | **Words** — söz sayı | `wc -w file.log` |
| `-c` | **Characters/bytes** — simvol sayı | `wc -c file.log` |

> 🧠 **Pipe ilə:** `grep 'pattern' file.log | wc -l` → uyğun sətirlərin sayı

---

### `tr` — Simvol Çevirmə

```bash
tr [flags] 'set1' 'set2'
```

| Flag | Mənası | Nümunə |
|---|---|---|
| `-d 'set'` | Simvolları **sil** | `echo "1.2.3.4" \| tr -d '.'` → `1234` |
| `-s 'set'` | Ardıcıl **təkrarları** birdə **sıxış** | `tr -s ' '` |
| `'a-z' 'A-Z'` | Kiçikdən böyüyə **çevir** | `echo "hello" \| tr 'a-z' 'A-Z'` |

---

### `du` & `df` — Disk Məlumatı

```bash
du [flags] path     # directory/file ölçüsü
df [flags]          # disk bölmələrinin vəziyyəti
```

| Komanda | Mənası |
|---|---|
| `du -h file` | Faylın ölçüsü (human-readable: KB, MB) |
| `du -sh dir/` | Qovluğun **yekun** ölçüsü (`-s` = summary) |
| `du -sh logs/` | Logs qovluğunun ümumi ölçüsü |
| `df -h` | Bütün disk bölmələrinin boş/dolu vəziyyəti |

---

### `ls` — Fayl Siyahısı

```bash
ls [flags] path
```

| Flag | Mənası |
|---|---|
| `-l` | **Long listing** — icazələr, tarix, ölçü |
| `-h` | **Human-readable** — MB, KB formatında |
| `-a` | **All** — gizli fayllar da (`.*`) |
| `-lah` | Hamısı birlikdə |

---

### `tee` — Həm Ekran, Həm Fayl

```bash
command | tee [flags] output.txt
```

| Flag | Mənası |
|---|---|
| (yox) | Faylı **üzərinə yaz** (overwrite) |
| `-a` | Faylın **sonuna əlavə et** (append) |

---

## 2. Regex Əsasları (grep üçün)

> `-E` flag-i ilə extended regex istifadə et.

| Pattern | Mənası | Nümunə |
|---|---|---|
| `.` | İstənilən **bir** simvol | `1.2` → `1x2`, `1a2` |
| `*` | Əvvəlki simvoldan **sıfır və ya çox** | `ab*c` → `ac`, `abc`, `abbc` |
| `+` | Əvvəlki simvoldan **bir və ya çox** | `ab+c` → `abc`, `abbc` (ac deyil) |
| `?` | Əvvəlki simvoldan **sıfır və ya bir** | `colou?r` → `color`, `colour` |
| `{n}` | Tam **n** dəfə | `[0-9]{3}` → tam 3 rəqəm |
| `{n,m}` | **n-dən m-ə** qədər | `[0-9]{1,3}` → 1-3 rəqəm |
| `[0-9]` | Rəqəm (0-dan 9-a) | `[0-9]+` → bir və ya çox rəqəm |
| `[a-z]` | Kiçik hərf | `[a-zA-Z]+` → hərflər |
| `^` | Sətrin **başlanğıcı** | `^192` → 192 ilə başlayan sətir |
| `$` | Sətrin **sonu** | `200$` → 200 ilə bitən sətir |
| `\\.` | Literal **nöqtə** (regex nöqtəsi deyil) | `192\\.168` → `192.168` |
| `[^abc]` | Bu simvollar **xaricində** hər şey | `[^0-9]` → rəqəm olmayan |

---

## 3. IP Ünvanı Tapma & Sayma

> 🎯 **Holberton Tasks-lərinin əsas mövzusu**

### IP Format Anlayışı

```
x.x.x.x       → hər oktet 1 rəqəm (məs: 1.2.3.4)
xx.x.x.x      → birinci oktet 2 rəqəm (məs: 10.0.0.1)
xxx.xxx.xxx.xxx → hər oktet 3 rəqəm (məs: 192.168.100.200)
```

### Bütün IPv4 Ünvanlarını Tap (istənilən formatlı)

```bash
# Hər okteti 1-3 rəqəm olan IP-ləri tap (ən geniş regex)
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' log-test

# Unikal IP-lər (sort + uniq)
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' log-test | sort -u

# Neçə unikal IP var? (say)
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' log-test | sort -u | wc -l
```

### Task 1 — `x.x.x.x` Formatı (hər oktet 1 rəqəm)

```bash
# x.x.x.x = hər oktet tam 1 rəqəm
# Regex: bir rəqəm, nöqtə, bir rəqəm, nöqtə, bir rəqəm, nöqtə, bir rəqəm
grep -oE '\b[0-9]\.[0-9]\.[0-9]\.[0-9]\b' log-test

# Whole word match ilə (-w)
grep -wE '[0-9]\.[0-9]\.[0-9]\.[0-9]' log-test

# Recursive (bütün log-test qovluğunda axtar)
grep -roE '\b[0-9]\.[0-9]\.[0-9]\.[0-9]\b' log-test/

# Bütün nəticələri siyahıla
grep -oE '\b[0-9]\.[0-9]\.[0-9]\.[0-9]\b' log-test | sort -u
```

### Task 2 — `xx.xxx.xxx.xxx` Formatı (exact: 2.3.3.3 rəqəm)

```bash
# xx.xxx.xxx.xxx = 2 rəqəm, nöqtə, 3 rəqəm, nöqtə, 3 rəqəm, nöqtə, 3 rəqəm
grep -oE '\b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b' log-test

# Recursive — bütün log-test qovluğunda
grep -roE '\b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b' log-test/

# SAYI tap (-c ilə)
grep -roE '\b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b' log-test/ | wc -l

# Alternativ: grep -c (bir fayldırsa)
grep -cE '\b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b' log-test

# Whole word versiyası (əlavə simvollarla toxunmasın deyə)
grep -wrE '[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}' ./log-test
```

> 🧠 **`-c` vs `| wc -l` fərqi:**  
> `grep -c` → uyğun **sətirlərin** sayını verir (bir sətirdə çox IP olsa, yenə 1 sayır)  
> `grep -o | wc -l` → uyğun **tapıntıların** sayını verir (bir sətirdə 2 IP varsa, 2 sayır)  
> IP sayı üçün `grep -o | wc -l` daha dəqiqdir!

### IP-ləri Sayla & Sırala

```bash
# Hər IP neçə dəfə görünür (ən çoxdan aza)
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' log-test | sort | uniq -c | sort -rn

# Yalnız dublikat IP-lər (birdən çox görünənlər)
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' log-test | sort | uniq -d

# Dublikat IP-lərin sayı ilə birlikdə
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' log-test | sort | uniq -cd
```

---

## 4. Təkrar Sətirlərlə İş (uniq)

### Task 3 — Yalnız Dublikat Sətirlərin Sayı

```bash
# Metod 1: uniq -d ilə dublikatları tap, wc ilə say
sort log-test | uniq -d | wc -l

# Metod 2: uniq -cd ilə dublikatları say ilə göstər
sort log-test | uniq -cd

# Metod 3: Neçə fərqli dublikat var? (unikal dublikat sayı)
sort log-test | uniq -d | wc -l
```

> 🧠 **Fərqi anla:**
> ```
> Fayl məzmunu:          sort + uniq -cd nəticəsi:
> apple                      2 apple       ← dublikat (2 dəfə)
> apple                      3 banana      ← dublikat (3 dəfə)
> banana
> banana                 sort + uniq -u nəticəsi:
> banana                     cherry        ← unikal (1 dəfə)
> cherry
> ```
> `uniq -d | wc -l` → **2** (iki fərqli dublikat sətir var: "apple" və "banana")

### uniq Kombinasiyaları

```bash
# Bütün sətirləri say ilə göstər (ən çoxdan aza)
sort file.log | uniq -c | sort -rn

# Yalnız bir dəfə görünən sətirləri göstər (unikal)
sort file.log | uniq -u

# Yalnız birdən çox görünən sətirləri göstər (dublikat)
sort file.log | uniq -d

# Dublikatları say ilə göstər (neçə dəfə təkrarlanıb)
sort file.log | uniq -cd

# Unikal sıralanmış sətir siyahısı
sort -u file.log
```

---

## 5. Fayl & Qovluq Axtarışı (find + xargs)

```bash
# log-test qovluğundakı bütün faylları siyahıla
find log-test/ -type f

# Adı .log ilə bitən faylları tap
find log-test/ -type f -name '*.log'

# Qovluqları tap
find log-test/ -type d

# Tapılan faylların hər birini grep ilə axtar
find log-test/ -type f | xargs grep 'ERROR'

# Fayl adını gizlət (-h)
find log-test/ -type f | xargs grep -h 'ERROR'

# IP pattern-i bütün faylda axtar
find log-test/ -type f | xargs grep -hE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}'

# Alternativ: grep -rh (eyni nəticə, daha qısa)
grep -rh 'ERROR' log-test/
grep -roE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' log-test/
```

---

## 6. Sıralama & Sayma Kombinasiyaları

### Ən Çox Təkrarlanan Sətirləri Tap

```bash
# Hər sətir neçə dəfə görünür (ən çoxdan aza)
sort file.log | uniq -c | sort -rn

# Yalnız ən çox 5 sətri göstər
sort file.log | uniq -c | sort -rn | head -5

# Ən az görünən sətirləri göstər
sort file.log | uniq -c | sort -n | head -5
```

### Boş Olmayan Sətirləri Say

```bash
# Boş sətirləri say (yalnız boş)
grep -c '^$' file.log

# Boş olmayan sətirləri say
grep -vc '^$' file.log

# Boşluq olan sətirləri də boş say
grep -vc '^[[:space:]]*$' file.log
```

---

## 7. Sahə (Field) Çıxarma — cut

Apache log-larında sahələr **boşluq** ilə ayrılır:

```
192.168.1.15 - - [24/Jun/2026:10:00:01] "GET /index.php" 200 1024
    1        2 3          4               5       6        7    8
```

```bash
# 1-ci sahə = IP ünvanı
cut -d' ' -f1 access.log

# 7-ci sahə = URL/path (GET-dən sonrakı)
cut -d' ' -f7 access.log

# Bir neçə sahə birlikdə (1 və 7)
cut -d' ' -f1,7 access.log

# Sahə aralığı (1-dən 3-ə)
cut -d' ' -f1-3 access.log

# Syslog - colon delimiter ilə (user:password:uid:...)
cut -d':' -f1 /etc/passwd        # Yalnız username-lər

# Unikal IP-ləri çıxar (cut + sort + uniq)
cut -d' ' -f1 access.log | sort -u

# Ən çox müraciət edən IP-lər
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn
```

> 🧠 **`cut` vs IP regex:**  
> `cut -d' ' -f1` → log-un **birinci sahəsini** alır (IP fərz edilir, amma qarışıq log-larda yanlış ola bilər)  
> `grep -oE '[0-9]{...}'` → istənilən yerdən IP-ni dəqiq çəkir  
> Log strukturu sabitdirsə `cut`, qarışıqdırsa `grep -oE` istifadə et.

---

## 8. Disk & Fayl Məlumatları (du, df, ls)

```bash
# Qovluğun ümumi ölçüsü
du -sh logs/

# Hər alt-qovluğun ölçüsü
du -h logs/

# Bütün disk bölmələrinin vəziyyəti
df -h

# Faylları ölçüyə görə sırala (ən böyükdən)
ls -lh logs/ | sort -rk5

# Gizli fayllar daxil tam siyahı
ls -lah logs/

# Yalnız fayl adları (siyahı üçün)
ls logs/
ls logs/*.log
```

---

## 9. Output İdarəetmə (tee, redirect)

```bash
# Ekranda gör VƏ fayla yaz (overwrite)
grep 'ERROR' file.log | tee output.txt

# Ekranda gör VƏ faylın sonuna əlavə et (append)
grep 'ERROR' file.log | tee -a output.txt

# Yalnız fayla yaz (ekranda görünmür) — overwrite
grep 'ERROR' file.log > output.txt

# Faylın sonuna əlavə et
grep 'ERROR' file.log >> output.txt

# Bir komandanın nəticəsini başqa komandaya ver
grep 'ERROR' file.log | wc -l > count.txt

# Birdən çox pipe
grep 'ERROR' file.log | sort | uniq -c | sort -rn | tee result.txt
```

---

## 10. Holberton Task-larının Həlli

### Task 1 — `x.x.x.x` formatındakı IP-ləri tap

```
Şərt: hər oktet tam 1 rəqəm (x.x.x.x)
Regex: \b[0-9]\.[0-9]\.[0-9]\.[0-9]\b
```

```bash
# Tapıntıları göstər
grep -oE '\b[0-9]\.[0-9]\.[0-9]\.[0-9]\b' log-test

# Qovluqsa recursive
grep -roE '\b[0-9]\.[0-9]\.[0-9]\.[0-9]\b' log-test/

# Fayl adı olmadan
grep -roE '\b[0-9]\.[0-9]\.[0-9]\.[0-9]\b' log-test/ | grep -oE '[0-9]\.[0-9]\.[0-9]\.[0-9]'

# Unikal siyahı
grep -roE '\b[0-9]\.[0-9]\.[0-9]\.[0-9]\b' log-test/ | sort -u
```

### Task 2 — `xx.xxx.xxx.xxx` formatındakı IP-lərin SAYI

```
Şərt: 2.3.3.3 rəqəm (xx.xxx.xxx.xxx)
Regex: \b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b
```

```bash
# Tapıntıları göstər
grep -roE '\b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b' log-test/

# Sayı tap
grep -roE '\b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b' log-test/ | wc -l

# Holberton-da tək fayl olduqda
grep -oE '\b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b' log-test | wc -l

# Whole word versiyası
grep -wrE '[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}' ./log-test | wc -l
```

### Task 3 — Yalnız Dublikat Sətirlərin Sayı

```
Şərt: uniq -d ilə dublikat olan sətirlərin sayını tap
```

```bash
# Dublikat sətirləri göstər
sort log-test | uniq -d

# Dublikat sətirlərin SAYI
sort log-test | uniq -d | wc -l

# Dublikatları say ilə göstər (neçə dəfə təkrarlanıb)
sort log-test | uniq -cd

# Yalnız unikal sətirləri göstər (dublikat olmayan)
sort log-test | uniq -u
```

---

## 11. Log Formatları & Sahə Xəritəsi

### Apache/Nginx Access Log (Combined Format)

```
192.168.1.15 - frank [24/Jun/2026:10:00:01 +0400] "GET /index.php?id=1 HTTP/1.1" 200 1024 "-" "Mozilla/5.0"
     1        2    3             4                        5                    6    7    8   9       10
```

| Sahə # | Məzmun | cut ilə | Nümunə |
|---|---|---|---|
| `f1` | IP ünvanı | `cut -d' ' -f1` | `192.168.1.15` |
| `f2` | İdent (adətən `-`) | `cut -d' ' -f2` | `-` |
| `f3` | Auth user | `cut -d' ' -f3` | `frank` |
| `f4,f5` | Tarix+saat | `cut -d' ' -f4,5` | `[24/Jun/2026:10:00:01 +0400]` |
| `f6` | Method+URL | `cut -d' ' -f6,7` | `"GET /index.php?id=1` |
| `f7` | URL/Path | `cut -d' ' -f7` | `/index.php?id=1` |
| `f9` | HTTP Status | `cut -d' ' -f9` | `200` |
| `f10` | Response size | `cut -d' ' -f10` | `1024` |

### Syslog Format

```
Jun 24 10:00:01 hostname sshd[1234]: Failed password for root from 192.168.1.5 port 22
```

```bash
# Hansı servisden gəlir
cut -d' ' -f5 auth.log

# Uğursuz giriş cəhdlərini tap
grep 'Failed password' auth.log

# Uğursuz giriş cəhdlərinin sayı
grep -c 'Failed password' auth.log

# Uğurlu girişləri tap
grep 'Accepted password' auth.log
```

---

## 12. Hücum İzlərini Tapma

> ⚠️ Bu bölmə log-larda keçmiş hücumları **forensic** məqsədli analiz üçündür.

### SQL Injection İzləri

```bash
grep -iE 'union|select|insert|drop|sleep|--' access.log
grep -icE 'union|select|insert|drop|sleep' access.log
```

### Directory Traversal / LFI

```bash
grep -iE '\.\./|/etc/passwd|/etc/shadow' access.log
grep -c '\.\.' access.log
```

### XSS Cəhdləri

```bash
grep -iE '<script>|javascript:|alert\(' access.log
grep -icE '<script>|%3Cscript' access.log
```

### Skaner İzləri (User-Agent)

```bash
grep -iE 'sqlmap|nikto|nmap|gobuster|dirbuster' access.log
grep -ic 'sqlmap' access.log
```

### HTTP Status Koduna görə Filtrləmə

```bash
# 404 xətaları (tapılmayan səhifələr — directory brute force əlaməti)
grep ' 404 ' access.log

# 500 xətaları (server xətaları — injection cəhdi ola bilər)
grep ' 500 ' access.log

# 200 olan uğurlu sorğular
grep ' 200 ' access.log

# 4xx xəta sətirləri
grep -E ' 4[0-9]{2} ' access.log

# 404-lərin sayı
grep -c ' 404 ' access.log
```

### SSH Brute Force Analizi

```bash
# Uğursuz SSH giriş cəhdləri
grep 'Failed password' auth.log | wc -l

# Hansı usernamə qarşı cəhd edilib
grep 'Failed password' auth.log | grep -oE 'for [a-zA-Z0-9_]+' | sort | uniq -c | sort -rn

# Hansı IP-dən gəlir (sondan 4-cü söz)
grep 'Failed password' auth.log | grep -oE 'from [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | sort | uniq -c | sort -rn
```

---

## 13. Quick Reference — Bütün Komandalar Bir Yerdə

```bash
# ===== GREP =====
grep 'pattern' file.log                         # Sadə axtarış
grep -oE 'regex' file.log                        # Yalnız uyğun hissəni çıxar
grep -c 'pattern' file.log                       # Uyğun sətirləri say
grep -v 'pattern' file.log                       # Uyğun olmayanları göstər
grep -r 'pattern' logs/                          # Recursive axtarış
grep -rh 'pattern' logs/                         # Recursive, fayl adı olmadan
grep -n 'pattern' file.log                       # Sətir nömrəsi ilə
grep -w 'word' file.log                          # Tam söz uyğunluğu
grep -i 'pattern' file.log                       # Case-insensitive
grep -vc '^$' file.log                           # Boş olmayan sətirləri say

# ===== IP REGEX-LƏRİ =====
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' file  # istənilən IPv4
grep -oE '\b[0-9]\.[0-9]\.[0-9]\.[0-9]\b' file                   # x.x.x.x (1.1.1.1)
grep -oE '\b[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}\b' file       # xx.xxx.xxx.xxx
grep -wrE '[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}' ./log-test    # Holberton Task 2

# ===== SORT =====
sort file.log                                    # Əlifba sırası
sort -u file.log                                 # Unikal sıralı
sort -rn file.log                                # Rəqəmsal, tərsinə
sort file.log | uniq -c | sort -rn              # Ən çox təkrarlanan yuxarıda

# ===== UNIQ =====
sort file.log | uniq                             # Dublikatları sil
sort file.log | uniq -c                          # Hər sətri sayla
sort file.log | uniq -d                          # Yalnız dublikatları göstər
sort file.log | uniq -u                          # Yalnız unikal sətirləri göstər
sort file.log | uniq -cd                         # Dublikatları say ilə göstər
sort file.log | uniq -d | wc -l                  # Dublikat sətirlərin sayı

# ===== CUT =====
cut -d' ' -f1 access.log                        # 1-ci sahə (IP)
cut -d' ' -f7 access.log                        # 7-ci sahə (URL)
cut -d':' -f1 /etc/passwd                       # Colon ilə username
cut -d' ' -f1,7 access.log                      # 1 və 7-ci sahələr

# ===== FIND + XARGS =====
find logs/ -type f                               # Bütün fayllar
find logs/ -type f -name '*.log'                 # .log faylları
find logs/ -type f | xargs grep 'ERROR'          # Faylları grep ilə axtar
find logs/ -type f | xargs grep -h 'ERROR'       # Fayl adı olmadan

# ===== WC =====
wc -l file.log                                   # Sətir sayı
wc -w file.log                                   # Söz sayı
grep 'pattern' file.log | wc -l                  # Uyğun sətirlərin sayı

# ===== SAYMA KOMBİNASİYALARI =====
grep -oE '[regex]' file | wc -l                  # Uyğun tapıntıların sayı
grep -oE '[regex]' file | sort -u | wc -l        # Unikal tapıntıların sayı
grep -oE '[regex]' file | sort | uniq -c | sort -rn  # Hər birini sayla

# ===== DU, DF, LS =====
du -sh logs/                                     # Qovluq ölçüsü
df -h                                            # Disk vəziyyəti
ls -lah logs/                                    # Detallı siyahı

# ===== TEE & REDIRECT =====
grep 'ERROR' file.log | tee output.txt           # Həm ekran, həm fayl
grep 'ERROR' file.log | tee -a output.txt        # Əlavə et
grep 'ERROR' file.log > output.txt               # Yalnız fayla yaz
grep 'ERROR' file.log >> output.txt              # Sonuna əlavə et

# ===== TR =====
echo "1.2.3.4" | tr -d '.'                      # Nöqtələri sil → 1234
echo "hello" | tr 'a-z' 'A-Z'                   # Böyük hərflərə çevir
```

---
