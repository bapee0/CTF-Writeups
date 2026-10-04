
## 1. Reconocimiento

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 9.6p1 Ubuntu |
| 80 | HTTP | Apache httpd 2.4.58 (Ubuntu) |

El hostname que nmap resuelve es `gatekeeperhr.com`.

---

## 2. Enumeración web — credenciales en comentario HTML

```bash
curl http://172.17.0.2
```

La respuesta es la página por defecto de Apache. Sin contenido aparente. Se revisa el código fuente completo y al final, después de todo el HTML, aparece un comentario con dos cadenas en base64 separadas por ` : `:

```
#.........................................................................................................ZGFuaWVsYQ== : Zm9jYXJvamE=
```

Se decodifican las dos cadenas por separado:

```bash
echo "ZGFuaWVsYQ==" | base64 -d   # daniela
echo "Zm9jYXJvamE=" | base64 -d   # focaroja
```

Credenciales: `daniela:focaroja`

---

## 3. Acceso SSH como daniela

```bash
ssh daniela@172.17.0.2
# contraseña: focaroja
```

```
daniela@d3640eec253b:~$
```

---

## 4. Movimiento lateral a diego — base64 en directorio oculto

```bash
ls -lah
```

El home de daniela tiene dos directorios de interés: `Desktop/` y `.secreto/`.

```bash
cat Desktop/nota
```

```
Daniela no recuerdo donde guarde la password de root, si la encuentras me dices.
```

```bash
cat .secreto/passdiego
```

```
YmFsbGVuYW5lZ3Jh
```

```bash
echo "YmFsbGVuYW5lZ3Jh" | base64 -d
```

```
ballenanegra
```

```bash
su diego
# contraseña: ballenanegra
```

---

## 5. Movimiento lateral a root — acertijo en archivo oculto

El home de diego tiene un archivo `pass` con un mensaje de pista y un directorio `.passroot/` con `.pass`:

```bash
cat pass
```

```
donde estara?
```

```bash
cat .passroot/.pass
```

```
YWNhdGFtcG9jb2VzdGE=
```

```bash
echo "YWNhdGFtcG9jb2VzdGE=" | base64 -d
```

```
acatampocoesta
```

Esta cadena no es la contraseña de root — es otra pista ("a cat tampoco está"). Se sigue explorando. Dentro de `.local/share/` hay un archivo con nombre `.-`:

```bash
cat .local/share/.-
```

```
password de root

En un mundo de hielo, me muevo sin prisa,
con un pelaje que brilla, como la brisa.
No soy un rey, pero en cuentos soy fiel,
de un color inusual, como el cielo y el mar tambien.
Soy amigo de los niños, en historias de ensueño.
Quien soy, que en el frio encuentro mi dueño?

osoazul
```

El acertijo describe a un oso azul. La respuesta está incluida al final del propio archivo: `osoazul`.

```bash
su root
# contraseña: osoazul
```

```
root@d3640eec253b:~# id
uid=0(root) gid=0(root) groups=0(root)
```