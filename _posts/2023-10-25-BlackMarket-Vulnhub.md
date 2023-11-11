---
layout: single
title: BlackMarket - Vulnhub
date: 25/10/2023
classes: wide
header:
  teaser:
  icon: /assets/images/vulnhub.webp
tags:
  - SQLI
  - reverse shell
  - ssh username enumeration
  - sudo abuse
  - linux
  - fácil
---

Para vulnerar la máquina BlackMarket, debemos hacer un reconocimiento de puertos, enumeración de servicios & posibles usuarios con las flags que se nos van dejando mediante avanzamos a la máquina, para luego con esas credenciales entrar en un panel el cual es vulnerable a SQLI, allí obtendremos las credenciales para loguearnos en el panel del mercado negro, y seguiremos obteniendo pistas hasta que finalmene ingresemos a la máquina & escalemos privilegios.


# **PortScan**
______

```
# Nmap 7.93 scan initiated Wed Oct 25 09:36:13 2023 as: nmap -sCV -p21,22,80,110,143,993,995 -oN targeted 192.168.204.151
Nmap scan report for 192.168.204.151
Host is up (0.00044s latency).
PORT    STATE SERVICE    VERSION
21/tcp  open  ftp        vsftpd 3.0.2
22/tcp  open  ssh        OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 a99884aa907ef1e6bec0843efaaa838a (DSA)
|   2048 075c7715305a17958e0f91f02d0bc37a (RSA)
|   256 2f9c29b5f5dcf495076d41eef90d15b8 (ECDSA)
|_  256 24ac30c7797f43ccfc23dfeadbbb4acc (ED25519)
80/tcp  open  http       Apache httpd 2.4.7 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: BlackMarket Weapon Management System
|_http-server-header: Apache/2.4.7 (Ubuntu)
110/tcp open  pop3       Dovecot pop3d
|_pop3-capabilities: SASL UIDL CAPA PIPELINING AUTH-RESP-CODE STLS RESP-CODES TOP
|_ssl-date: TLS randomness does not represent time
143/tcp open  imap       Dovecot imapd (Ubuntu)
|_imap-capabilities: ID SASL-IR have more post-login capabilities listed LITERAL+ STARTTLS LOGIN-REFERRALS Pre-login OK IMAP4rev1 ENAB
LE
|_ssl-date: TLS randomness does not represent time
993/tcp open  ssl/imaps?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2017-11-01T07:05:35
|_Not valid after:  2027-11-01T07:05:35
995/tcp open  ssl/pop3s?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2017-11-01T07:05:35
|_Not valid after:  2027-11-01T07:05:35
```
_____
# **BlackMarket**
_____


El servidor web en el :80 es para loguearnos.

![](/assets/images/blackmarket/Pasted image 20231025094113.png)

Que en el código fuente nos da una pista.


____
# **Enumeración de usuarios SSH.**
___


Nos aprovechamos qué la versión del OpenSSH es vulnerable a enumeración de usuarios, esto al verlo en Searchsploit.
![](/assets/images/blackmarket/Pasted image 20231025095641.png)

Usamos este script para validar usuarios qué sospechamos tras investigar con la pista del código fuente, hasta qué encontramos uno qué es válido.

![](/assets/images/blackmarket/Pasted image 20231025100302.png)

Ya conocemos un usuario válido, por ende, de la misma fuente de donde sacamos la información de posibles usuarios tras la pista del código fuente del servidor web en el :80, creamos un diccionario personalizado de palabras, esto para ejecutar un ataque de fuerza bruta por FTP.

![](/assets/images/blackmarket/Pasted image 20231025101244.png)

Una vez con el diccionario de palabras creado, con Hydra hacemos un ataque de fuerza bruta para probar contraseñas con ese diccionario por FTP.

![](/assets/images/blackmarket/Pasted image 20231025101306.png)

Obtendríamos la contraseña de Nicky para FTP.


______
# Reconocimiento de FTP
_______

Luego de ingresar con las credenciales nicky a FTP, habrá un único directorio que tendrá en el subdirectorio un .txt, el cual nos bajamos en local.

![](/assets/images/blackmarket/Pasted image 20231025101609.png)

El cual dentro tendrá otra flag con una pista para entrar a otro directorio.


___
# **Inyecciones SQL.**
___


Una vez estemos en el directorio qué nos dió como pista el IMP.TXT, entramos a "Spare Parts" veremos en la URL qué puede ser vulnerable a inyecciones.

![](/assets/images/blackmarket/Pasted image 20231025102246.png)

Tras probar inyecciones SQL, veremos qué es vulnerable ya que duró 5 segundos para recargar, y aquí empezamos a indagar hasta descubrir base de datos qué nos interesen, tablas, columnas & valores.

Luego ir inyectando query para aprovecharnos, descubriremos los siguientes valores de las columnas "username" y "password":

![](/assets/images/blackmarket/Pasted image 20231025103112.png)

En el cual vemos qué las contraseñas están hasheadas con 32bytes, lo que probablemente sea md5sum.

Por lo qué buscamos un recurso para desencriptar ver el valor de este Hash, por ejemplo el recurso web hashes.com, el cual al poner el HASH nos dará en texto claro la contraseña de admin.


_____
# **Inicio de sesión con credenciales válidas.**
_______


Al tener estas credenciales de admin, sí nos vamos al login del mercado negro, y usamos estas credenciales al loguearnos veremos lo siguiente:

![](/assets/images/blackmarket/Pasted image 20231025103811.png)

Lo cual es una flag con una pista de credenciales.


_____
# **inicio de sesión squirrelmail**
______

La ruta **"/squirrelmail/"** la descubrimos tras realizar fuzzing en el servidor web del :80.

![](/assets/images/blackmarket/Pasted image 20231025104109.png)

qué aquí con la pista que se nos dio al ingresar al BlackMarket , podemos loguearnos perfectamente.

Una vez logueado veremos lo siguiente:

![](/assets/images/blackmarket/Pasted image 20231025104247.png)

En el INBOX.Drafts, veremos un mensaje el cual está ilegible porque los caracteres tiene una especie de rotación extraña, para esto usamos el recurso de quipqiup.com

Una vez el mensaje claro, veremos que nos dará una pista con una ruta y a la vez un recurso .jpg.

![](/assets/images/blackmarket/Pasted image 20231025104818.png)


Sí dicha imagen no las bajamos en local, para hacerle un "strings", al final veremos un campo PASS con un contenido en Decimal, cuyo decimal podemos convertir en hexadecimal para luego convertirlo en texto claro, con el recurso rapidtables.com, una vez en texto claro tendremos unas credenciales.


____
# **Fuzzing al recurso "kgbbackdoor"**
_______

```
gobuster dir -u http://192.168.204.151/vworkshop/kgbbackdoor/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php -t 20
```
Descubriremos un nuevo recurso .php en el **/kgbbackdoor/**

![](/assets/images/blackmarket/Pasted image 20231025105840.png)

y aquí deberíamos poner las credenciales que obtuvimos de la imagen, una vez puesta las credenciales correctas tendríamos una webShell, en la cual podemos entablarnos una Reverse Shell

![](/assets/images/blackmarket/Pasted image 20231025110033.png)


______
# **Escalada de Privilegios.**
_____


Una vez entablada nuestra reverse shell, estaremos como el usuario "www-data" pero sí nos vamos al directorio "/home" veríamos un directorio oculto con nombre ".Mylife" que dentro tendrá un recurso con las credenciales del usuario "dimitri"

![](/assets/images/blackmarket/Pasted image 20231025110431.png)

una vez vemos los permisos del usuario "dimitri" vemos los permisos.

![](/assets/images/blackmarket/Pasted image 20231025110545.png)

Y como pertenece al grupo sudo, tan sólo proporcionado la contraseña válida deberíamos convertirnos en usuario privilegiado.

![](/assets/images/blackmarket/Pasted image 20231025110636.png)
