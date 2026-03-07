# Intelligence

## Información del Objetivo

- **IP**: 10.129.95.154
- **Hostname**: dc.intelligence.htb
- **Dominio**: intelligence.htb
- **Dificultad**: Media
- **SO**: Windows Server (Domain Controller)

---

## 1. Reconocimiento (Reconnaissance)

### 1.1 Escaneo de Puertos - All Ports

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.95.154 -oG allPorts
```

**Resultado**: Se identificaron **18 puertos abiertos** indicando un controlador de dominio Active Directory.

### 1.2 Escaneo de Versiones y Scripts

```bash
nmap -sV -sC -p53,80,88,135,139,389,445,464,593,636,3268,3269,9389,49667,49691,49692,49710,49713 10.129.95.154 -oN target
```

---

## 2. Análisis de Puertos y Servicios

### Tabla de Puertos Abiertos

| Puerto | Protocolo | Servicio | Versión | Notas |
|--------|-----------|----------|---------|-------|
| 53 | TCP | DNS | Simple DNS Plus | Domain Controller DNS |
| 80 | TCP | HTTP | Microsoft IIS 10.0 | Sitio Intelligence (TRACE habilitado) |
| 88 | TCP | Kerberos | Microsoft Windows | Autenticación AD |
| 135 | TCP | MSRPC | Microsoft Windows | Remote Procedure Call |
| 139 | TCP | NetBIOS | Microsoft Windows | Network Basic I/O |
| 389 | TCP | LDAP | Microsoft Windows AD | Directorio activo |
| 445 | TCP | SMB | Microsoft Windows | Compartir archivos/impresoras |
| 464 | TCP | Kpasswd | ? | Cambio de contraseña Kerberos |
| 593 | TCP | HTTP-RPC | Microsoft Windows | RPC over HTTP |
| 636 | TCP | LDAP SSL | Microsoft Windows AD | LDAP seguro |
| 3268 | TCP | Global Catalog LDAP | Microsoft Windows AD | Global Catalog LDAP |
| 3269 | TCP | Global Catalog LDAPS | Microsoft Windows AD | Global Catalog LDAP SSL |
| 9389 | TCP | ADWS | .NET Message Framing | Active Directory Web Services |
| 49667 | TCP | MSRPC | Microsoft Windows | Dynamic RPC |
| 49691 | TCP | HTTP-RPC | Microsoft Windows | RPC over HTTP (Dynamic) |
| 49692 | TCP | MSRPC | Microsoft Windows | Dynamic RPC |
| 49710 | TCP | MSRPC | Microsoft Windows | Dynamic RPC |
| 49713 | TCP | MSRPC | Microsoft Windows | Dynamic RPC |

### Hallazgos de Certificados SSL/TLS

```
Subject: CN=dc.intelligence.htb
Subject Alternative Name: DNS:dc.intelligence.htb
Valid: 2021-04-19 to 2022-04-19 (EXPIRADO)
```

### Información del Sistema

```
Host: DC
OS: Windows
Clock Skew: ~8 horas diferencia con el escáner
SMB Signing: Habilitado y requerido
```

---

## 3. Servicios Críticos Identificados

### 3.1 Active Directory
- Dominio: **intelligence.htb**
- DC: **dc.intelligence.htb**
- LDAP abierto en puertos: 389, 636, 3268, 3269
- Kerberos activo en puerto 88

### 3.2 HTTP/Web (Puerto 80)
- Servidor: Microsoft IIS 10.0
- Título: Intelligence
- Métodos potencialmente peligrosos: TRACE

### 3.3 SMB (Puerto 445)
- Protocolo activo para compartir recursos
- Signing habilitado

---

## 4. Próximos Pasos

- [ ] Enumeración LDAP anónima
- [ ] Enumeración SMB - búsqueda de shares
- [ ] Análisis del sitio web HTTP
- [ ] Búsqueda de usuarios válidos en AD
- [ ] Kerberoasting
- [ ] AS-REP Roasting

---