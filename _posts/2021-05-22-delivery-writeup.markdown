---
description: >-
  G4l1l30 write-up on the easy-difficulty Linux machine Delivery from
  https://hackthebox.eu
title: Hack the Box - Delivery Writeup
date: 2021-05-22 15:40:00 -0600
categories: [Hack the Box, Writeup]
tags: [htb, hacking, hack the box, redteam, Linux,hashcat,hashcat rules, Password Reuse, Account misconfiguration, bash, Web]     # TAG names should always be lowercase
show_image_post: true
image: /assets/img/Linux/delivery/InfoCard-Delivery.png
---


[TOC]

# Summary

Esta es la 2da maquina de ippsec, para ser easy no se le puede exigir mucho, la parte del usuario es enumeracion basica y sentido comun, lo divertido empieza con root, me llevo un poco de tiempo intentando romper el hash obtenido por el simple hecho de no ser cuidadoso con el hint totalmente regalado en la plataforma de Mattermost.

![](/assets/img/Linux/delivery/InfoCard-Delivery.png)



## Info de la maquina

| VM column  | Detalles     |
| ---------- | ------------ |
| Nombre     | Delivery     |
| Dificultad | Easy         |
| Release    | 09/01/2021   |
| OS         | Linux        |
| IP         | 10.10.10.222 |

Editamos ***/etc/hosts***  y agregamos la IP **10.10.10.222**que apunte hacia **delivery.htb** 

# Escaneo de Puertos

Escaneamos los puertos con Masscan y Nmap, utilizando **Masscan_To_Nmap** , esta herramienta escanea puertos TCP y UDP, toma los abiertos y ejecuta Nmap en búsqueda de servicios y scripts por default, mas info y donde conseguir el script: 

* > [https://github.com/Purp1eW0lf/Masscan_to_Nmap](masscan_to_nmap.py)

![](/assets/img/Linux/delivery/mass_scan01.png)

```bash
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)
|_  256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)
80/tcp   open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Welcome
8065/tcp open  unknown
| fingerprint-strings: 
|   GenericLines, Help, RTSPRequest, SSLSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: no-cache, max-age=31556926, public
|     Content-Length: 3108
|     Content-Security-Policy: frame-ancestors 'self'; script-src 'self' cdn.rudderlabs.com
|     Content-Type: text/html; charset=utf-8
|     Last-Modified: Fri, 21 May 2021 20:15:56 GMT
|     X-Frame-Options: SAMEORIGIN
|     X-Request-Id: kmk7wkwd3fr4xgk35hainyyeha
|     X-Version-Id: 5.30.0.5.30.1.57fb31b889bf81d99d8af8176d4bbaaa.false
|     Date: Fri, 21 May 2021 20:28:36 GMT

```

Tanto el puerto **22** y **80** nos son conocidos a excepción del **8065** 

# User Part

## WebEnum:Delivery.htb

Visitando la web, vamos al apartado de **Contacts** y encontramos dos información que nos van a resultar útil

![](/assets/img/Linux/delivery/web_01.png)

- HelpDesk: Un sistema de ticket que apunta hacia `helpdesk.delivery.htb` 
- MatterMostServer : Un servicio que corre en el puerto `8065`

Agregamos **helpdesk.delivery.htb** en `/etc/hosts`.

A tener a consideración, la info es un hint para lo que sigue mas adelante.

## MatterMost

Visitamos MatterMost en el puerto **8065** :

![](/assets/img/Linux/delivery/Mattermost_01.png)

+Info sobre lo que es Mattermost: 

* > .[https://github.com/mattermost/mattermost-server].(MatterMost Server)

Si completamos el registro, inmediatamente nos informara que un correo de confirmación fue enviado, así que necesitamos hacernos de un correo legitimo :) 

Nota: **Recuerde que esto es un ctf, la VM no le enviara nada a su correo personal.**

## HelpDesk

Ahora verificaremos `helpdesk.delivery.htb` 

![](/assets/img/Linux/delivery/HelpDesk.png)



A simple vista es una plataforma de tickets, sin ninguna versión aparente. Ahora mismo, la idea es hacernos con un correo `@delivery.htb` y tal como leímos en **Contacts** , podemos tener acceso a Mattermost vía HelpDesk, así que procedemos a crear un ticket.

![](/assets/img/Linux/delivery/HelpDesk_01.png)



Y tenemos nuestro correo `@delivery.htb` :) : 

**Nota**: Tener pendiente el # de ticket.

![](/assets/img/Linux/delivery/HelpDesk_02.png)

## Mattermost Parte II

Ahora procederemos a registrarnos en Mattermost con el correo obtenido:

![](/assets/img/Linux/delivery/Mattermost_03.png)



Verificamos nuestro ticket en HelpDesk

![](/assets/img/Linux/delivery/HelpDesk_03.png)

Copiamos y pegamos la validación del correo, nos logueamos y tenemos acceso a la plataforma: 

![](/assets/img/Linux/delivery/Mattermost_04.png)

En el thread tenemos un comentario de `root` y obtenemos las credenciales:

`maildeliverer:Youve_G0t_Mail! ` 

Nos logueamos via **ssh** 

![](/assets/img/Linux/delivery/ssh_01.png)



# Root 

Utilizando enum manual y automatizado con **linpeas** no encontre nada interesante, el usuario que obtuvimos no pertenece a un grupo de interes comun o tiene algun privilegio con sudo, sin embargo hay una DB MySQL que se esta ejecutando en la maquina. El usuario que tenemos no nos sirve. 

![](/assets/img/Linux/delivery/mysql_01.png)

Las credenciales correctas de MySQL se encuentran en `/opt/mattermost/config/config.json` , es decir, en la carpeta de configuracion de MatterMost 

![](/assets/img/Linux/delivery/mysql_02.png)

```text
User:mmuser
Pass:Crack_The_MM_Admin_PW
DB:mattermost
```

Nos logueamos a la DB : `mysql -u mmuser -D mattermost -p`

Verificamos las tablas : 

```mysql
show tables;
```







![](/assets/img/Linux/delivery/mysql_03.png)



Ahora a ver el contenido de dicha tabla: 

``` mysql
Select * From Users;
```



![](/assets/img/Linux/delivery/mysql_04.png)

Algo desorganizado, verifiquemos espesificamente el usuario **root** : 

```mysql
SELECT username, password FROM Users WHERE username = 'root';
```

![](/assets/img/Linux/delivery/mysql_05.png)

```
User:root
Hash:$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO
```

Cuyo tipo de hash es segun **hashid** : 

![](/assets/img/Linux/delivery/hashid.png)

So...esta parte puede ser un poco tediosa si no se presta atencion en los comentarios en Mattermost:

![](/assets/img/Linux/delivery/hint_root.png)

Dos hint :1)  `PleaseSubscribe!` que podemos utilizarlo en nuestro wordlist; 2) `hashcat`. 

Necesitamos crear toda las posibles variaciones del hint #1 con una regla de hashcat, en mi caso utilizare `Hob0Rules` :

```bash
git clone https://github.com/praetorian-inc/Hob0Rules
```

Podemos hacerlo de dos formas.

1) Creando un wordlist y luego utilizar hashcat o john

```bash
 hashcat --force dic.txt -r hob064.rule --stdout > wordlist.txt

```

![](/assets/img/Linux/delivery/wordlist_01.png)



```bash
hashcat -a 0 -m 3200 hash wordlist.txt -r Hob0Rules/hob064.rule -o root.txt
```

![](/assets/img/Linux/delivery/hashcat.png)

Revisamos el output: 

![](/assets/img/Linux/delivery/hashcat_01.png)

Probamos el pass:

![](/assets/img/Linux/delivery/root.png)



Root :) 

