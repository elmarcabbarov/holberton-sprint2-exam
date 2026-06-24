## LDAP

Discover the Base DN first

```bash
ldapsearch -H ldap://htb.local -x -s base namingcontexts
```

Full anonymous enumeration

```bash
ldapsearch -H ldap://htb.local -x -b “DC=htb,DC=local”
```

Enumerate all users:

```bash
ldapsearch -H ldap://htb.local -x -b “DC=htb,DC=local” "(objectClass=user)"
sAMAccountName cn description
```

sonra Evil Winrm 

## SMB

# SMB commands

```
smbclient-L //IP-U'%'
```

Anonymous share enumeration.

```
smbclient-NL //IP
```

Anonymous enumeration alternativi.

```
netexec smb IP-u''-p''
```

Null session test.

```
netexec smb IP-u''-p''--shares
```

Share və permission-ları göstər.

```
smbclient-N //IP/SHARE
```

Share-ə qoşul.

```
ls
```

Faylları göstər.

```
cd folder
```

Qovluğa gir.

```
cd ..
```

Geri çıx.

```
get file.txt
```

Faylı yüklə.

```
mget *
```

Hamısını yüklə.

```
exit
```
