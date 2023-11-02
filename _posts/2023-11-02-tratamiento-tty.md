---
layout: single
title: Tratamiento TTY
date: 2023-11-02
classes: wide
header:
  teaser: /assets/images/tty-logo.png
categories:
  - tutorial
tags:
  - tratamiento tty
  - fix tty
  - shell
  - fix reverse shell
---



Es importante estar cómodos a la hora de entablarnos una Reverse Shell, habrán ocasiones en qué no podremos estar cómodos, es decir, a veces no tendremos una **bash**, o la consola qué solamos usar, incluso no podremos limpiar la consola, por ende, para ello debemos llevar a cabo un tratamiento de la TTY para qué se adapte a nuestra máquina, y esto es lo qué podrás aprender hoy leyendo este post.


# Tratamiento TTY.

Una vez recibamos inmediatamente la Reverse Shell por el puerto de escucha, debemos poner lo siguiente.
```bash
script /dev/null/ -c bash
```

Para posterior a ello hacer un **CTRL + Z**, esto nos llevará a nuestra consola de nuestra máquina, veremos que no estamos "conectados", pero para volver a conectarnos debemos hacer lo siguiente:

```bash
stty raw -echo;fg
```

Después de poner esto, esperará que pongamos un input, en el cual pondremos:

```bash
reset xterm
```

Una vez estemos nuevamente dentro de la máquina la cual nos proporcionó la **Reverse Shell,** es cuando llevamos a cabo los últimos pasos, qué es indicar qué valor tendrá la variable de entorno TERM, le indicaremos una variable de entorno la cual nos permita limpiar la pantalla con CTRL + L, para posterior a ello cambiar también la variable SHELL, con la SHELL que más cómodos estemos, ya sea Bash, zh, etcétera.

Modificamos la variable de entorno **TERM** para poder limpiar la pantalla:
```bash
export TERM=xterm
```

Modificamos la **SHELL** qué tiene por la que usaremos:
```bash
export SHELL=bash
```

Ahora necesitamos ajustar las columnas & filas de la **Reverse Shell**, para ello debemos saber cuáles son las columnas & filas de nuestra máquina, para ello nos abrimos una nueva consola desde nuestra máquina, y hacemos lo siguiente pasos:

```bash
stty size
[Filas] [Columnas] 
```

Una vez conozcamos las columnas & filas de nuestra máquina, volvemos a la consola donde estamos con la **Reverse Shell**, para configurarla con las columnas qué tengamos & filas, tal que así:
```
stty rows [Filas] columns [Columnas]
```

Y ya tendríamos totalmente la TTY en la **Reverse Shell** configurada, y ya no tendríamos problemas a la hora de navegar por la máquina con esta consola interactiva.
