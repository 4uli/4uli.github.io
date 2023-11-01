---
layout: single
title: Topology - Hack The Box
excerpt: "Para vulnerar esta máquina debemos inspecionar detalladamente el sitio web principal, hasta ver que hay un apartado en el cual podemos hacer ecuaciones con LaTeX para verlas en .PNG's, y aprovecharnos de que podemos controlar el input & poder leer archivos a través de las inyecciones maliciosas LaTeX, así lograremos leer un archivo el cual contenga el hash de un usuario, para posterior a ello conectarlos a través de SSH, y finalmente abusar del permiso SUID para convertirnos en usuario privilegiado."
date: 2023-11-01
classes: wide
header:
  teaser: /assets/images/htb-writeup-topology/topology_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - Injection LaTeX
  - Injection LaTeX > LFI
  - pass the hash
  - abusing SUID
---

![](/assets/images/htb-writeup-topology/topology_logo.png)

Para vulnerar esta máquina debemos inspecionar detalladamente el sitio web principal, hasta ver que hay un apartado en el cual podemos hacer ecuaciones con LaTeX para verlas en .PNG's, y aprovecharnos de que podemos controlar el input & poder leer archivos a través de las inyecciones maliciosas LaTeX, así lograremos leer un archivo el cual contenga el hash de un usuario, para posterior a ello conectarlos a través de SSH, y finalmente abusar del permiso SUID para convertirnos en usuario privilegiado.


# PortScan
____

```
 # Nmap 7.93 scan initiated Wed Nov  1 10:12:41 2023 as: nmap -sCV -p22,80 10.10.11.217
 
 Nmap scan report for 10.10.11.217
 Host is up (0.063s latency).
 
 PORT   STATE SERVICE VERSION
 22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
 | ssh-hostkey: 
 |   3072 dcbc3286e8e8457810bc2b5dbf0f55c6 (RSA)
 |   256 d9f339692c6c27f1a92d506ca79f1c33 (ECDSA)
 |_  256 4ca65075d0934f9c4a1b890a7a2708d7 (ED25519)
 80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
 |_http-server-header: Apache/2.4.41 (Ubuntu)
 |_http-title: Miskatonic University | Topology Group
 Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
 
 Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
 # Nmap done at Wed Nov  1 10:12:50 2023 -- 1 IP address (1 host up) scanned in 9.59 seconds
```


# Sitio Web
_____

![](/assets/images/htb-writeup-topology/Pasted image 20231101114923.png)

Sí hacemos fuzzing para descubrir nuevos directorios en el sitio web no descubriremos nada interesante, pero sí nos fijamos en la web contiene un apartado qué genera ecuaciones con LaTeX y nos la representa como .PNG 

![](/assets/images/htb-writeup-topology/Pasted image 20231101115125.png)

# Inyección LaTeX
_____


Sí miramos allí ya que quizás no esté sanitizado y podremos inyectar instrucciones maliciosa con LaTeX, para ello indagamos en [HackTricks](https://book.hacktricks.xyz/pentesting-web/formula-csv-doc-latex-ghostscript-injection) para saber qué tipo de instrucciones podemos llevar a cabo & cómo hacerlas.

![](/assets/images/htb-writeup-topology/read_file_latex.png)

En este caso veremos un apartado para Leer archivos, en este caso usaremos la instrucción qué está seleccionada, pero para qué funcione debemos añadir un signo de **$** al principio y al final de la instrucción, es decir, así:

![](/assets/images/htb-writeup-topology/Pasted image 20231101115700.png)

Y nos leerá el archivo indicado, por ende estamos llevando a cabo una inyección LaTeX, ahora, sí intentamos ver claves privadas en .SSH, o claves públicas no veremos nada, así qué descartamos totalmente, pero sí leemos la ruta del **".htpasswd"** que generalmente se utiliza para almacenar un archivo de contraseñas htpasswd en un servidor web.


![](/assets/images/htb-writeup-topology/Pasted image 20231101120131.png)

![]/assets/images/htb-writeup-topology/(Pasted image 20231101120525.png)

Encontraríamos las credenciales de lo que es un usuario válido del sistema (ya qué al leer el "/etc/passwd" anteriormente estaba), por lo que podríamos aprovecharnos de esto para conectarnos a través de SSH, pero primero debemos desencriptar el hash de la contraseña o también podríamos aplicar una comparativa con la herramienta " **John** the Ripper" y un diccionario, para qué compare el hash y las contraseñas del diccionario.

Primero debemos saber qué tipo de encriptación tiene el Hash, para ello usamos la herramienta ["hash-identifier"](https://github.com/blackploit/hash-identifier) o también podríamos identificar el tipo de encriptación del Hash con la página web [hashes.com](https://hashes.com/en/tools/hash_identifier)
Herramienta:


![](/assets/images/htb-writeup-topology/Pasted image 20231101121222.png)

Web:

![](/assets/images/htb-writeup-topology/Pasted image 20231101121302.png)

Ya qué conocemos la encriptación, procedemos a con la herramienta "**John**" tratar de aplicar la comparativa del hash con un diccionario, en este caso usaremos el tan popular diccionario [rockyou.txt](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt).

Primero deberíamos meter el hash en un archivo de texto, por ejemplo en un "hash.txt". (tiene que estar dentro primero el usuario y el hash, estos separados por dos puntos).

![](/assets/images/htb-writeup-topology/Pasted image 20231101131008.png)


Obtendríamos las credenciales válidas para el usuario v... ahora probamos conectar por SHH.

![](/assets/images/htb-writeup-topology/Pasted image 20231101122825.png)




# Escalada de Privilegios
_______

Sí buscamos por permisos SUID.
```
find / -perm -4000 2>/dev/null
```

![](/assets/images/htb-writeup-topology/Pasted image 20231101123014.png)


Encontraríamos que el binario de la **BASH** contiene permisos SUID de **ROOT**, por lo qué tan sólo haciendo un
```
bash -p
```

obtendríamos una bash como el usuario ROOT.

![](/assets/images/htb-writeup-topology/Pasted image 20231101123046.png)
