# 📊 Log Analiz (Log Analysis) Eksploytasiya Qeydləri

Bu sənəd veb server (Apache/Nginx) və sistem loglarını analiz etmək, zərərli hücumları (SQLi, LFI, XSS) tapmaq və kiber təhlükəsizlik insidentlərini araşdırmaq üçün Linux komanda sətri (CLI) alətlərini əhatə edir.

(* cut -d' ' -f1 users > usernames)
---

## 🛠️ 1. Əsas Log Analiz Komandaları (Core Tools)

Log fayllarını oxumaq və məlumatı manipulyasiya etmək üçün bu vasitələrin iş məntiqini bilmək mütləqdir:

| Komanda | Funksiyası | İmtahanda İstifadə Məqsədi |
| :--- | :--- | :--- |
| `grep` | Mətn daxilində xüsusi sözləri və ya nümunələri (regex) axtarır. | Hücum payload-larını (məsələn: `union`, `select`) tapmaq üçün. |
| `awk` | Sətirləri boşluqlara görə sütunlara bölür və xüsusi sütunu çap edir. | IP ünvanlarını (`$1`) və ya HTTP status kodlarını (`$9`) ayırmaq üçün. |
| `sort` | Məlumatları əlifba və ya rəqəm sırasına görə düzür. | `uniq` komandasından əvvəl məlumatları qruplaşdırmaq üçün. |
| `uniq -c` | Ardıcıl təkrarlanan sətirləri sayır. | Hansı IP-nin neçə dəfə müraciət etdiyini öyrənmək üçün. |
| `head` / `tail` | Faylın yalnız ilk və ya son sətirlərini göstərir. | Ən çox müraciət edən "Top 10" siyahısını çıxarmaq üçün. |
| `wc -l` | Nəticədəki sətirlərin (line) sayını hesablayır. | Ümumi xətaların və ya hücum cəhdlərinin sayını tapmaq üçün. |

---

## 📂 2. Veb Server Loglarının Strukturu (Apache/Nginx)

Komandaları düzgün yazmaq üçün log sətirinin hansı sütunlardan (Field) ibarət olduğunu bilməlisən. Standart bir Apache log sətri belə görünür:

> `192.168.1.15 - - [24/Jun/2026:10:00:01 +0400] "GET /index.php?id=1 HTTP/1.1" 200 1024 "-" "Mozilla/5.0"`

* **$1** -> IP Ünvanı (`192.168.1.15`)
* **$4 / $5** -> Tarix və Saat (`[24/Jun/2026:10:00:01 +0400]`)
* **$6** -> HTTP Metodu (`"GET`)
* **$7** -> Tələb olunan URL/Path (`/index.php?id=1`)
* **$9** -> HTTP Status Kodu (`200`)
* **$10** -> Cavabın Ölçüsü (`1024`)

---

## 🚀 3. Sürətli Analiz Komandaları (İmtahan Ssenariləri)

Aşağıdakı komandaları imtahanda verilən log faylının adına uyğunlaşdıraraq (`access.log`) birbaşa terminalda icra edə bilərsən.

### Məlumatların Çıxarılması və Sayılması
* **Bütün unikal IP ünvanlarının siyahısını çıxarmaq:**
  `awk '{print $1}' access.log | sort | uniq`

* **Ən çox müraciət edən TOP 10 IP ünvanını (və sayını) tapmaq:**
  `awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -n 10`

* **Ən çox tələb olunan TOP 10 URL-i tapmaq:**
  `awk '{print $7}' access.log | sort | uniq -c | sort -nr | head -n 10`

### HTTP Xətalarının Analizi
* **404 (Not Found) xətası alan bütün müraciətləri tapmaq:**
  `awk '($9 == 404)' access.log`

* **500 (Internal Server Error) xətası alan IP-ləri çıxarmaq:**
  `awk '($9 == 500) {print $1}' access.log | sort | uniq -c | sort -nr`

* **Günün müəyyən bir saatında (məsələn, 10:00-10:59 arası) olan müraciətləri görmək:**
  `grep "24/Jun/2026:10:" access.log`

---

## 🛡️ 4. Zərərli Fəaliyyətlərin (Hücumların) Aşkarlanması

Təhlükəsizlik analizi zamanı xüsusi açar sözlərdən istifadə edərək kiber hücumları filter edirik. `-i` parametri böyük/kiçik hərfləri nəzərə almamaq üçün, `-E` parametri isə birdən çox sözü (Regex) axtarmaq üçündür.

### A. SQL Injection (SQLi) Cəhdləri
SQL hücumlarında istifadə olunan əsas komandaları URL daxilində axtarmaq:
`grep -i -E "union|select|insert|update|delete|drop|sleep|--|%27" access.log`

### B. Directory Traversal / LFI (Fayl Oxuma) Cəhdləri
Sistem fayllarına daxil olmaq üçün istifadə edilən `../` simvollarını və məşhur Linux fayllarını axtarmaq:
`grep -i -E "\.\./|\.\.\\|/etc/passwd|/etc/shadow|boot.ini" access.log`

### C. Cross-Site Scripting (XSS) Cəhdləri
URL-də JavaScript kodlarının işlədilməyə çalışılmasını aşkar etmək:
`grep -i -E "<script>|%3Cscript%3E|javascript:|alert\(" access.log`

### D. Zərərli Botlar və Avtomatlaşdırılmış Skanerlər (User-Agent Analizi)
Hücumçuların istifadə etdiyi alətlərin buraxdığı izləri tapmaq:
`grep -i -E "nmap|sqlmap|nikto|dirbuster|gobuster|curl|wget" access.log`

*Qeyd: Yalnız skanerlərin IP-lərini çıxarmaq üçün:*
`grep -i -E "sqlmap|nikto" access.log | awk '{print $1}' | sort | uniq -c`

---

## 🔑 5. SSH və Autentifikasiya Loglarının Analizi (`auth.log`)

Linux sistemlərində giriş cəhdləri adətən `/var/log/auth.log` (və ya CentOS-da `/var/log/secure`) faylına yazılır. Bu faylları analiz edərək Brute-Force (kaba qüvvə) hücumlarını tapa bilərik.

* **Bütün uğursuz (Failed) SSH giriş cəhdlərini tapmaq:**
  `grep "Failed password" /var/log/auth.log`

* **Uğursuz giriş cəhdi edən IP-ləri və onların sayını tapmaq:**
  `grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr`
  *(Burada `$(NF-3)` ifadəsi sondan 3-cü sütunu, yəni IP-ni götürür)*

* **Sistemə UĞURLA daxil olmuş istifadəçiləri və IP-ləri görmək:**
  `grep "Accepted password" /var/log/auth.log`
