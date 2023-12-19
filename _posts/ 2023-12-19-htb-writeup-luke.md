---
layout: single
title: Luke - Hack The Box
excerpt: ""
date: 2023-12-19
classes: wide
header:
  teaser: /assets/images/htb-writeup-luke/luke_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - CuteNews Exploitation > RCE

---

# PortScan
___

```java
catnp targeted -l java
# Nmap 7.94SVN scan initiated Tue Dec 19 09:04:14 2023 as: nmap -sCV -p21,22,80,3000,8000 -oN targeted 10.10.10.137
Nmap scan report for 10.10.10.137
Host is up (0.058s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3+ (ext.1)
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 0        0             512 Apr 14  2019 webapp
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.3
|      Logged in as ftp
|      TYPE: ASCII
|      No session upload bandwidth limit
|      No session download bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3+ (ext.1) - secure, fast, stable
|_End of status
22/tcp   open  ssh?
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp   open  http    Apache httpd 2.4.38 ((FreeBSD) PHP/7.3.3)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.38 (FreeBSD) PHP/7.3.3
|_http-title: Luke
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (application/json; charset=utf-8).
8000/tcp open  http    Ajenti http control panel
|_http-title: Ajenti

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Dec 19 09:07:09 2023 -- 1 IP address (1 host up) scanned in 175.67 seconds
```


# FTP Enumeration

**Nmap** nos reportó un **230** a la hora de intentar conectarse como **"anonymous"**, es decir, que podríamos conectarnos al servicio, así que opté por entrar & ver qué recursos habían dentro.

Descubriendo así dentro de un directorio con nombre "**webapp**" un .txt que decía lo siguiente:
```
Dear Chihiro !!

As you told me that you wanted to learn Web Development and Frontend, I can give you a little push by showing the sources of 
the actual website I've created .
Normally you should know where to look but hurry up because I will delete them soon because of our security policies ! 

Derry  
```

una vez leí éste mensaje, me dio sospecha de que **Derry** está exponiendo un recurso que no debería estarlo, así qué mas adelante procederé a hacer una enumeración mediante **Fuzzing**.

# WebSite
___
![](/assets/images/htb-writeup-luke/Pasted image 20231219095114.png)

Busqué en el código fuente, no encontré nada, la página principal no redirigiría a ningún lado así que opté por ver los otro puertos qué corrían en los protocolos **HTTP**.



## Port 3000:
![](/assets/images/htb-writeup-luke/Pasted image 20231219100023.png)

Aquí tenía que tener un **token** para poder listar el contenido **Json** del framework que corría por detrás que era un **Node.js Express**


## Port 8000:
![](/assets/images/htb-writeup-luke/Pasted image 20231219100318.png)

en éste panel debía loguearme para poder acceder al Panel del control del servicio **Ajenti**, y actualmente no conocía ninguna creenciales.

# Web Enumeration.

Tenía 3 puertos de mi interés para fuzzear, el puerto del servicio web principal que corría en el :80, luego el :3000 y por último :8000, empecé enumerando & fuzzeando el principal.

```bash
 gobuster dir -u http://luke.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php, js, bak, php.bak

===============================================================
/.                    (Status: 200) [Size: 3138]
/login.php            (Status: 200) [Size: 1593]
/member               (Status: 301) [Size: 231] [--> http://luke.htb/member/]
/management           (Status: 401) [Size: 381]
/css                  (Status: 301) [Size: 228] [--> http://luke.htb/css/]
/js                   (Status: 301) [Size: 227] [--> http://luke.htb/js/]
/vendor               (Status: 301) [Size: 231] [--> http://luke.htb/vendor/]
/config.php           (Status: 200) [Size: 202]
```

veo qué hay un directorio con nombre "**login.php**" de primera se me ocurre entrar a éste, y ir probando credenciales defaults o básicas, ya que hasta ahora no he encontrado ninguna credencial leakeada, pero ésto no resultó, así que fuí viendo los otros directorios...

una vez probé el directorio "**management**" ví que había un panel de autenticación, allí también probé credenciales defaults & básicas pero no logré nada éxitoso, así que seguí viendo los otros directorios.

uno que desde el principio me llamó la atención que esté expuesto fue el "**config.php**", ya que es un recurso que NO suele estar expuesto así que pensé que a éste recurso se refirió Derry cuando le mandó el mensaje a Chihiro.

![](/assets/images/htb-writeup-luke/Pasted image 20231219101346.png)

ésto serían credenciales para la DB, igual pruebo con esa contraseña a ver sí se está reutilizando en algún sitio... probé, y probé en varios login's con varios nombre de usuarios sin lograr ningún acceso, sin embargo... un rato después envié una petición con CURL por **POST** al :**3000** con una DATA en **Json** que llevaba esta contraseña.
```bash
curl -s -X POST "http://luke.htb:3000/login" -H 'Content-Type: application/json' -d '{"username": "admin", "password": "Zk6heYCyv6ZE9Xcg" }'
```

obteniendo la siguiente respuesta:
![](/assets/images/htb-writeup-luke/Pasted image 20231219101909.png)
Vemos que nos da un **Token** ya que las credenciales eran correctas al loguearnos, sin embargo así de primera no podemos hacer nada con el token, así que procedo a hacer una enumeración de posibles directorios en el **:3000**.

```bash
❯ gobuster dir -u http://luke.htb:3000 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20

/login                (Status: 200) [Size: 13]
/users                (Status: 200) [Size: 56]
/Login                (Status: 200) [Size: 13]

```

veo qué hay un directorio **users** así que me interesa, procedo a enviar una petición con CURL por **GET** para ver el contenido, pero arrastrando el **JWT** qué nos dió al acceder anteriormente.

```bash
curl -s -X GET "http://luke.htb:3000/users/" -H "Authorization: Bearer JWT_HERE"
```

![](/assets/images/htb-writeup-luke/Pasted image 20231219102639.png)

Viendo así contenido sensible en **JSON**, y por como se nombran sus directorios se me ocurre que podría ser una API, así que luego del directorio "**/users**" procedo a poner el nombre de éstos usuarios válidos a ver si logro ver más información.

```bash
curl -s -X GET "http://luke.htb:3000/users/Admin" -H "Authorization: JWT_HERE"
```

![](/assets/images/htb-writeup-luke/Pasted image 20231219103022.png)

Viendo así las credenciales para **Admin**, así que procedo a ver las credenciales de todos los usuarios uno por uno, para tenerlas guardadas y ver sí podríamos reutilizarlas.

Fuí probando en todos los paneles con cada usuario & sus contraseñas, logrando así en el panel de autenticación de "**Management**" acceder con las credenciales de **Derry**.

Allí había un recurso con nombre "**config.json**" que me llamó la atención, entré para ver que contenía...

![](/assets/images/htb-writeup-luke/Pasted image 20231219103649.png)

aquí encontré más credenciales, veía que decía algo de "**ajenti**" así que fuí directamente al servicio **Ajenti** del :8000 y probé estas credenciales, logrando así...

![](/assets/images/htb-writeup-luke/Pasted image 20231219103933.png)

Acceder, directamente fuí a la terminal, y me cree una terminal para ingresar & ver como quien se estaba corriendo el servicio web... 
![](/assets/images/htb-writeup-luke/Pasted image 20231219104306.png)

y para mi sorpresa el servicio lo corría **ROOT**, por ende no tendríamos que llevar a cabo una escalada de privilegios & **PWNED**!

