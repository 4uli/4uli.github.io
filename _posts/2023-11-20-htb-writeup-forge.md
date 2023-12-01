---
layout: single
title: Forge - Hack The Box
excerpt: "Para resolver ésta máquina, nos aprovechamos de qué no está sanitizada correctamente el servicio web, logrando en la sección de subir imagenes mediante URL bypassear la lista negra para apuntar al localhost de la misma máquina víctima, logrando así descubrir recursos qué externamente no están expuestos, obteniendo así un LFI del FTP para ver la clave privada SSH y ganar acceso a la máquina, posterior a ello nos convertimos en usuarios privilegiados abusando de los SUDOERS en un script de python3 del cual nos aprovechamos de la librería pdb. "
date: 2023-11-22
classes: wide
header:
  teaser: /assets/images/htb-writeup-forge/forge_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - Bypassing URL Blacklist
  - SSRF 
  - SSRF > LFI
  - abusing sudoers (python script)
---

Para resolver ésta máquina, nos aprovechamos de qué no está sanitizada correctamente el servicio web, logrando en la sección de subir imagenes mediante URL bypassear la lista negra para apuntar al localhost de la misma máquina víctima, logrando así descubrir recursos qué externamente no están expuestos, obteniendo así un LFI del FTP para ver la clave privada SSH y ganar acceso a la máquina, posterior a ello nos convertimos en usuarios privilegiados abusando de los SUDOERS en un script de python3 del cual nos aprovechamos de la librería pdb.

![](/assets/images/htb-writeup-forge/forge_logo.png)

# PortScan
__________

```
# Nmap 7.93 scan initiated Wed Nov 22 11:52:51 2023 as: nmap -sCV -p22,80 -oN targeted 10.10.11.111
Nmap scan report for 10.10.11.111
Host is up (0.059s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4f78656629e4876b3cccb43ad25720ac (RSA)
|   256 79df3af1fe874a57b0fd4ed054c628d9 (ECDSA)
|_  256 b05811406d8cbdc572aa8308c551fb33 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://forge.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Nov 22 11:53:00 2023 -- 1 IP address (1 host up) scanned in 9.08 seconds
```


# Website
_______
![](/assets/images/htb-writeup-forge/Pasted image 20231122173611.png)

Arriba a la derecha vemos qué podemos subir imágenes.
![](/assets/images/htb-writeup-forge/Pasted image 20231122173728.png)

Mediante alguna URL, o directamente cargándola desde local, por lo qué  pruebo a ver cómo se suben estas imagenes.
![](/assets/images/htb-writeup-forge/Pasted image 20231122173941.png)

Y al subir una imagen se guarda en "**/upload**", y con unos caracteres qué aparentar ser aleatorio y sin ninguna extensión, por lo qué no podríamos tratar de llevar a cabo una subida de archivo con extensión & script malicioso, así qué intento subir otra cosa que no sea una imagen, por ejemplo un .txt con un texto dentro...

![](/assets/images/htb-writeup-forge/Pasted image 20231122174337.png)

Y me dejó subir el "**upload.txt**", así que procede a ir al enlace donde se guarda el archivo subido, pero veo éste error:
![](/assets/images/htb-writeup-forge/Pasted image 20231122174416.png)

No lo está interpretado porque no es una imagen, pero descargo esto en local para ver que ocurre.
```bash
wget [link]
```

![](/assets/images/htb-writeup-forge/Pasted image 20231122174502.png)

Viendo así qué el archivo a pesar de no ser una imagen se subió correctamente, y obteniéndolo en local con "**wget**" puedo ver su contenido.

De primera no parece ser algo arriesgado de lo cuál podamos aprovecharnos ya qué no podemos controlar la extensión del archivo que subamos, así que opto por probar la opción de subir imágenes con URL.

Se me ocurrió de qué en la opción de subir imágenes con URL apuntar mediante **http** al **localhost**, a ver sí puede hacerse obtener recursos de sí misma., es decir:
```bash
http://localhost:22
```

esto para ver si podemos ver el servicio que está corriendo en local **:22**, pero recibiríamos éste mensaje...
![](/assets/images/htb-writeup-forge/Pasted image 20231122175106.png)

Por detrás parece ser qué existe una una **lista negra** de lo qué podemos poner como input para descargar imágenes mediante **URL**, hago intento de otras manera de llamar al **localhost**, como por ejemplo en hexadecimal.
```
http://0x7f000001:80
```

Logrando así subirlo y **bypasseando** la lista negra, procedemos a descargar el enlance guardado en "**/upload**"
![](/assets/images/htb-writeup-forge/Pasted image 20231122180615.png)

Logrando ver el código fuente, por ende estamos haciendo qué el servidor web realice peticiones HTTP como queramos, un **SSRF**.

En el código fuente nos vemos nada, podríamos tratar de ver todos los puertos & ver qué servicio están corriendo en los que estén abierto, pero duraríamos una eternidad, así que seguí enumerando...

Logrando así encontrar un subdominio adyacente para el dominio principal con la herramienta GoBuster.
```bash
gobuster vhost -u http://forge.htb/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -k | grep -v "302"

===============================================================
2023/11/22 18:08:21 Starting gobuster in VHOST enumeration mode
===============================================================
Found: admin.forge.htb (Status: 200) [Size: 27] 

```

Procedemos a meter este "**subdominio**" en el **"/etc/hosts"** para aplicar el Virtual Hosting, y poder ingresar a éste subdominio.

Ingresamos al subdominio...
![](/assets/images/htb-writeup-forge/Pasted image 20231122181036.png)

Vemos qué sólo se puede entrar desde local, por lo que se me ocurre apuntar a éste **subdominio** desde la subida de imágenes con URL, tal que así:
![](/assets/images/htb-writeup-forge/Pasted image 20231122181123.png)

Pero... También está en lista negra, por lo que opto por ponerlo todo mayúscula a ver sí podemos bypassear esta medida de "seguridad"
![](/assets/images/htb-writeup-forge/Pasted image 20231122181229.png)

Logrando así **bypassearla** y subiéndonos el contenido de esto en el "**/upload**", procedemos a descarganoslo en local para ver qué contiene...

![](/assets/images/htb-writeup-forge/Pasted image 20231122181345.png)
Éste aparentemente es el portal del **Admin**, con un hipervínculo a una ruta desconocida **"/announcements"**, que externamente no tenemos acceso, pero nos aprovechamos del **SSRF** para ver qué hay allí.

![](/assets/images/htb-writeup-forge/Pasted image 20231122181559.png)

Qué nos chiva que el punto final de **"/upload"** acepta un parámetro "**u**" para cargar imágenes por los protocolos **ftps**, **ftp**, **http's**, **http**, por lo que teniendo las credenciales de "**ftp**", podríamos tratar de ver hay en el protocolo **"ftp"**.
```bash
http://ADMIN.FORGE.HTB/upload?u=ftp://user:heightofsecurity123!@0x7f000001
```

Esto para conectarnos al **ftp** de la misma máquina local.
![](/assets/images/htb-writeup-forge/Pasted image 20231122182726.png)

Logrando así ver el contenido, por lo qué se me ocurre qué podríamos tratar de ver sí está expuesta la clave privada.

```bash
http://ADMIN.FORGE.HTB/upload?u=ftp://user:heightofsecurity123!@0x7f000001/.ssh/id_rsa
```

Logrando así obtener la clave privada válida.
![](/assets/images/htb-writeup-forge/Pasted image 20231122183006.png)

Podríamos conectarnos usando ésta clave privada remotamente a la máquina mediante **SSH**, pero no sabemos ningún usuario a ciencia cierta, pero sí conocemos el usuario "**user**" de **ftp**, probamos con éste y... **VOALÁ**
![](/assets/images/htb-writeup-forge/Pasted image 20231122183219.png)
# Escalada de Privilegios.

Listando por los permisos **SUDOERS**, descubrimos ésto:

![](/assets/images/htb-writeup-forge/Pasted image 20231122183323.png)

Es decir, podemos ejecutar como el usuario qué queramos con **python3** el script en "**/opt/remote-manage.py**", así que decido echarle un ojo a éste script.

Código del "**remote-manage.py**":
![](/assets/images/htb-writeup-forge/Pasted image 20231122183607.png)

Qué interpretándolo entendí que abre un puerto de manera aleatoria, y luego para conectarse a ese puerto hay que poner unas credenciales, qué sí ponemos las credenciales válidas, y en las opciones ponemos algo que no sea un **INT**, entrará a la excepción y se quedará en el "**breakpoint**" de la librería **pdb**, por lo qué podríamos en el "**breakpoint**" de la librería pdb importar otra librería y ejecutar código...

1. Ejecutamos el .py.
![](/assets/images/htb-writeup-forge/Pasted image 20231122183836.png)

2. Nos conectamos de otra ventana mediante **SSH**, para conectarnos a éste puerto con las credenciales válidas.
![](/assets/images/htb-writeup-forge/Pasted image 20231122184017.png)

3. Manipulamos el "**breakpoint**" del pdb para meter script malicioso.
![](/assets/images/htb-writeup-forge/Pasted image 20231122184119.png)

Le damos permisos ** ** a la Bash, para convertirnos en Root con tan sólo un
```bash 
bash -p
```
![](/assets/images/htb-writeup-forge/Pasted image 20231122184157.png)
