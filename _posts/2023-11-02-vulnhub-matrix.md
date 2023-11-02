---
layout: single
title: Matrix - Vulnhub
date: 2/11/2023
classes: wide
header:
  teaser_home_page: true
  icon: /assets/images/vulnhub.webp

tags:
  - Web Enumeration
  - filtration source code
  - brainfuck
  - brute force SSH
  - abusing SUID
  - fácil
  - linux
---


Para resolver esta máquina debemos de enumerar todo los puertos, para posterior a ello encontrarnos qué en uno corre un servicio web aparte del principal, qué en el código fuente nos revela una pista para un directorio, el cual contiene un código en Brainfuck, el cual al descodificarlo nos da credenciales válidas para el usuario invitado, pero esas credenciales no son del todo válida, debemos completarla, para conectarnos & luego abusar de un binario vulnerable al SUID.



# PortScan
__________



```
# Nmap 7.93 scan initiated Thu Nov  2 10:22:50 2023 as: nmap -sCV -p22,80,31337 -oN targeted 192.x.x.x
Nmap scan report for 192.168.204.156
Host is up (0.00040s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 9c8bc77b48dbdb0c4b6869807b124e49 (RSA)
|   256 496c2338fb79cbe0b3feb2f432a2708e (ECDSA)
|_  256 53276f04edd1e781fb009854e600844a (ED25519)
80/tcp    open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
|_http-title: Welcome in Matrix
31337/tcp open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
|_http-title: Welcome in Matrix
MAC Address: 00:0C:29:25:EE:4F (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Nov  2 10:22:58 2023 -- 1 IP address (1 host up) scanned in 8.19 seconds

```


# Sitio Web
__________




![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102110631.png)


Sí analizamos el código fuente no encontraríamos nada interesante, incluso sí hacemos un descubrimiento de nuevas rutas mediante fuzzing con GoBuster encontraremos directorios interesantes.

Por ende, probamos ver qué hay en el puerto **:31337** y descubriremos otro sitio web.



![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102110927.png)

Tendríamos este sitio web qué es muy similar, y sí vemos el código fuente encontraríamos una pista en base64.

![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102111014.png)


Qué al desconvertirla, tendremos una pista en texto claro.

![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102111111.png)

Lo primero que se me ocurre al leer esto, es verificar sí en dicho servicio web existe algún directorio con ese nombre, y al ponerlo efectivamente existe.

![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102111503.png)


Que cómo vemos en su interior contiene lo que sería un script en **Brainfuck**, qué al decodificarlo nos mostraría un texto plano.

![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102111431.png)


Mostrándonos la contraseña para el usuario invitado, sabiendo que el puerto **:22** está abierto podríamos aprovecharnos de estas credenciales para conectarnos, pero no tenemos los últimos dos dígitos, por lo que podríamos hacer un diccionario con la herramienta "**crunch**" para posterior a ello hacer un ataque de fuerza fuerza bruta al servicio SSH con Hydra.


```
crunch 8 8 -t k1ll0r@% > passwords.txt
crunch 8 8 -t k1ll0r%@ >> passwords.txt
```

Como desconocemos los dos caracteres después de la "**r**" creamos el diccionario a partir de ahí, con todas las posibles combinaciones en caracteres minúsculos & números, y luego añadimos combinaciones de números primero & luego caracteres minúsculos.

# Fuerza Bruta (SSH)
_________


Una vez tengamos este diccionario, con la herramienta "**hydra**", haremos un ataque de fuerza bruta probando con el usuario invitado, y como payload de contraseña pondremos el diccionario que creamos.

```
hydra -l guest -P passwords.txt ssh://192.x.x.x -t 20 2>/dev/null
```


![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102112702.png)

Así obtendríamos las credenciales válidas para el usuario invitado, y ya podemos conectarnos a través de SSH.

```
ssh guest@192.x.x.x
```

Una vez dentro, tendremos una bash restringida, por lo qué antes de ingresar podemos ingresar directamente con una bash, poniéndolo al final de la instrucción SSH, es decir, así:


```
ssh guest@192.x.x.x bash
```

Ingresaríamos con una Bash, y tan sólo sería darle tratamiento a la TTY. 
![Aquí verías como darle tratamiento]()



# Escalada de Privilegios
_____



Sí hacemos una búsqueda de permisos SUID.

```
find / -perm -4000 2>/dev/null
```


![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102113352.png)

Encontraríamos que el binario "sudo" tiene permisos SUID como ROOT, así qué tan sólo haciendo un.

```
sudo bash -p
```

proporcionaríamos la contraseña del usuario guest y obtendríamos una consola como Root.

![](/assets/images/vulnhub-writeup-matrix/Pasted image 20231102113437.png)
