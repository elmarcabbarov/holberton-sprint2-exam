```markdown
# 🛡️ IDOR (Insecure Direct Object Reference) Eksploytasiya Qeydləri

Bu sənəd praktiki imtahanlar və CTF-lər zamanı IDOR zəifliklərini tapmaq, analiz etmək və eksployt etmək üçün lazım olan metodologiya, alətlər və komandaları əhatə edir.

---

## 📌 1. IDOR Hücum Axışı (Holberton Metodologiyası)

Tətbiqdə IDOR zəifliyini yoxlayarkən bu 4 taktiki addımı izlə:

1. **Kəşfiyyat (Reconnaissance):** Saytda gəziş və ID-lərin (`id`, `uid`, `uuid`, `account`, `number`) göründüyü hər bir nöqtəni (URL, POST body, Başlıqlar/Headers) qeyd et.
2. **İlkin Vəziyyətin Tutulması (Capture Baseline):** **Burp Suite** vasitəsilə öz hesabına (İstifadəçi A) aid olan legitim bir sorğunu (request) tut. Cavabın (response) ölçüsünü və strukturunu qeyd et.
3. **İstinadın Yaradılması (Create Reference):** İkinci bir istifadəçi kimi (İstifadəçi B) daxil ol və ya digər bir istifadəçiyə aid ID tap. İstifadəçi B-nin Sessiya Tokenini (Cookie) qeyd et.
4. **Dəyişdirmə və Yoxlama (Swap & Test):** Sorğunu **İstifadəçi B-nin Sessiya Tokeni** ilə göndər, lakin daxildəki **ID-ni İstifadəçi A-nın ID-si** ilə dəyiş (və ya əksinə). Digər istifadəçiyə aid məlumatları oxuya, dəyişdirə və ya silə bildiyini yoxla.

---

## 🛠️ 2. Manual Eksploytasiya və Parametrlərin Dəyişdirilməsi

### A. URL Parametrlərinin Dəyişdirilməsi (Parameter Tampering)
Ünvanda (URL) təxmin edilə bilən rəqəmsal ID-ləri axtar və onları artırıb/azaltmaqla yoxla.
* **Orijinal:** `https://target.com/profile?id=1337`
* **Hücum:** `https://target.com/profile?id=1338`

### B. POST/PUT Body Dəyişiklikləri
Məlumat yenilənməsi və ya yaradılması zamanı göndərilən JSON və ya XML datalarını yoxla.
```json
// Orijinal Sorğu (User A)
{
  "user_id": 42,
  "email": "menimhesabim@mail.com"
}

// Hücum Sorğusu (ID-ni qurbanın ID-si ilə dəyişirik)
{
  "user_id": 43,
  "email": "hacked@mail.com"
}

```

### C. HTTP Metodunun Dəyişdirilməsi (HTTP Method Tampering)

Əgər `/api/v1/invoice/1001` ünvanına göndərilən `GET` sorğusu 403 Forbidden (Giriş qadağandır) qaytararsa, digər metodları yoxla. Ola bilsin ki, sistem digər metodlarda avtorizasiyanı yoxlamır.

* Yoxla: `POST`, `PUT`, `DELETE`, `PATCH`

### D. Parametr Çirklənməsi (Parameter Pollution - HPP)

Arxa fondakı proqram məntiqini çaşdırmaq üçün eyni parametri sorğuda iki dəfə göndər.

* `GET /api/get_profile?user_id=qurbanin_id&user_id=senin_id`
* `GET /api/get_profile?user_id=senin_id&user_id=qurbanin_id`

---

## 🚀 3. Qabaqcıl Yan keçmə Üsulları (Advanced Bypasses)

### A. Kodlaşdırma və Heşləmə (Encoding & Hashing)

Əgər ID-lər birbaşa rəqəm deyilsə, onların kodlaşdırılıb-kodlaşdırılmadığını yoxla. Dekod etmək üçün **CyberChef** alətindən istifadə et.

* **Base64:** `id=MTMzNw==` (Dekod edildikdə `1337` olur). Növbəti ID-ni tapmaq üçün `1338`-i Base64-ə çevir -> `MTMzOA==`
* **Hex:** `id=31333337` (`1337` rəqəminin Hex formatı).
* **MD5/SHA1:** Əgər `id=c4ca4238a0b923820dcc509a6f75849b` görsən, bu `1` rəqəminin MD5 heşidir. Qurbanın ID-sinin (məsələn, `2`) MD5 heşini tap (`c81e728d9d4c2f636f067f89cc14862c`) və sorğuda əvəzlə.

### B. Content-Type Dəyişdirilməsi

Əgər API sənin JSON daxilində etdiyin ID dəyişikliyini bloklayırsa, sorğunu standart forma datasına (form data) və ya əksinə çevir.

* `Content-Type: application/json` hissəsini `Content-Type: application/x-www-form-urlencoded` olaraq dəyiş.

### C. API Versiyaları Arasında Keçid

Bəzən köhnə API versiyalarında təhlükəsizlik yoxlanışları unudulur.

* `GET /api/v2/user/1337` (403 Forbidden)
* `GET /api/v1/user/1337` (200 OK - Zəiflik tapıldı!)

---

## 🤖 4. Avtomatlaşdırılmış Alətlər (Automated Tools)

### A. Burp Suite Extensions (İmtahanda Ən Sürətli Yol)

İmtahan zamanı manual olaraq hər sorğunu dəyişmək çox vaxt aparır. Bu plaginlərdən istifadə et:

1. **Autorize (Mütləq İstifadə Et):** * İstifadəçi B-nin Sessiya Kökisini (Cookie) Autorize bölməsinə yapışdır və plagini aktiv et (`Autorize is on`).
* Brauzerdə İstifadəçi A olaraq normal şəkildə gəziş.
* Autorize arxa fonda hər bir sorğunu İstifadəçi B-nin adından təkrar göndərəcək. Uğurlu keçidləri **Yaşıl** (Bypassed) və ya **Narıncı** (Is enforced?? - Diqqət yetirilməli) rənglərlə göstərəcək.


2. **AutoRepeater:** Sorğulardakı müəyyən elementləri (məsələn, istifadəçi ID-lərini) avtomatik dəyişdirib nəticələri müqayisə edir.

### B. Komanda Sətri ilə Fuzzing (`ffuf` və `curl`)

Əgər çoxlu sayda ID-ni sürətlə yoxlamaq (Brute-force) lazımdırsa:

```bash
# Sürətli şəkildə dövr (loop) daxilində curl ilə ID-lərin yoxlanması
for id in {1000..1050}; do curl -s -H "Cookie: session=SENIN_COOKIE_DEYERIN" "[http://target.com/dashboard?id=$id](http://target.com/dashboard?id=$id)" | grep -i "user_profile"; done

# ffuf vasitəsilə IDOR Fuzzing (Parametr yoxlanması)
ffuf -u '[http://target.com/index.php?page=FUZZ](http://target.com/index.php?page=FUZZ)' \
     -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
     -fs 0 \
     -mc 200

```

---

## 🌐 5. Faydalı Onlayn və Lokal Resurslar

* **CyberChef:** [https://gchq.github.io/CyberChef/](https://gchq.github.io/CyberChef/) (Sürətli Base64, Hex, URL kodlaşdırmaları və Heşlər üçün).
* **PayloadAllTheThings (IDOR bölməsi):** İnternetdəki ən geniş payload bazası.
* **HackTricks (IDOR):** CTF və imtahanlar üçün ən yaxşı metodoloji bələdçi sənədi.
* **Lokal Söz Siyahıları (Wordlists):** Kali/Parrot virtual maşınında parametr adlarını tapmaq üçün: `/usr/share/seclists/`

```
